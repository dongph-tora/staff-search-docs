# [DB-02] Migrate All Services from LocalStorage to Real DB

| Field | Value |
|---|---|
| **Type** | Infrastructure |
| **Priority** | P1 MVP |
| **Estimate** | 2 days manual / 0.5 days vibe coding |
| **Dependencies** | DB-01 — full `users` table column definitions (Section 6.1). AUTH-01 — `ApiClient` and `AuthProvider` created in Sections 4.1–4.4; `GET /api/v1/auth/me` defined in Section 5.7. |
| **Platforms** | iOS · Android · Web |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format (Section 2), JWT middleware (Section 3), GORM conventions (Section 4), `ApiClient` (Section 6), `AuthProvider` (Section 7), token storage (Section 8), navigation (Section 9), error display (Section 10). |

---

## 1. Overview

This ticket replaces the SharedPreferences-based local storage implementations in the Flutter app with real PostgreSQL-backed API calls. It has two concrete deliverables: (1) a user profile update endpoint (`PATCH /api/v1/users/me`) on the Go backend, and (2) the migration of `AppModeService` and all profile-related screens to use `AuthProvider` and `ApiClient` instead of `StorageHelper`.

DB-02 also establishes the migration pattern that all subsequent service tickets (BOOK-01, PAY-08, SOCIAL-01, REV-01, CHAT-01, etc.) must follow when replacing their own local mock implementations with real API calls.

The migration targets three categories of local state:

1. **User profile data** — currently stored in `AppModeService` via `StorageHelper` (SharedPreferences). Migrated to `AuthProvider.currentUser` backed by the real API.
2. **App mode state** — whether the current session is in "user mode" or "staff mode". Derived from `AuthProvider.currentUser.role` rather than a separate SharedPreferences key.
3. **Feature-specific data** — bookings, tips, follows, reviews, messages, and media. These are out of scope for DB-02; each will be migrated in their respective feature tickets (BOOK-01, PAY-08, SOCIAL-01, REV-01, CHAT-01, FEED-01).

### End-to-end flow — Profile Update

    [Flutter App]                   [Go API]                    [PostgreSQL]
         │                               │                            │
         │── PATCH /api/v1/users/me ────►│                            │
         │   {name, bio, phone}          │── UPDATE users SET ... ───►│
         │                               │◄── updated row ────────────│
         │◄── 200 {user} ────────────────│                            │
         │── AuthProvider.updateCurrentUser()                         │
         │── navigate back to profile    │                            │

### End-to-end flow — Session Restore with Role Detection

    [Flutter App]                   [Go API]
         │
         ├── app start
         ├── AuthProvider.restoreSession()
         │── GET /api/v1/auth/me ────────►│
         │◄── 200 {user, role} ──────────│
         │
         ├── AppModeService reads role
         │   from AuthProvider.currentUser.role
         │   (no SharedPreferences read)

---

## 2. User Flows

### 2.1 Happy Path — User Updates Profile

1. Authenticated user opens the Profile screen and taps "Edit profile".
2. `profile_edit_screen.dart` opens, pre-populated with data from `AuthProvider.currentUser`.
3. User edits name, bio, or phone number and taps "Save".
4. Flutter validates the form inline. If validation fails, the Save button remains enabled but no API call is made.
5. If validation passes, the Save button shows `CircularProgressIndicator` (20×20 px) and is disabled.
6. `UserService.updateProfile(fields)` calls `PATCH /api/v1/users/me`.
7. Go API validates the request body, updates the `users` row in PostgreSQL, and returns the updated user object.
8. Flutter calls `AuthProvider.updateCurrentUser(updatedUser)` to refresh in-memory state.
9. Flutter shows green SnackBar: `"Profile updated successfully."`.
10. User is navigated back to the Profile screen, which reflects the updated data.

### 2.2 Happy Path — App Start Role Detection

1. App launches and calls `AuthProvider.restoreSession()`.
2. `restoreSession` reads tokens from `FlutterSecureStorage`, validates them, and calls `GET /api/v1/auth/me`.
3. `AuthProvider.currentUser` is populated with the full user object including `role`.
4. `AppModeService` reads `role` directly from `AuthProvider.currentUser.role` — no SharedPreferences read occurs.
5. App renders the appropriate navigation (user tab bar vs. staff dashboard) based on the returned role.

### 2.3 Error Cases

