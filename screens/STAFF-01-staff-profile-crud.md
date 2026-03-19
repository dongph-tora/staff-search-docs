# [STAFF-01] Staff Profile CRUD

| Field | Value |
|---|---|
| **Type** | Feature |
| **Priority** | P1 MVP |
| **Estimate** | 3 days manual / 1 day vibe coding |
| **Dependencies** | DB-01 — `staff_profiles`, `staff_portfolio_photos`, `staff_social_links` tables defined in Section 6.2. AUTH-01 — `ApiClient` and `AuthProvider` created in Sections 4.1–4.4. |
| **Platforms** | iOS · Android · Web |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format (Section 2), JWT middleware (Section 3), GORM conventions (Section 4), `ApiClient` (Section 6), `AuthProvider` (Section 7), error display (Section 10). |

---

## 1. Overview

This ticket implements the complete staff profile lifecycle: creation (when a user opts in as staff), retrieval (by the staff themselves or by another user), and update (editing bio, location, job title, accept_bookings). It establishes the `StaffProfile` GORM model, repository, service, and handler that all downstream STAFF tickets build on. After completing this ticket, any authenticated user with `role=staff` or `is_staff_registered=false` can create a staff profile, and any user can view a staff profile by user ID.

### End-to-end flow — Create Staff Profile

    [Flutter App]                    [Go API]                 [PostgreSQL]
         │                               │                           │
         │── POST /api/v1/staff/profile ►│                           │
         │   {job_title, job_category,   │── INSERT staff_profiles ─►│
         │    location, bio}             │◄── row created ───────────│
         │                               │── UPDATE users            │
         │                               │   is_staff_registered=true│
         │◄── 201 {staff_profile} ───────│                           │
         │── StaffProvider.setProfile()  │                           │
         │── navigate to staff home      │                           │

### End-to-end flow — Get Staff Profile

    [Flutter App]                    [Go API]                 [PostgreSQL]
         │                               │                           │
         │── GET /api/v1/staff/:userID ─►│                           │
         │                               │── SELECT staff_profiles ─►│
         │                               │── SELECT portfolio_photos ►│
         │                               │◄── rows ──────────────────│
         │◄── 200 {staff_profile} ───────│                           │

---

## 2. User Flows

### 2.1 Happy Path — Create Staff Profile (First Time)

1. Authenticated user (role=user or staff) taps "Register as Staff" on the profile screen.
2. The app navigates to `StaffRegistrationScreen`.
3. User fills in job title, selects job category, enters location and bio.
4. User taps "Save".
5. App shows `CircularProgressIndicator` on the Save button and disables it.
6. App calls `POST /api/v1/staff/profile`.
7. API creates the `staff_profiles` row, generates a unique 6-digit `staff_number` (STAFF-02), and updates `users.is_staff_registered = true` and `users.role = 'staff'`.
8. API returns `201` with the new `StaffProfileResponse`.
9. `StaffProvider` updates local state.
10. Green SnackBar: `"Staff profile created successfully."` App navigates to the staff home tab.

### 2.2 Happy Path — Update Staff Profile

1. Staff user opens their own profile screen and taps "Edit Profile".
2. App navigates to `StaffProfileEditScreen` pre-filled with current values.
3. Staff edits one or more fields and taps "Save".
4. App calls `PATCH /api/v1/staff/profile`.
5. API updates the `staff_profiles` row.
6. API returns `200` with the updated `StaffProfileResponse`.
7. `StaffProvider` updates local state.
8. Green SnackBar: `"Profile updated."` App pops back to the profile screen.

### 2.3 Happy Path — View Another Staff's Profile

1. User taps a staff card anywhere in the app.
2. App navigates to `StaffProfileScreen` passing the `userID`.
3. App calls `GET /api/v1/staff/:userID`.
4. API returns `200` with the `StaffProfileResponse`.
5. Screen renders the staff's name, avatar, job title, bio, rating, follower count, portfolio photos, and social links.

### 2.4 Error Cases

