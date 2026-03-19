# [STAFF-02] Unique 6-Digit Staff ID Generation

| Field | Value |
|---|---|
| **Type** | Feature |
| **Priority** | P1 MVP |
| **Estimate** | 0.5 days manual / 0.25 days vibe coding |
| **Dependencies** | STAFF-01 — `StaffService.CreateProfile` and `staff_profiles.staff_number` column defined in Section 5.6 and DB-01 Section 6.2. |
| **Platforms** | Backend only (Go) |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for GORM conventions (Section 4), repository context propagation (Section 4.3). |

---

## 1. Overview

Every staff member receives a unique, human-readable 6-digit number (e.g., `012345`, `987654`) displayed on their profile and used for direct search (SEARCH-04). This ticket defines the generation algorithm for that number: generate a random 6-digit zero-padded string, check uniqueness against `staff_profiles`, retry up to 10 times on collision, and fail with a server error if no unique number can be found (statistically near-impossible in practice). The algorithm is encapsulated in `StaffNumberService` which is called by `StaffService.CreateProfile` during staff profile creation.

### End-to-end flow

    [StaffService.CreateProfile]
         │
         ├── Call StaffNumberService.Generate(ctx)
         │     ├── Generate random int in [0, 999999]
         │     ├── Zero-pad to 6 digits → e.g. "007423"
         │     ├── SELECT COUNT(*) FROM staff_profiles WHERE staff_number = ?
         │     ├── If count == 0 → return the number ✓
         │     └── If count > 0  → retry (up to 10 attempts)
         │           └── If all 10 attempts collide → return ErrStaffNumberExhausted
         │
         └── Use returned number as staff_profiles.staff_number

---

## 2. User Flows

### 2.1 Happy Path — Number Generated on Profile Creation

1. `StaffService.CreateProfile` calls `StaffNumberService.Generate(ctx)`.
2. Service generates a random 6-digit string; checks uniqueness.
3. No collision found; returns the 6-digit string.
4. `CreateProfile` stores it in `staff_profiles.staff_number`.
5. The number appears on the staff profile screen (STAFF-07) as "Staff ID: XXXXXX".

### 2.2 Error Cases

| Scenario | Behaviour |
|---|---|
| First generated number is already taken | `StaffNumberService` retries with a new random number (up to 10 attempts automatically) |
| All 10 attempts collide (extremely unlikely below 100k registered staff) | `StaffService.CreateProfile` returns `ErrStaffNumberExhausted`; handler returns `500 server_error`; error is logged |
| Database error during uniqueness check | `StaffNumberService` returns the DB error; `StaffService` returns `500 server_error` |
| Network error on client side | Red SnackBar: `"Something went wrong. Please try again later."` |

---

## 3. UI / Screen

Not applicable for this ticket. The staff number is generated server-side and displayed by STAFF-07. There is no dedicated screen for this ticket.

---

## 4. Frontend — Flutter

Not applicable for this ticket. The 6-digit staff number is returned in the `StaffProfileResponse` from `POST /api/v1/staff/profile` (STAFF-01 Section 5.2) and displayed on the profile screen (STAFF-07).

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

No new API endpoints are introduced in this ticket. The staff number generation logic is an internal service component called from `StaffService.CreateProfile` (STAFF-01 Section 5.6).

### 5.2 StaffNumberService — Generation Algorithm

`StaffNumberService.Generate(ctx context.Context) (string, error)`:

1. Initialize `maxAttempts = 10`.
2. For each attempt from 1 to `maxAttempts`:
   a. Generate a random integer in the range [0, 999999] using `crypto/rand` for uniform distribution.
   b. Zero-pad to 6 digits using `fmt.Sprintf("%06d", n)` to produce values like `"000001"` through `"999999"`.
   c. Call `StaffRepository.StaffNumberExists(ctx, number)`. If the query returns an error, return that error immediately.
   d. If `StaffNumberExists` returns false, return the generated number — generation is complete.
3. If all `maxAttempts` attempts result in collisions, return `ErrStaffNumberExhausted`.

The use of `crypto/rand` (not `math/rand`) ensures that the number is cryptographically random, preventing sequential pattern-guessing.

### 5.3 Service Responsibilities

`StaffNumberService`:

- `Generate(ctx context.Context) (string, error)` — Generates a unique 6-digit zero-padded string per the algorithm above. Returns `ErrStaffNumberExhausted` if 10 consecutive collisions occur.

`StaffRepository` (added method):

- `StaffNumberExists(ctx context.Context, number string) (bool, error)` — Queries `SELECT COUNT(1) FROM staff_profiles WHERE staff_number = ?` using `db.WithContext(ctx)`. Returns true if count > 0, false otherwise.

---

## 6. Database

Not applicable for this ticket. The `staff_profiles.staff_number` column (`VARCHAR(6) UNIQUE NOT NULL`) is already defined in DB-01 Section 6.2. No migration changes are required.

---

## 7. Configuration

Not applicable for this ticket. No new environment variables are required.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Generate number — no existing staff profiles | Returns a valid 6-digit zero-padded string on first attempt |
| 2 | Generate number — first 9 candidates are taken, 10th is free | Returns the 10th candidate successfully |
| 3 | Generate number — all 10 candidates collide | Returns `ErrStaffNumberExhausted`; handler returns `500 server_error` |
| 4 | Generated number format | Always exactly 6 characters; zero-padded (e.g., `"000042"`, not `"42"`) |
| 5 | `StaffNumberExists` with existing number | Returns `true` |
| 6 | `StaffNumberExists` with non-existent number | Returns `false` |
| 7 | DB error during `StaffNumberExists` | Error propagated to caller; `500 server_error` returned to client |
| 8 | Two simultaneous registrations — same number generated | Only one succeeds due to `UNIQUE` constraint on `staff_number`; other retries and succeeds |

### 8.2 Flutter Test Cases

Not applicable for this ticket. The staff number is an opaque string value displayed by STAFF-07. No Flutter-side logic is added here.

---

## 9. Security Checklist

- [ ] Staff numbers are generated using `crypto/rand`, not `math/rand`, to prevent sequential guessing attacks.
- [ ] The `staff_number` column has a `UNIQUE` constraint at the database level as a last line of defence against race conditions in concurrent registrations.
- [ ] No sensitive data is returned in error messages relating to staff number generation failures.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Go — Service | `internal/service/staff_number_service.go` | Create — StaffNumberService with Generate method |
| Go — Repository | `internal/repository/staff_repository.go` | Modify — add StaffNumberExists method |
| Go — Service | `internal/service/staff_service.go` | Modify — inject StaffNumberService; call Generate in CreateProfile |

---

## 11. Acceptance Criteria

- [ ] `StaffNumberService.Generate` returns a string of exactly 6 characters on every call.
- [ ] The returned string is zero-padded (values less than 100000 are left-padded with zeros).
- [ ] The returned string does not match any existing `staff_number` in `staff_profiles`.
- [ ] `StaffNumberExists` returns true for a number that is already in `staff_profiles`.
- [ ] `StaffNumberExists` returns false for a number that is not in `staff_profiles`.
- [ ] When all 10 generation attempts collide, `ErrStaffNumberExhausted` is returned and the handler returns `500 server_error`.
- [ ] Two concurrent staff registrations cannot receive the same staff number due to the database `UNIQUE` constraint.
- [ ] `crypto/rand` is used for number generation; `math/rand` is not used anywhere in this service.