| Scenario | Behaviour |
|---|---|
| Name exceeds 100 characters | Inline validation error below the field: `"Name must be 100 characters or fewer."` No API call is made. |
| Bio exceeds 500 characters | Inline validation error below the field: `"Bio must be 500 characters or fewer."` No API call is made. |
| Phone number format is invalid | Inline validation error below the field: `"Enter a valid phone number."` No API call is made. |
| 401 — token expired and refresh fails | `ApiClient` clears stored tokens and redirects to `/login` clearing the back stack. |
| Network error | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| Server error (500) | Red SnackBar: `"Something went wrong. Please try again later."` |

---

## 3. UI / Screen

### 3.1 Profile Edit Screen (`profile_edit_screen.dart`)

> Current location: `lib/screens/profile_edit_screen.dart`
> Status: **Modify existing**

    ┌─────────────────────────────────────────┐
    │  ← Edit Profile                         │
    │                                         │
    │  [  Profile Photo (circle avatar)  ]    │
    │     (tap → coming soon SnackBar)        │
    │                                         │
    │  Name                                   │
    │  ___________________________________    │
    │                                         │
    │  Bio                                    │
    │  ___________________________________    │
    │  ___________________________________    │
    │                                         │
    │  Phone                                  │
    │  ___________________________________    │
    │                                         │
    │  [         Save Changes          ]      │
    └─────────────────────────────────────────┘

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| Profile photo | `GestureDetector` + `CircleAvatar` | Tap shows SnackBar: `"Profile photo upload coming soon."` Implemented in AUTH-09 after DB-05. |
| Name field | `TextFormField` | Pre-filled from `AuthProvider.currentUser.name`. Max 100 characters. |
| Bio field | `TextFormField` | Pre-filled from `AuthProvider.currentUser.bio`. Max 500 characters. Multiline, max 5 lines. |
| Phone field | `TextFormField` | Pre-filled from `AuthProvider.currentUser.phone`. Numeric keyboard hint. |
| Save button | `ElevatedButton` | Disabled and shows `CircularProgressIndicator` (20×20 px, strokeWidth 2) while saving. |

Changes required:

- Remove all calls to `AppModeService.saveUserProfile()` and `StorageHelper.setString('user_profile', ...)`.
- Remove all reads from `StorageHelper` or SharedPreferences for pre-filling form fields.
- Pre-populate fields from `context.read<AuthProvider>().currentUser`.
- On Save, call `UserService.updateProfile(fields)` and await the `ApiResponse<User>` result.
- On success, call `AuthProvider.updateCurrentUser(response.data)` then show green SnackBar and navigate back.
- On failure, show red SnackBar with the appropriate message per Section 2.3.

### 3.2 Profile Settings Screen (`profile_settings_screen.dart`)

> Current location: `lib/screens/profile_settings_screen.dart`
> Status: **Modify existing**

No layout changes. Changes required:

- Remove any reads from `StorageHelper` or `AppModeService` that retrieve the current user's name, avatar URL, or role.
- Replace with reads from `context.watch<AuthProvider>().currentUser`.

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/services/user_service.dart` | Wraps `ApiClient` calls for user profile operations. Exposes `updateProfile(Map<String, dynamic> fields)` returning `Future<ApiResponse<User>>`. |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/services/app_mode_service.dart` | Remove all SharedPreferences reads and writes for profile data (`user_profile`, `staff_profile`) and login state (`user_logged_in`, `staff_logged_in`). Derive current role from `AuthProvider.currentUser.role`. Retain only the in-memory UI mode toggle if needed for the "switch to staff view" feature. |
| `lib/screens/profile_edit_screen.dart` | Remove `StorageHelper` usage. Pre-fill fields from `AuthProvider`. Call `UserService.updateProfile()` on save. Handle response per Section 2. |
| `lib/screens/profile_settings_screen.dart` | Remove `StorageHelper` reads. Read user data from `AuthProvider.currentUser`. |
| `lib/providers/auth_provider.dart` | Add `updateCurrentUser(User user)` method that replaces `currentUser` in memory and calls `notifyListeners()`. |
| `lib/utils/storage_helper.dart` | Add a deprecation comment block at the top of the file stating: new code must not use this class for user-facing persistent data — use `AuthProvider` or `ApiClient` instead. No logic changes in this ticket. |

### 4.3 UserService Responsibilities

`UserService` wraps all API calls related to user data that are not authentication operations. It holds a reference to `ApiClient` (injected or accessed as a singleton) and exposes:

`updateProfile(Map<String, dynamic> fields)` — Calls `PATCH /api/v1/users/me` with the provided fields map. Returns `Future<ApiResponse<User>>`. The caller is responsible for interpreting the response and updating `AuthProvider`. Only non-null, non-empty values in the fields map are included in the request body.

### 4.4 Form Validation Rules