| Scenario | Behaviour |
|---|---|
| Create profile when one already exists | `409 conflict` returned. Red SnackBar: `"You already have a staff profile."` |
| PATCH called without any fields | `400 bad_request`. Red SnackBar: `"No changes to save."` |
| GET profile for a non-existent userID | `404 not_found`. Red SnackBar: `"Staff profile not found."` |
| Network error | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| Server error (500) | Red SnackBar: `"Something went wrong. Please try again later."` |

---

## 3. UI / Screen

### 3.1 Staff Registration Screen (`staff_registration_screen.dart`)

> Current location: `lib/screens/staff/staff_registration_screen.dart`
> Status: **Create new**

    ┌─────────────────────────────────────────┐
    │  ← Register as Staff                   │
    │                                         │
    │  [Job Title text field]                 │
    │  [Job Category dropdown]                │
    │  [Location text field]                  │
    │  [Bio multi-line text field, 3 rows]    │
    │                                         │
    │  Accept Bookings  [Toggle switch]       │
    │                                         │
    │  [  Save and Continue  ]                │
    └─────────────────────────────────────────┘

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| `jobTitleField` | `TextFormField` | Required; max 100 chars |
| `jobCategoryDropdown` | `DropdownButtonFormField` | Lists all 21 job categories from STAFF-03; required |
| `locationField` | `TextFormField` | Optional; max 255 chars |
| `bioField` | `TextFormField` | Optional; maxLines 5; max 1000 chars |
| `acceptBookingsSwitch` | `Switch` | Defaults to true |
| `saveButton` | `ElevatedButton` | Shows `CircularProgressIndicator` (20×20, strokeWidth 2) while loading; disabled during request |

### 3.2 Staff Profile Edit Screen (`staff_profile_edit_screen.dart`)

> Current location: `lib/screens/staff/staff_profile_edit_screen.dart`
> Status: **Create new**

    ┌─────────────────────────────────────────┐
    │  ← Edit Profile                        │
    │                                         │
    │  [Job Title text field (pre-filled)]    │
    │  [Job Category dropdown (pre-filled)]   │
    │  [Location text field (pre-filled)]     │
    │  [Bio multi-line text field]            │
    │  Accept Bookings [Toggle switch]        │
    │                                         │
    │  [  Save Changes  ]                     │
    └─────────────────────────────────────────┘

Key UI elements: same as 3.1 but all fields pre-filled from `StaffProvider.currentProfile`.

### 3.3 Staff Profile View Screen (`staff_profile_screen.dart`)

> Current location: `lib/screens/staff/staff_profile_screen.dart`
> Status: **Modify existing** — replace mock data bindings with `StaffProvider` / `StaffService` calls

    ┌─────────────────────────────────────────┐
    │  ← [Staff Name]            [Follow btn]│
    │                                         │
    │  [Avatar circle, 80px]                  │
    │  Staff #: 123456                        │
    │  [Job Title] · [Job Category badge]     │
    │  ★ 4.8  (120 reviews)                  │
    │  👥 1,234 followers                     │
    │  📍 Tokyo, Shibuya                      │
    │                                         │
    │  About ────────────────────────────     │
    │  [Bio text]                             │
    │                                         │
    │  Portfolio ─────────────────────────   │
    │  [3-column photo grid]                  │
    │                                         │
    │  [  Book Now  ]  [  Tip  ]             │
    └─────────────────────────────────────────┘

Changes required:
- Replace all mock data with data from `StaffService.getProfile(userID)`.
- Show `CircularProgressIndicator` while loading.
- Show `"Staff profile not found."` message if 404 returned.

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/models/staff_profile.dart` | `StaffProfile` model with all fields from `staff_profiles` table plus nested `portfolioPhotos` list |
| `lib/services/staff_service.dart` | API calls: `createProfile`, `updateProfile`, `getProfile(userID)`, `getMyProfile` |
| `lib/providers/staff_provider.dart` | `ChangeNotifier` owning `currentProfile` (own staff's data), `isLoading`, `error` |
| `lib/screens/staff/staff_registration_screen.dart` | First-time staff profile creation form |
| `lib/screens/staff/staff_profile_edit_screen.dart` | Edit existing staff profile |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/screens/staff/staff_profile_screen.dart` | Replace mock data with `StaffService.getProfile(userID)` calls |
| `lib/main.dart` | Register `StaffProvider` in `MultiProvider` |