| Field | Required | Rule | Error Message |
|---|---|---|---|
| `name` | No | Max 100 characters. If provided, must not be blank. | `"Name must be 100 characters or fewer."` |
| `bio` | No | Max 500 characters. | `"Bio must be 500 characters or fewer."` |
| `phone` | No | Must match the pattern `^\+?[0-9]{7,15}$` if non-empty. Empty string is valid and clears the field. | `"Enter a valid phone number."` |

### 4.5 Migration Pattern for Future Feature Tickets

All future feature tickets that replace local mock implementations must follow this pattern:

1. Create `internal/repository/<domain>_repository.go` with GORM queries. All queries use `db.WithContext(ctx)`.
2. Create `internal/service/<domain>_service.go` with business logic. Methods accept typed request structs and return typed results plus an error.
3. Create `internal/handler/<domain>_handler.go` that parses HTTP, calls the service, and maps errors to HTTP status codes.
4. Register routes in `router/router.go`.
5. In Flutter, create `lib/services/<domain>_service.dart` that calls `ApiClient` instead of SharedPreferences or mock data.
6. Update the relevant Flutter screens to use the new service. Remove all `StorageHelper` references and mock data reads for the domain.
7. Mark the corresponding section of `lib/data/mock_data.dart` with a deprecation comment and return an empty list. Do not delete the function signature until all callers have been updated.

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| `PATCH` | `/api/v1/users/me` | Update the authenticated user's profile fields | Yes |

Note: `GET /api/v1/auth/me` (AUTH-01, Section 5.7) already handles reading the current user's profile. No new read endpoint is needed in this ticket.

### 5.2 PATCH /api/v1/users/me

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `name` | string | No | 1–100 characters if provided; empty string is rejected |
| `bio` | string | No | Max 500 characters if provided; empty string clears the field |
| `phone` | string | No | Must match `^\+?[0-9]{7,15}$` if non-empty; empty string clears the field |

All fields are optional. An empty JSON object `{}` is a valid request and returns the current user unchanged.

**Response `200`:**

| Field | Type | Description |
|---|---|---|
| `id` | string | ULID of the user |
| `email` | string | User's email address |
| `name` | string | Current display name |
| `bio` | string | Current bio, may be empty |
| `phone` | string | Current phone number, may be empty |
| `avatar_url` | string | Profile photo CDN URL, may be empty |
| `role` | string | `user`, `staff`, or `admin` |
| `points` | int | Current coin/point balance |
| `is_staff_registered` | bool | Whether staff registration is complete |
| `created_at` | string | ISO 8601 timestamp |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed JSON body |
| `401` | `unauthorized` | Missing or invalid JWT |
| `422` | `validation_error` | `name` exceeds 100 chars; `bio` exceeds 500 chars; `phone` is non-empty and fails format check; or `name` is an empty string |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")` (set by `JWTMiddleware`).
2. Parse the request body into `UpdateUserRequest` DTO. Return `400 bad_request` if JSON is malformed.
3. Call `UserService.UpdateProfile(ctx, userID, req)`. Map returned typed errors to HTTP status codes.
4. Return `200` with the updated `UserResponse` DTO.

### 5.3 Service Responsibilities

`UserService`:

- `UpdateProfile(ctx context.Context, userID string, req UpdateUserRequest) (User, error)` — Validates field lengths and phone format, returning `model.ErrValidation` if any rule fails. Calls `UserRepository.Update(ctx, userID, fields)` with only the fields present in the request. Returns the updated `User` model.

The `UpdateProfile` method must not allow modification of `role`, `points`, `is_active`, `password_hash`, or `google_id` — even if those fields are somehow present in the request body, the DTO must not contain them.

---

## 6. Database

Not applicable for this ticket. The `users` table and all columns updated by this ticket (`name`, `bio`, `phone`, `avatar_url`) are fully defined in DB-01 Section 6.1. No new tables or columns are introduced.

---

## 7. Configuration