### 4.3 StaffService Responsibilities

`StaffService` is a singleton that wraps all staff profile API calls. Every method calls `ApiClient` and returns typed results:

- `createProfile(CreateStaffProfileRequest req)` — Calls `POST /api/v1/staff/profile`. Returns `StaffProfile` on success.
- `updateProfile(UpdateStaffProfileRequest req)` — Calls `PATCH /api/v1/staff/profile`. Returns updated `StaffProfile`.
- `getProfile(String userID)` — Calls `GET /api/v1/staff/:userID`. Returns `StaffProfile` or null if 404.
- `getMyProfile()` — Calls `GET /api/v1/staff/me`. Returns `StaffProfile` or null.

### 4.4 StaffProvider Responsibilities

`StaffProvider` (ChangeNotifier) owns:
- `currentProfile` (`StaffProfile?`) — the authenticated staff user's own profile, null if not a staff user.
- `isLoading` (`bool`) — set true during any async operation, always cleared in a finally block.
- `error` (`String?`) — last error message.

On app start, if `AuthProvider.currentUser?.isStaffRegistered == true`, `StaffProvider` calls `getMyProfile()` to hydrate `currentProfile`.

### 4.5 Form Validation Rules

| Field | Required | Rule | Error Message |
|---|---|---|---|
| `jobTitle` | Yes | 1–100 characters | `"Job title is required."` |
| `jobCategory` | Yes | Must be one of the 21 valid category keys | `"Please select a job category."` |
| `location` | No | Max 255 characters | `"Location must be 255 characters or less."` |
| `bio` | No | Max 1000 characters | `"Bio must be 1000 characters or less."` |

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/staff/profile` | Create staff profile for the authenticated user | Yes |
| `PATCH` | `/api/v1/staff/profile` | Update the authenticated staff's own profile | Yes |
| `GET` | `/api/v1/staff/me` | Get the authenticated user's own staff profile | Yes |
| `GET` | `/api/v1/staff/:userID` | Get any staff's public profile by their user ID | Yes |

### 5.2 POST /api/v1/staff/profile

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `job_title` | string | Yes | 1–100 characters |
| `job_category` | string | Yes | Must be one of the 21 valid category keys (see STAFF-03) |
| `location` | string | No | Max 255 characters |
| `bio` | string | No | Max 1000 characters |
| `accept_bookings` | bool | No | Defaults to true if omitted |

**Response `201`:**

| Field | Type | Description |
|---|---|---|
| `id` | string | ULID of the staff_profiles row |
| `user_id` | string | The authenticated user's ID |
| `staff_number` | string | Unique 6-digit staff ID |
| `job_title` | string | Job title |
| `job_category` | string | Job category key |
| `location` | string | Location or empty string |
| `bio` | string | Bio or empty string |
| `intro_video_url` | string\|null | null until STAFF-06 |
| `is_available` | bool | false initially |
| `accept_bookings` | bool | As provided |
| `rating` | number | 0.00 initially |
| `review_count` | int | 0 initially |
| `followers_count` | int | 0 initially |
| `total_tips_received` | int | 0 initially |
| `portfolio_photos` | array | Empty array initially |
| `created_at` | string | ISO 8601 timestamp |
| `updated_at` | string | ISO 8601 timestamp |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed body or missing required fields |
| `401` | `unauthorized` | Missing or invalid JWT |
| `409` | `conflict` | Staff profile already exists for this user |
| `422` | `validation_error` | `job_category` is not a valid category key; `job_title` exceeds 100 chars |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Parse and validate request body into `CreateStaffProfileRequest` DTO. Return `400` if parsing fails.
3. Validate `job_category` against the allowed list. Return `422` if invalid.
4. Validate `job_title` length. Return `422` if exceeds 100 chars.
5. Call `StaffService.CreateProfile(ctx, userID, req)`. Return `409` if service returns `ErrConflict`. Return `500` for other errors.
6. Return `201` with `StaffProfileResponse`.

### 5.3 PATCH /api/v1/staff/profile

**Request body:** (all fields optional; at least one must be provided)

| Field | Type | Required | Validation |
|---|---|---|---|
| `job_title` | string | No | 1–100 characters if provided |
| `job_category` | string | No | Must be a valid category key if provided |
| `location` | string | No | Max 255 characters |
| `bio` | string | No | Max 1000 characters |
| `accept_bookings` | bool | No | — |
| `is_available` | bool | No | Online/offline toggle |

**Response `200`:** Same fields as the `201` response in Section 5.2.

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed body or no fields provided |
| `401` | `unauthorized` | Missing or invalid JWT |
| `404` | `not_found` | Authenticated user has no staff profile |
| `422` | `validation_error` | Invalid field values |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Parse request body. Return `400` if no fields are present.
3. Validate each provided field. Return `422` if any validation fails.
4. Call `StaffService.UpdateProfile(ctx, userID, req)`. Return `404` if `ErrNotFound`. Return `500` for other errors.
5. Return `200` with updated `StaffProfileResponse`.

### 5.4 GET /api/v1/staff/me

**Response `200`:** Same fields as the `201` response in Section 5.2, plus `portfolio_photos` array populated.

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `401` | `unauthorized` | Missing or invalid JWT |
| `404` | `not_found` | Authenticated user has no staff profile |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Call `StaffService.GetByUserID(ctx, userID)`. Return `404` if `ErrNotFound`.
3. Return `200` with `StaffProfileResponse`.

### 5.5 GET /api/v1/staff/:userID

**Response `200`:** Same fields as the `201` response in Section 5.2, plus `portfolio_photos` array and `user` object with `name`, `avatar_url`.

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `401` | `unauthorized` | Missing or invalid JWT |
| `404` | `not_found` | No staff profile for this userID |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from path param `c.Params("userID")`.
2. Call `StaffService.GetByUserID(ctx, userID)`. Return `404` if `ErrNotFound`.
3. Return `200` with `StaffProfileResponse`.

### 5.6 Service Responsibilities

`StaffService`:

- `CreateProfile(ctx, userID string, req CreateStaffProfileRequest) (StaffProfile, error)` — Checks no profile exists for userID (return ErrConflict if one does). Generates a unique staff_number via `StaffNumberService.Generate(ctx)` (STAFF-02). Creates `staff_profiles` row. Updates `users.is_staff_registered = true` and `users.role = 'staff'` in the same transaction. Returns the created profile.
- `UpdateProfile(ctx, userID string, req UpdateStaffProfileRequest) (StaffProfile, error)` — Finds profile by userID. Applies only the non-nil fields from `req`. Saves. Returns updated profile.
- `GetByUserID(ctx, userID string) (StaffProfile, error)` — Queries `staff_profiles` by `user_id`. Preloads `portfolio_photos`. Returns `model.ErrNotFound` if not found.

---

## 6. Database

Not applicable for this ticket. The `staff_profiles`, `staff_portfolio_photos`, and `staff_social_links` tables are defined in DB-01 Section 6.2. This ticket adds the GORM model structs and repository methods to the scaffolding created in DB-01.

The `StaffProfile` GORM model is created in `internal/model/staff.go` and registered in `main.go`'s `db.AutoMigrate()` call.

---

## 7. Configuration

Not applicable for this ticket. No new environment variables are required.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Valid POST — all required fields | `201`, staff profile returned, `staff_number` is 6 digits, `users.is_staff_registered = true` |
| 2 | POST — missing `job_title` | `400 bad_request` |
| 3 | POST — invalid `job_category` value | `422 validation_error` |
| 4 | POST — duplicate (user already has profile) | `409 conflict` |
| 5 | POST — no JWT | `401 unauthorized` |
| 6 | Valid PATCH — update `bio` only | `200`, only `bio` field changed, other fields unchanged |
| 7 | PATCH — no fields in body | `400 bad_request` |
| 8 | PATCH — user has no staff profile | `404 not_found` |
| 9 | GET /staff/me — authenticated staff | `200`, full profile with portfolio_photos |
| 10 | GET /staff/me — user has no profile | `404 not_found` |
| 11 | GET /staff/:userID — valid ID | `200`, public profile returned |
| 12 | GET /staff/:userID — non-existent user | `404 not_found` |
| 13 | All endpoints without JWT | `401 unauthorized` |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Submit valid registration form | `201` received; green SnackBar; navigate to staff home tab |
| 2 | Submit with empty job title | Inline error `"Job title is required."`; no API call |
| 3 | Submit without selecting job category | Inline error `"Please select a job category."`; no API call |
| 4 | Server returns 409 | Red SnackBar: `"You already have a staff profile."` |
| 5 | Save button tapped | `CircularProgressIndicator` shown (20×20 px); button disabled |
| 6 | Network error | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| 7 | Profile screen loaded | Staff data rendered from API; no mock data visible |
| 8 | Profile not found (404) | Message `"Staff profile not found."` displayed; no crash |

---

## 9. Security Checklist

- [ ] Only the authenticated user can create or update their own staff profile; `userID` is read from JWT claims, never from the request body.
- [ ] `job_category` is validated against an explicit allowlist server-side; clients cannot inject arbitrary category values.
- [ ] Staff profile data for other users (GET by userID) does not include sensitive fields (email, password_hash, google_id).
- [ ] Input validated server-side, not only client-side.
- [ ] No sensitive data returned in error messages.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Model | `lib/models/staff_profile.dart` | Create — StaffProfile, StaffPortfolioPhoto models |
| Flutter — Service | `lib/services/staff_service.dart` | Create — createProfile, updateProfile, getProfile, getMyProfile |
| Flutter — Provider | `lib/providers/staff_provider.dart` | Create — StaffProvider ChangeNotifier |
| Flutter — Screen | `lib/screens/staff/staff_registration_screen.dart` | Create — first-time registration form |
| Flutter — Screen | `lib/screens/staff/staff_profile_edit_screen.dart` | Create — edit profile form |
| Flutter — Screen | `lib/screens/staff/staff_profile_screen.dart` | Modify — replace mock data with API calls |
| Flutter — App | `lib/main.dart` | Modify — register StaffProvider in MultiProvider |
| Go — Model | `internal/model/staff.go` | Create — StaffProfile, StaffPortfolioPhoto, StaffSocialLink GORM models |
| Go — Handler | `internal/handler/staff_handler.go` | Create — CreateProfile, UpdateProfile, GetMyProfile, GetProfile |
| Go — Service | `internal/service/staff_service.go` | Create — CreateProfile, UpdateProfile, GetByUserID |
| Go — Repository | `internal/repository/staff_repository.go` | Modify — add FindByUserID, Create, Update methods to scaffolded file from DB-01 |
| Go — DTO | `internal/dto/staff_dto.go` | Create — CreateStaffProfileRequest, UpdateStaffProfileRequest, StaffProfileResponse |
| Go — Router | `router/router.go` | Modify — register /api/v1/staff routes under JWT middleware group |
| Go — main | `main.go` | Modify — register StaffProfile model in AutoMigrate call |

---

## 11. Acceptance Criteria

- [ ] `POST /api/v1/staff/profile` with valid data creates a staff profile, sets `users.is_staff_registered = true`, `users.role = 'staff'`, and returns `201` with a unique 6-digit `staff_number`.
- [ ] `POST /api/v1/staff/profile` called a second time for the same user returns `409 conflict`.
- [ ] `PATCH /api/v1/staff/profile` with only `bio` updated returns `200` with `bio` changed and all other fields unchanged.
- [ ] `GET /api/v1/staff/me` returns the authenticated staff's own profile including `portfolio_photos` array.
- [ ] `GET /api/v1/staff/:userID` returns a public staff profile for any valid userID; does not expose `email` or `password_hash`.
- [ ] `StaffRegistrationScreen` validates required fields before making any API call.
- [ ] `StaffProfileScreen` displays live data from the API with no hardcoded mock values.
- [ ] Save button shows `CircularProgressIndicator` (20×20 px, strokeWidth 2) while any async action is in progress and is re-enabled afterward.
- [ ] All API endpoints return the standard error envelope for non-2xx responses.
- [ ] Errors are displayed via red SnackBar per conventions.
- [ ] Loading state is always cleared in a finally block regardless of success or failure.
- [ ] All endpoints return `401` when called without a valid JWT.