Not applicable for this ticket. No new environment variables are required.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Valid PATCH with `name` and `bio` | `200`, updated user object returned; `users` row in DB reflects changes |
| 2 | Valid PATCH with empty body `{}` | `200`, user returned unchanged |
| 3 | `name` is 101 characters | `422 validation_error` |
| 4 | `name` is `""` (empty string) | `422 validation_error` |
| 5 | `bio` is 501 characters | `422 validation_error` |
| 6 | `phone` is `"abc123"` | `422 validation_error` |
| 7 | `phone` is `""` (empty string) | `200`, phone field cleared in DB |
| 8 | `phone` is `"+81901234567"` | `200`, phone stored correctly |
| 9 | No Authorization header | `401 unauthorized` |
| 10 | Expired access token | `401 unauthorized` |
| 11 | Malformed JSON body | `400 bad_request` |
| 12 | DB unavailable | `500 server_error` |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Valid form submit | Save button shows loading; green SnackBar shown; navigate back to profile |
| 2 | Name > 100 characters typed | Inline validation error shown; Save tap does not make API call |
| 3 | Bio > 500 characters typed | Inline validation error shown |
| 4 | Invalid phone format entered | Inline validation error shown |
| 5 | Network error on save | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| 6 | Server error (500) on save | Red SnackBar: `"Something went wrong. Please try again later."` |
| 7 | Profile screen opened after edit | Updated name and bio are visible |
| 8 | App restarts after profile edit | `AuthProvider.restoreSession()` loads updated data via `GET /api/v1/auth/me`; no SharedPreferences read for profile |
| 9 | `AppModeService` accessed on startup | Returns role from `AuthProvider.currentUser.role` without reading SharedPreferences |

---

## 9. Security Checklist

- [ ] Profile updates are scoped to the authenticated user only — the user ID is read from the JWT via `c.Locals("userID")`, never from the request body.
- [ ] No user can modify another user's profile via this endpoint.
- [ ] Input is validated server-side before the database write. Client-side validation is supplementary only.
- [ ] `name`, `bio`, and `phone` are trimmed of leading and trailing whitespace before persistence.
- [ ] Sensitive fields (`password_hash`, `google_id`, `role`, `points`, `is_active`, `is_staff_registered`) cannot be modified via this endpoint. The `UpdateUserRequest` DTO does not contain these fields.
- [ ] Error responses do not expose internal Go error details, stack traces, or raw database messages.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Screen | `lib/screens/profile_edit_screen.dart` | Modify — remove StorageHelper usage; pre-fill from AuthProvider; call UserService on save |
| Flutter — Screen | `lib/screens/profile_settings_screen.dart` | Modify — remove StorageHelper reads; read from AuthProvider.currentUser |
| Flutter — Service | `lib/services/user_service.dart` | Create — wraps PATCH /api/v1/users/me |
| Flutter — Service | `lib/services/app_mode_service.dart` | Modify — remove SharedPreferences profile and login state; derive role from AuthProvider |
| Flutter — Provider | `lib/providers/auth_provider.dart` | Modify — add updateCurrentUser(User) method |
| Flutter — Util | `lib/utils/storage_helper.dart` | Modify — add deprecation comment block at top of file |
| Go — Handler | `internal/handler/user_handler.go` | Create — handles PATCH /api/v1/users/me |
| Go — Service | `internal/service/user_service.go` | Modify — add UpdateProfile(ctx, userID, req) method |
| Go — Repository | `internal/repository/user_repository.go` | Modify — add Update(ctx, id, fields) method |
| Go — DTO | `internal/dto/user_dto.go` | Create — UpdateUserRequest, UserResponse |
| Go — Router | `router/router.go` | Modify — register PATCH /api/v1/users/me under the JWT-protected group |

---

## 11. Acceptance Criteria

- [ ] `PATCH /api/v1/users/me` updates `name`, `bio`, and `phone` in the `users` table and returns the full updated user object as `200`.
- [ ] `PATCH /api/v1/users/me` with an empty body `{}` returns `200` with the current user unchanged.
- [ ] `name` value longer than 100 characters returns `422 validation_error`.
- [ ] `bio` value longer than 500 characters returns `422 validation_error`.
- [ ] `phone` with an invalid format returns `422 validation_error`.
- [ ] `phone` set to empty string clears the field and returns `200`.
- [ ] `PATCH /api/v1/users/me` without a valid JWT returns `401 unauthorized`.
- [ ] The `role`, `points`, and `password_hash` fields cannot be changed via this endpoint.
- [ ] Profile edit screen pre-populates all fields from `AuthProvider.currentUser`, not from SharedPreferences.
- [ ] Saving a valid profile update shows green SnackBar `"Profile updated successfully."` and reflects changes on the profile screen.
- [ ] `AppModeService` no longer reads `app_mode`, `user_profile`, or `staff_profile` from SharedPreferences after this ticket.
- [ ] `AuthProvider.updateCurrentUser(user)` updates `currentUser` in memory and notifies listeners.
- [ ] All API endpoints return the standard error envelope for non-2xx responses.
- [ ] Loading state (`CircularProgressIndicator` 20×20 px) is shown while the save action is in progress and is always cleared in a finally block.
- [ ] Errors are displayed via red SnackBar per `_CONVENTIONS.md` Section 10.
