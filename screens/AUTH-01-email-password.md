# [AUTH-01] Email / Password Authentication

| Field | Value |
|---|---|
| Type | Feature — Authentication |
| Priority | P1 MVP |
| Estimate | 1 day manual / 0.25 day vibe |
| Dependencies | AUTH-08 RBAC |
| Platforms | iOS, Android, Web |

---

## 1. Overview

This feature covers three sub-flows: Login, Registration, and Password Reset. All three communicate with the Go + Fiber backend over HTTPS. On a successful login or registration, the backend returns a JWT pair (access token + refresh token). The Flutter client stores both tokens in FlutterSecureStorage and updates the AuthProvider so the rest of the app can react to the authenticated state.

The current `LocalAuthService` (SharedPreferences-based) is a stub that will be replaced by `AuthProvider` backed by `ApiClient`. The migration is in-place — existing screen files are modified, not replaced.

### End-to-end flow: Login

    [Flutter App] ──── POST /api/v1/auth/login ────────────────► [Go API]
                       { email, password }                         │
                                                                   ├── verify credentials against DB
                                                                   ├── update users.last_login_at
                                                                   └── generate JWT pair (access + refresh)
                   ◄──── { access_token, refresh_token, user } ────

    [Flutter]
      ├── ApiClient.post('/auth/login') returns ApiResponse
      ├── AuthProvider.login() called
      │     ├── stores access_token → FlutterSecureStorage key: access_token
      │     ├── stores refresh_token → FlutterSecureStorage key: refresh_token
      │     └── sets currentUser from returned user object
      └── Navigator.pushNamedAndRemoveUntil('/home', (_) => false)

### End-to-end flow: Registration

    [Flutter App] ──── POST /api/v1/auth/register ──────────────► [Go API]
                       { name, email, phone?,                      │
                         password, role,                           ├── validate fields
                         privacy_policy_accepted }                 ├── check email uniqueness
                                                                   ├── hash password (bcrypt cost 12)
                                                                   ├── insert user row
                                                                   └── generate JWT pair
                   ◄──── 201 { access_token, refresh_token, user } ────

    [Flutter]
      ├── AuthProvider.register() called
      │     ├── stores tokens in FlutterSecureStorage
      │     └── sets currentUser
      └── Navigator.pushNamedAndRemoveUntil('/home', (_) => false)

### End-to-end flow: Password Reset

    [Flutter App] ──── POST /api/v1/auth/password-reset/request ──► [Go API]
                       { email }                                     │
                                                                     ├── look up email (silently ignore if not found)
                                                                     ├── generate reset token (ULID)
                                                                     ├── store SHA-256 hash in password_reset_tokens
                                                                     └── send email with reset link (expires 1 hour)
                   ◄──── 200 { message: "If that email exists, a reset link has been sent." } ────

    [Flutter]
      └── shows generic success dialog regardless of outcome

    --- user clicks link in email ---

    [Browser / App deep link] ──── POST /api/v1/auth/password-reset/confirm ──► [Go API]
                                   { token, new_password }                        │
                                                                                  ├── validate token (hash match, expiry, not used)
                                                                                  ├── update users.password_hash
                                                                                  └── mark token as used_at

                   ◄──── 200 { message: "Password updated successfully." } ────

---

## 2. Sub-flows

### 2.1 Login Flow

1. User opens the app and lands on `/login` (LoginScreen).
2. User selects a tab — [User] or [Staff] — which determines which demo account hint is shown and which registration screen the "Sign Up" link navigates to.
3. User enters email and password.
4. User taps the "Login" button.
5. Flutter calls `AuthProvider.login(email, password)`, which sets `isLoading = true` and notifies listeners.
6. `AuthProvider` delegates to `ApiClient.post('/api/v1/auth/login', body)`.
7. `ApiClient` sends the HTTP POST and awaits a response.
8. On HTTP 200: `AuthProvider` extracts `access_token`, `refresh_token`, and `user` from the response body.
9. Both tokens are written to `FlutterSecureStorage`.
10. `currentUser` is set to the deserialized `User` object.
11. `isLoading` is set to `false`; listeners are notified.
12. The screen calls `Navigator.pushNamedAndRemoveUntil('/home', (_) => false)`.
13. On any error: `isLoading` is set to `false`, an error message is surfaced to the UI.

### 2.2 Registration Flow

1. User taps "Sign Up" on the Login screen.
2. Flutter navigates to `RegisterScreen` (route varies by selected tab: user vs staff).
3. User fills in: name, email, phone (optional), password, confirm password, and ticks the Privacy Policy checkbox.
4. User taps "Register".
5. Flutter validates all fields client-side (see section 4.5). If invalid, inline error messages are shown.
6. If valid, `AuthProvider.register(name, email, phone, password, role, privacyPolicyAccepted)` is called.
7. `AuthProvider` sets `isLoading = true` and delegates to `ApiClient.post('/api/v1/auth/register', body)`.
8. On HTTP 201: tokens and user are stored identically to the login flow.
9. `Navigator.pushNamedAndRemoveUntil('/home', (_) => false)` is called.
10. On 409 (email already exists): error banner shown, user stays on the register screen.
11. On 400 (validation error): field-level errors shown where possible.

### 2.3 Password Reset Flow

1. User taps "Forgot Password?" on the Login screen.
2. A confirm dialog appears asking the user to enter their email address.
3. User enters email and confirms.
4. Flutter calls `ApiClient.post('/api/v1/auth/password-reset/request', { email })`.
5. Regardless of the HTTP response, Flutter shows a generic success message: "If that email is registered, you will receive a reset link shortly."
6. The backend (if the email matches a user) generates a reset token, stores its SHA-256 hash in `password_reset_tokens`, and sends an email containing a link in the format: `{APP_BASE_URL}/reset-password?token={plaintext_token}`.
7. User clicks the link (opens in browser or via deep link).
8. The reset page (or in-app screen) collects a new password.
9. Flutter calls `ApiClient.post('/api/v1/auth/password-reset/confirm', { token, new_password })`.
10. On success, the user is shown a confirmation and redirected to `/login`.
11. On error (expired or already-used token), a clear error message is shown.

### 2.4 Auto-login on App Start (restoreSession)

1. On app startup, before rendering any route, `AuthProvider.restoreSession()` is called.
2. `AuthProvider` reads the `access_token` value from `FlutterSecureStorage`.
3. If no token is found, `isLoggedIn` remains `false` and the app navigates to `/login`.
4. If a token exists, `ApiClient.get('/api/v1/auth/me')` is called with the token injected into the `Authorization: Bearer` header.
5. On HTTP 200: the returned user object is set as `currentUser`, `isLoggedIn` becomes `true`, and the app navigates to `/home`.
6. On HTTP 401: both `access_token` and `refresh_token` are deleted from `FlutterSecureStorage`, `currentUser` is set to null, and the app navigates to `/login`.

### 2.5 Error Cases

| Scenario | Behaviour |
|---|---|
| Wrong email or password | 401 returned; UI shows: "Invalid email or password." No distinction between the two. |
| Email not registered | Same 401 as wrong password — no "email not found" message (security) |
| Account disabled | 403 returned; UI shows: "Your account has been disabled. Contact support." |
| Email already registered (register) | 409 returned; UI shows: "An account with this email already exists." |
| Validation error — missing field | 400 returned; UI shows field-level inline error |
| Confirm password mismatch | Client-side only; UI shows: "Passwords do not match." |
| Password reset token expired | 400 returned; UI shows: "This reset link has expired. Please request a new one." |
| Password reset token already used | 400 returned; UI shows: "This reset link has already been used." |
| Network error / timeout | No response; UI shows: "Network error. Please check your connection." |
| Server error | 500 returned; UI shows: "Something went wrong. Please try again later." |
| Too many login attempts | 429 returned; UI shows: "Too many attempts. Please wait before trying again." |
| Access token expired during session | ApiClient intercepts 401, attempts refresh; if refresh fails, clears session and routes to /login |

---

## 3. UI / Screens

### 3.1 Login Screen (`login_screen.dart`)

Current location: `lib/screens/auth/login_screen.dart`

    ┌─────────────────────────────────────────┐
    │            staffsearch logo             │
    │                                         │
    │  ┌─────────────┬──────────────────────┐ │
    │  │    User     │        Staff         │ │  ← Tab selector (affects demo hints + register route)
    │  └─────────────┴──────────────────────┘ │
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │  Email                              ││  ← TextFormField
    │  └─────────────────────────────────────┘│
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │  Password                    [hide] ││  ← TextFormField (obscure)
    │  └─────────────────────────────────────┘│
    │                                         │
    │  ← Forgot Password?          [CHANGED] →│  ← calls _handlePasswordReset() → ApiClient
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │              Login                  ││  ← ElevatedButton → AuthProvider.login() [CHANGED]
    │  └─────────────────────────────────────┘│
    │                                         │
    │  ─────────────── OR ─────────────────── │
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │  Demo: demo@example.com / demo123   ││  ← stays (pre-fills fields; no LocalAuthService call)
    │  └─────────────────────────────────────┘│
    │                                         │
    │          Don't have an account?         │
    │              Sign Up →                  │  ← navigates to RegisterScreen (role from tab)
    └─────────────────────────────────────────┘

Changes required:

- Remove all imports of `LocalAuthService`.
- Replace `LocalAuthService.login()` call with `AuthProvider.login(email, password)` via `context.read<AuthProvider>()`.
- Replace `LocalAuthService.isLoggedIn` checks with `AuthProvider.isLoggedIn`.
- Show a loading indicator (CircularProgressIndicator) while `AuthProvider.isLoading` is true.
- Display API error messages from `AuthProvider.errorMessage` below the login button.
- The "Forgot Password?" handler `_handlePasswordReset()` must call `ApiClient` instead of `LocalAuthService`.
- Wrap the Login button in a conditional to disable it while `isLoading` is true.

### 3.2 Register Screen (`register_screen.dart`)

Current location: `lib/screens/auth/register_screen.dart`

    ┌─────────────────────────────────────────┐
    │           Create Account                │
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │  Full Name *                        ││
    │  └─────────────────────────────────────┘│
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │  Email *                            ││
    │  └─────────────────────────────────────┘│
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │  Phone Number (optional)            ││
    │  └─────────────────────────────────────┘│
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │  Password *                  [hide] ││
    │  └─────────────────────────────────────┘│
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │  Confirm Password *          [hide] ││
    │  └─────────────────────────────────────┘│
    │                                         │
    │  ☐ I agree to the Privacy Policy (v1.0)│  ← required checkbox
    │                                         │
    │  ┌─────────────────────────────────────┐│
    │  │             Register                ││  ← ElevatedButton → AuthProvider.register() [CHANGED]
    │  └─────────────────────────────────────┘│
    │                                         │
    │        Already have an account?         │
    │              Log In →                   │
    └─────────────────────────────────────────┘

Form field validation rules:

| Field | Required | Validation Rule |
|---|---|---|
| Full Name | Yes | Non-empty after trimming whitespace |
| Email | Yes | Valid email format (RFC 5322-like regex) |
| Phone Number | No | If provided: digits only, 7–15 characters |
| Password | Yes | Minimum 6 characters |
| Confirm Password | Yes | Must match Password field exactly |
| Privacy Policy checkbox | Yes | Must be ticked to enable Register button |

Changes required:

- Remove all imports of `LocalAuthService`.
- Replace `LocalAuthService.register()` with `AuthProvider.register(name, email, phone, password, role, privacyPolicyAccepted)`.
- Pass `role` derived from the tab that was selected on LoginScreen (passed as a constructor argument or route argument).
- Include `privacy_policy_accepted: true` and `privacy_policy_version: "v1.0"` in the registration payload.
- Show loading indicator while `AuthProvider.isLoading` is true.
- Display API error messages below the Register button.
- Map 409 error code `email_already_exists` to a field-level error on the Email field.

### 3.3 Privacy Policy Consent Modal

**When it appears:** Immediately after any login or OAuth sign-in where the returned `user.privacy_policy_accepted` is `false`. This applies to new Google Sign-In users (AUTH-02) and any legacy accounts migrated without a recorded acceptance. It does **not** appear for users who register via the standard form (Section 3.2), because the form already includes the PP checkbox.

**Behaviour:** The modal is non-dismissible — the user cannot close it without tapping "Accept". Tapping outside the modal or pressing Back does nothing.

```
┌─────────────────────────────────────────┐
│                                         │
│         Privacy Policy                  │
│                                         │
│  Before using staffsearch, please       │
│  read and accept our Privacy Policy.    │
│                                         │
│  ┌─────────────────────────────────────┐│
│  │  [scrollable Privacy Policy text   ]││  ← or a link that opens PrivacyPolicyScreen
│  │  [or a "Read Privacy Policy" link  ]││
│  └─────────────────────────────────────┘│
│                                         │
│  ┌─────────────────────────────────────┐│
│  │         Accept and Continue         ││  ← ElevatedButton
│  └─────────────────────────────────────┘│
│                                         │
└─────────────────────────────────────────┘
```

**Accept and Continue flow:**

1. User taps "Accept and Continue".
2. Button shows loading spinner; button disabled.
3. Flutter calls `POST /api/v1/auth/accept-privacy-policy` via `ApiClient` (JWT in header).
4. On `200`: `AuthProvider` updates `currentUser.privacyPolicyAccepted = true`. Modal closes. App proceeds to `/home`.
5. On any error: error SnackBar shown. Modal stays open so the user can retry.

**Implementation notes:**

- Shown as a `showDialog` with `barrierDismissible: false`.
- Triggered inside the navigation guard in `main.dart` or immediately after `AuthProvider` sets `currentUser` — whichever is cleaner, but must happen before the home screen renders.
- The `PrivacyPolicyScreen` (`lib/screens/privacy_policy_screen.dart`) already exists in the codebase and can be navigated to from the "Read Privacy Policy" link inside the modal.

### 3.4 Password Reset

The password reset UI is currently handled inline in `LoginScreen` via `_handlePasswordReset()`. No separate screen is created for Phase 1 — the flow uses a dialog.

Flow:

1. User taps "Forgot Password?" link on the Login screen.
2. A dialog appears with a single text field: "Enter your email address".
3. User taps "Send Reset Link".
4. `_handlePasswordReset()` calls `ApiClient.post('/api/v1/auth/password-reset/request', { "email": enteredEmail })`.
5. The dialog closes regardless of the response.
6. A SnackBar appears: "If that email is registered, you will receive a reset link shortly."
7. The actual password-reset confirmation form is served as a web page at `{APP_BASE_URL}/reset-password?token=...` and is out of scope for the Flutter app in Phase 1. A deep link hook will be added in a later phase.

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/services/api_client.dart` | HTTP client singleton; handles base URL, token injection, 401 auto-refresh, and typed ApiResponse<T> return |
| `lib/providers/auth_provider.dart` | ChangeNotifier that owns all auth state (currentUser, isLoggedIn, isLoading, errorMessage) and exposes login, register, logout, restoreSession |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/screens/auth/login_screen.dart` | Replace LocalAuthService calls with AuthProvider; add error display and loading state |
| `lib/screens/auth/register_screen.dart` | Replace LocalAuthService calls with AuthProvider; add Privacy Policy version to payload |
| `lib/main.dart` | Wrap app root with MultiProvider, add AuthProvider to the provider list, call AuthProvider.restoreSession() during app initialization before rendering routes |

### 4.3 ApiClient Responsibilities

ApiClient is a singleton accessed throughout the app. Its responsibilities are:

- Holds the base URL (read from environment config or a constants file, defaulting to the Go API at port 3000).
- Injects the JWT access token into every authenticated request as an `Authorization: Bearer <token>` header by reading from FlutterSecureStorage before each call.
- Provides typed methods: get, post, put, patch, delete — each returning a typed `ApiResponse<T>` that wraps the decoded body or an error.
- Intercepts 401 responses. When a 401 is received, ApiClient automatically calls POST `/api/v1/auth/refresh` with the stored refresh token. If the refresh succeeds, the original request is retried with the new access token. If the refresh fails (another 401), both tokens are deleted from storage and the app is navigated to `/login`.
- Does not throw exceptions for non-2xx responses — instead it returns an `ApiResponse` with `success: false`, the HTTP status code, the `error` code string, and the `message` string parsed from the standard error envelope.

### 4.4 AuthProvider Responsibilities

AuthProvider is a ChangeNotifier registered at the root of the widget tree via MultiProvider.

Fields:

- `currentUser` — nullable User object representing the authenticated user; null when logged out.
- `isLoggedIn` — bool derived from whether `currentUser` is non-null.
- `isLoading` — bool indicating an in-flight auth operation; used by UI to show spinners and disable buttons.
- `errorMessage` — nullable string set after a failed operation; cleared at the start of each new operation.

Methods:

- `login(email, password)` — calls ApiClient, on success stores tokens via FlutterSecureStorage and sets currentUser.
- `register(name, email, phone, password, role, privacyPolicyAccepted)` — calls ApiClient, on success stores tokens and sets currentUser.
- `logout()` — calls POST `/api/v1/auth/logout`, then regardless of response deletes both tokens from FlutterSecureStorage, sets currentUser to null, and navigates to `/login`.
- `restoreSession()` — reads access_token from FlutterSecureStorage, calls GET `/api/v1/auth/me`; on 200 sets currentUser; on 401 clears storage.

Token storage uses FlutterSecureStorage with two keys: `access_token` and `refresh_token`. Both are deleted on logout and on failed session restoration.

### 4.5 Form Validation Rules

| Field | Rule | Error Message |
|---|---|---|
| Email | Required; must match email regex | "Please enter a valid email address." |
| Password | Required; minimum 6 characters | "Password must be at least 6 characters." |
| Confirm Password | Required; must equal Password field | "Passwords do not match." |
| Full Name | Required; non-empty after trim | "Please enter your name." |
| Phone Number | Optional; if present must be 7–15 digits | "Please enter a valid phone number." |
| Privacy Policy | Must be checked | "You must accept the Privacy Policy to continue." |

### 4.6 Migration Note

The existing `lib/services/local_auth_service.dart` uses SharedPreferences to simulate login with hard-coded demo credentials. After this feature is implemented, `login_screen.dart` and `register_screen.dart` must not call `LocalAuthService` for any auth operation. `LocalAuthService` may be kept in the codebase solely as a documented demo/testing fallback, clearly marked with a deprecation notice and not imported from production screens.

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| POST | /api/v1/auth/register | Create a new user account | No |
| POST | /api/v1/auth/login | Sign in and receive JWT pair | No |
| POST | /api/v1/auth/logout | Invalidate current refresh token | Yes |
| POST | /api/v1/auth/refresh | Issue new access token using refresh token | No (uses refresh_token in body) |
| GET | /api/v1/auth/me | Retrieve current user profile | Yes |
| POST | /api/v1/auth/accept-privacy-policy | Record user's acceptance of the Privacy Policy | Yes |
| POST | /api/v1/auth/password-reset/request | Send password reset email | No |
| POST | /api/v1/auth/password-reset/confirm | Set new password using reset token | No |

### 5.2 POST /api/v1/auth/register

Request body fields:

| Field | Type | Required | Validation |
|---|---|---|---|
| name | string | Yes | Non-empty, max 255 characters |
| email | string | Yes | Valid email format, lowercase-normalized |
| phone_number | string | No | If present: digits only, 7–15 characters |
| password | string | Yes | Minimum 6 characters |
| role | string | No | One of: "user", "staff"; defaults to "user" if omitted |
| privacy_policy_accepted | bool | Yes | Must be true |
| privacy_policy_version | string | No | Defaults to "v1.0" |

Response 201 — user object returned:

| Field | Type |
|---|---|
| id | string (ULID) |
| email | string |
| name | string |
| phone_number | string or null |
| role | string |
| points | int (default 0) |
| is_email_verified | bool (default false) |
| is_staff_registered | bool (default false) |
| privacy_policy_accepted | bool |
| privacy_policy_version | string |
| created_at | string (ISO 8601) |

Plus `access_token`, `refresh_token`, `token_type` ("Bearer"), and `expires_in` (seconds).

Error responses:

| HTTP | error code | Condition |
|---|---|---|
| 400 | validation_error | One or more required fields missing or invalid |
| 400 | privacy_policy_not_accepted | privacy_policy_accepted is false or missing |
| 409 | email_already_exists | Email is already registered |
| 500 | server_error | Unexpected internal error |

### 5.3 POST /api/v1/auth/login

Request body:

| Field | Type | Required |
|---|---|---|
| email | string | Yes |
| password | string | Yes |

Response 200:

| Field | Type |
|---|---|
| access_token | string (JWT) |
| refresh_token | string (opaque ULID) |
| token_type | string ("Bearer") |
| expires_in | int (seconds until access_token expires) |
| user | User object (same shape as register response) |

Error responses:

| HTTP | error code | Condition |
|---|---|---|
| 400 | bad_request | Missing email or password field |
| 401 | invalid_credentials | Email not found or password incorrect (same message — no distinction) |
| 403 | account_disabled | User account is marked inactive |
| 500 | server_error | Unexpected internal error |

### 5.4 POST /api/v1/auth/logout

Request: No body required. The JWT access token must be provided in the `Authorization: Bearer` header.

Response 200:

    { "message": "Logged out successfully." }

Behaviour: The handler extracts the user ID from the validated JWT, then deletes all `refresh_tokens` rows associated with that user from the database. This invalidates all active sessions for the user, not just the current one. (Single-device revocation can be added in a later phase by passing the refresh token in the body.)

### 5.5 POST /api/v1/auth/refresh

Request body:

    { "refresh_token": "<opaque token string>" }

Behaviour: The handler computes the SHA-256 hash of the provided token, looks it up in `refresh_tokens`, checks that `expires_at` is in the future and `used_at` is null, then issues a new JWT pair. The old refresh token row is marked with `used_at = now()` (rotation — prevents reuse). The new refresh token is inserted.

Response 200: New `access_token` and `refresh_token` pair (same shape as login response, minus the user object).

Error responses:

| HTTP | error code | Condition |
|---|---|---|
| 401 | invalid_refresh_token | Token not found in DB |
| 401 | refresh_token_expired | Token found but expires_at is past |
| 401 | refresh_token_used | Token found but used_at is already set |

### 5.6 GET /api/v1/auth/me

Authorization: Bearer header required.

Response 200: Full user object (same shape as the user field in the login response).

Error responses:

| HTTP | error code | Condition |
|---|---|---|
| 401 | unauthorized | Missing, malformed, or expired JWT |

### 5.7 POST /api/v1/auth/accept-privacy-policy

Authorization: Bearer header required.

Request body: None.

Behaviour: Sets `privacy_policy_accepted = true`, `privacy_policy_accepted_at = now()`, and `privacy_policy_version = 'v1.0'` on the authenticated user's row. Idempotent — calling it multiple times has no harmful effect.

Response `200 OK`:

| Field | Type | Value |
|---|---|---|
| `message` | string | `"Privacy policy accepted."` |
| `user` | object | Updated full user object (same shape as login response user field) |

Error responses:

| HTTP | error code | Condition |
|---|---|---|
| 401 | `unauthorized` | Missing or invalid JWT |
| 500 | `server_error` | Unexpected internal error |

Handler responsibilities:

1. Extract `userID` from JWT middleware context.
2. Call `UserService.AcceptPrivacyPolicy(userID, version)` where version is the constant `"v1.0"`.
3. Return `200` with the success message and the updated user object.

### 5.8 POST /api/v1/auth/password-reset/request

Request body:

    { "email": "user@example.com" }

Response: Always 200 regardless of whether the email exists.

    { "message": "If that email is registered, you will receive a reset link shortly." }

Behaviour: If the email is found in the `users` table, the handler generates a ULID as the plaintext reset token, computes its SHA-256 hash, stores a row in `password_reset_tokens` with `expires_at = now() + 1 hour`, and sends an email to the user with a link: `{APP_BASE_URL}/reset-password?token={plaintext_token}`. If the email is not found, the handler returns the same 200 response after a constant-time delay to prevent timing-based email enumeration.

### 5.9 POST /api/v1/auth/password-reset/confirm

Request body:

| Field | Type | Required |
|---|---|---|
| token | string | Yes |
| new_password | string | Yes |

Behaviour: Computes SHA-256 of the provided token, looks up the matching row in `password_reset_tokens`, validates that `expires_at` is in the future and `used_at` is null, then updates the user's `password_hash` with a new bcrypt hash and sets `used_at = now()` on the reset token row.

Response 200:

    { "message": "Password updated successfully. Please log in with your new password." }

Error responses:

| HTTP | error code | Condition |
|---|---|---|
| 400 | invalid_reset_token | Token not found or hash mismatch |
| 400 | reset_token_expired | Token found but expires_at is past |
| 400 | reset_token_used | Token already consumed |
| 400 | password_too_short | new_password is fewer than 6 characters |
| 500 | server_error | Unexpected internal error |

### 5.10 Handler Responsibilities

POST /api/v1/auth/register handler:

1. Parse and validate the request body against the field rules in section 5.2.
2. Normalize the email to lowercase.
3. Check whether the email already exists in the `users` table via `UserService.FindByEmail`.
4. If found, return 409 with `email_already_exists`.
5. Call `AuthService.HashPassword(password)` to produce a bcrypt hash.
6. Call `UserService.CreateUser(...)` to insert the row.
7. Call `AuthService.GenerateTokenPair(userID, role)` to produce the JWT pair.
8. Persist the refresh token hash in `refresh_tokens`.
9. Return 201 with the token pair and user object.

POST /api/v1/auth/login handler:

1. Parse and validate the request body.
2. Look up the user by email via `UserService.FindByEmail`. If not found, return 401 `invalid_credentials`.
3. Check whether the user account is active. If not, return 403 `account_disabled`.
4. Call `AuthService.VerifyPassword(storedHash, providedPassword)`. If mismatch, return 401 `invalid_credentials`.
5. Call `UserService.UpdateLastLogin(userID)` to update `last_login_at`.
6. Call `AuthService.GenerateTokenPair(userID, role)`.
7. Persist the refresh token hash in `refresh_tokens`.
8. Return 200 with the token pair and user object.

POST /api/v1/auth/logout handler:

1. Extract the user ID from the validated JWT middleware context.
2. Delete all `refresh_tokens` rows where `user_id = extractedUserID`.
3. Return 200 with a success message.

POST /api/v1/auth/refresh handler:

1. Parse and validate the request body to extract `refresh_token`.
2. Compute the SHA-256 hash of the provided token.
3. Query `refresh_tokens` by `token_hash`. If not found, return 401 `invalid_refresh_token`.
4. Check `expires_at`. If past, return 401 `refresh_token_expired`.
5. Check `used_at`. If not null, return 401 `refresh_token_used`.
6. Mark the current row as `used_at = now()`.
7. Call `AuthService.GenerateTokenPair(userID, role)` to produce a new pair.
8. Insert the new refresh token hash in `refresh_tokens`.
9. Return 200 with the new pair.

GET /api/v1/auth/me handler:

1. Extract the user ID from the JWT middleware context.
2. Call `UserService.FindByID(userID)`.
3. Return 200 with the user object. If not found (edge case), return 401.

POST /api/v1/auth/password-reset/request handler:

1. Parse and validate the request body to extract `email`.
2. Look up the user by email via `UserService.FindByEmail`.
3. If found: generate a ULID as plaintext token, compute SHA-256, insert into `password_reset_tokens` with `expires_at = now() + 1 hour`.
4. Send email asynchronously (goroutine) containing the reset link.
5. If not found: introduce a short constant-time delay to prevent timing attacks.
6. Return 200 with the generic message in all cases.

POST /api/v1/auth/password-reset/confirm handler:

1. Parse and validate the request body.
2. Validate that `new_password` is at least 6 characters; if not, return 400 `password_too_short`.
3. Compute SHA-256 of the provided token.
4. Query `password_reset_tokens` by `token_hash`. If not found, return 400 `invalid_reset_token`.
5. Check `expires_at`. If past, return 400 `reset_token_expired`.
6. Check `used_at`. If set, return 400 `reset_token_used`.
7. Call `AuthService.HashPassword(new_password)`.
8. Call `UserService.UpdatePassword(userID, newHash)`.
9. Set `used_at = now()` on the reset token row.
10. Return 200 with the success message.

### 5.11 Service Responsibilities

AuthService:

- `HashPassword(plaintext string) (string, error)` — generates a bcrypt hash at cost factor 12.
- `VerifyPassword(hash string, plaintext string) bool` — compares plaintext against bcrypt hash using constant-time comparison.
- `GenerateTokenPair(userID string, role string) (accessToken string, refreshToken string, error)` — signs a JWT access token (HS256, 1-hour expiry, payload per section 7) and generates a ULID as the opaque refresh token.
- `ValidateResetToken(tokenHash string) (*PasswordResetToken, error)` — fetches the token row, checks expiry and used_at, returns the row or an appropriate error.

UserService:

- `CreateUser(input CreateUserInput) (*User, error)` — inserts a new user row and returns the created entity.
- `FindByEmail(email string) (*User, error)` — returns the user with the matching email (case-insensitive) or nil.
- `FindByID(id string) (*User, error)` — returns the user with the matching ULID or nil.
- `UpdateLastLogin(userID string) error` — sets `last_login_at = now()` on the given user row.
- `UpdatePassword(userID string, newHash string) error` — sets `password_hash = newHash` on the given user row.
- `AcceptPrivacyPolicy(userID string, version string) (*User, error)` — sets `privacy_policy_accepted = true`, `privacy_policy_accepted_at = now()`, `privacy_policy_version = version`, returns the updated user object.

---

## 6. Database

### 6.1 Migration File

Migration file name: `20260310000001_create_users_table.up.sql`

Users table columns:

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | VARCHAR(26) | PRIMARY KEY | ULID — generated application-side |
| email | VARCHAR(255) | NOT NULL UNIQUE | Normalized to lowercase |
| password_hash | VARCHAR(255) | NOT NULL | bcrypt hash, cost factor 12 |
| name | VARCHAR(255) | NOT NULL | Display name |
| phone_number | VARCHAR(20) | NULLABLE | Optional |
| profile_image | TEXT | NULLABLE | URL to stored image |
| bio | TEXT | NULLABLE | |
| birth_date | DATE | NULLABLE | |
| gender | VARCHAR(20) | NULLABLE | |
| address | TEXT | NULLABLE | |
| role | VARCHAR(20) | NOT NULL DEFAULT 'user' | 'user', 'staff', or 'admin' |
| points | INTEGER | NOT NULL DEFAULT 0 | Coin/tip balance |
| is_email_verified | BOOLEAN | NOT NULL DEFAULT FALSE | |
| is_staff_registered | BOOLEAN | NOT NULL DEFAULT FALSE | True after staff profile created |
| privacy_policy_accepted | BOOLEAN | NOT NULL DEFAULT FALSE | |
| privacy_policy_accepted_at | TIMESTAMPTZ | NULLABLE | Timestamp of acceptance |
| privacy_policy_version | VARCHAR(20) | NULLABLE | e.g. "v1.0" |
| is_active | BOOLEAN | NOT NULL DEFAULT TRUE | False = disabled account |
| last_login_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

Required indexes:

- Unique index on `email` (enforced by UNIQUE constraint).
- Primary key on `id` (ULID — VARCHAR(26)).
- Index on `role` (for admin queries filtering by role).
- Index on `is_active` (for login check).

### 6.2 New Table: `refresh_tokens`

| Column | Type | Description |
|---|---|---|
| id | VARCHAR(26) PRIMARY KEY | ULID — generated application-side |
| user_id | VARCHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE | Owner of the token |
| token_hash | VARCHAR(64) NOT NULL UNIQUE | SHA-256 hash of the plaintext refresh token — plaintext is never stored |
| expires_at | TIMESTAMPTZ NOT NULL | 30 days from time of issue |
| used_at | TIMESTAMPTZ NULLABLE | Set to now() when rotated; prevents reuse |
| created_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

Index: unique index on `token_hash`. Index on `user_id` for bulk-delete during logout.

### 6.3 New Table: `password_reset_tokens`

| Column | Type | Description |
|---|---|---|
| id | VARCHAR(26) PRIMARY KEY | ULID |
| user_id | VARCHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE | |
| token_hash | VARCHAR(64) NOT NULL UNIQUE | SHA-256 hash of plaintext token sent in email |
| expires_at | TIMESTAMPTZ NOT NULL | 1 hour from time of issue |
| used_at | TIMESTAMPTZ NULLABLE | Set when token is consumed; single-use enforcement |
| created_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

Index: unique index on `token_hash`. Index on `user_id`.

### 6.4 Seed Data (Development Only)

The seed file `staff-search-api/seeds/demo_data.sql` inserts the two demo accounts into the `users` table using `ON CONFLICT DO NOTHING` so it is safe to run multiple times. It must never be applied to the production database.

Full specification of seed records and run instructions is defined in `_CONVENTIONS.md` — Section 11 (Demo Data Seeding).

The Flutter login screen's demo button pre-fills the form fields with the demo credentials and submits the standard `POST /api/v1/auth/login` request. No special backend endpoint is used.

---

## 7. JWT Token Contract

The access token is a signed JWT using HS256. The signing secret is read from the `JWT_SECRET` environment variable (minimum 32 characters recommended). Access tokens expire after 1 hour. Refresh tokens are opaque ULIDs stored hashed in the database; they expire after 30 days and are rotated on each use.

Access token payload:

| Claim | Type | Value |
|---|---|---|
| sub | string | User ULID (e.g. "01JXYZ...") |
| role | string | "user", "staff", or "admin" |
| email | string | User email |
| iat | int | Issued-at Unix timestamp |
| exp | int | Expiry Unix timestamp (iat + 3600) |

The full JWT contract including algorithm, rotation policy, and blacklisting strategy is defined in AUTH-02. This document inherits that contract without duplication.

---

## 8. Configuration

All configuration is provided via environment variables. No secrets are hard-coded.

| Variable | Default | Description |
|---|---|---|
| DATABASE_URL | (required) | PostgreSQL connection string |
| JWT_SECRET | (required) | HS256 signing secret — minimum 32 characters |
| JWT_ACCESS_EXPIRY | 3600 | Access token lifetime in seconds (default: 1 hour) |
| JWT_REFRESH_EXPIRY | 2592000 | Refresh token lifetime in seconds (default: 30 days) |
| SMTP_HOST | (required) | SMTP server hostname for sending emails |
| SMTP_PORT | 587 | SMTP server port |
| SMTP_USER | (required) | SMTP authentication username |
| SMTP_PASS | (required) | SMTP authentication password |
| FROM_EMAIL | (required) | Sender address for auth emails (e.g. no-reply@staffsearch.app) |
| APP_BASE_URL | (required) | Base URL for reset links (e.g. https://staffsearch.app) |
| APP_PORT | 3000 | Port on which the Fiber HTTP server listens |

---

## 9. Testing

### 9.1 Backend Test Cases

| Endpoint | Scenario | Expected Result |
|---|---|---|
| POST /auth/register | Valid new user | 201 with JWT pair and user object |
| POST /auth/register | Missing name | 400 validation_error |
| POST /auth/register | Invalid email format | 400 validation_error |
| POST /auth/register | Password < 6 chars | 400 validation_error |
| POST /auth/register | privacy_policy_accepted = false | 400 privacy_policy_not_accepted |
| POST /auth/register | Duplicate email | 409 email_already_exists |
| POST /auth/login | Valid credentials | 200 with JWT pair and user object |
| POST /auth/login | Unknown email | 401 invalid_credentials |
| POST /auth/login | Wrong password | 401 invalid_credentials (same message) |
| POST /auth/login | Disabled account | 403 account_disabled |
| POST /auth/login | Missing fields | 400 bad_request |
| POST /auth/refresh | Valid refresh token | 200 new token pair; old token marked used |
| POST /auth/refresh | Already-used token | 401 refresh_token_used |
| POST /auth/refresh | Expired token | 401 refresh_token_expired |
| POST /auth/refresh | Token not in DB | 401 invalid_refresh_token |
| GET /auth/me | Valid JWT | 200 user object |
| GET /auth/me | No JWT | 401 unauthorized |
| GET /auth/me | Expired JWT | 401 unauthorized |
| POST /auth/logout | Valid JWT | 200; all user refresh tokens deleted |
| POST /auth/password-reset/request | Registered email | 200 generic message; reset email sent |
| POST /auth/password-reset/request | Unknown email | 200 same generic message; no email sent |
| POST /auth/password-reset/confirm | Valid token + valid password | 200; password updated; token marked used |
| POST /auth/password-reset/confirm | Expired token | 400 reset_token_expired |
| POST /auth/password-reset/confirm | Already-used token | 400 reset_token_used |
| POST /auth/password-reset/confirm | Password < 6 chars | 400 password_too_short |
| POST /auth/login (rate limit) | 6th attempt in 1 minute from same IP | 429 Too Many Requests |

### 9.2 Flutter Test Cases

| Scenario | Expected Result |
|---|---|
| Login with valid credentials | AuthProvider.currentUser is set; navigates to /home |
| Login with wrong password | Error message shown; user stays on login screen; no navigation |
| Login while isLoading is true | Login button is disabled; no duplicate request |
| Register with all valid fields | AuthProvider.currentUser is set; navigates to /home |
| Register with mismatched passwords | Client-side error shown; no API call made |
| Register without ticking Privacy Policy | Register button disabled or inline error shown |
| Register with duplicate email | 409 response mapped to field-level error on Email field |
| App start with valid stored token | restoreSession sets currentUser; routes to /home |
| App start with expired/missing token | restoreSession clears storage; routes to /login |
| Logout | currentUser cleared; tokens deleted; navigates to /login |
| Token auto-refresh on 401 | ApiClient retries original request with new token transparently |
| Auto-refresh fails | Tokens cleared; navigates to /login |
| Forgot Password tap | Dialog shown; ApiClient called; SnackBar shown after response |
| Demo account pre-fill (User tab) | Email and password fields pre-filled with demo@example.com / demo123 |
| Demo account pre-fill (Staff tab) | Fields pre-filled with staff-demo@example.com / demo123 |

---

## 10. Security Checklist

- Passwords are hashed with bcrypt at cost factor 12. Plaintext passwords are never stored, logged, or returned in any response.
- Refresh tokens are stored in the database as SHA-256 hashes only. The plaintext token is generated application-side and transmitted once (in the login/register response). It is never retrievable from the database.
- Password reset tokens are stored as SHA-256 hashes. The plaintext token is included only in the email link and is never logged or returned in API responses.
- Password reset links expire after exactly 1 hour from the time of issuance.
- Password reset tokens are single-use. The `used_at` timestamp is set atomically when the token is consumed; subsequent attempts with the same token return 400 `reset_token_used`.
- The login endpoint returns the same error message and HTTP status (401 `invalid_credentials`) whether the email is not found or the password is wrong, preventing account enumeration.
- The password-reset/request endpoint always returns 200 with the same generic message regardless of whether the email is registered, preventing email enumeration. A constant-time delay is applied when no user is found.
- Rate limiting is enforced at the Fiber middleware layer: a maximum of 5 login attempts per minute per IP address on POST /auth/login.
- Rate limiting is enforced on POST /auth/password-reset/request: a maximum of 3 requests per hour per IP address.
- Refresh token rotation is enforced: each use of a refresh token invalidates it and issues a new one. Reuse of a consumed token is detected and rejected.
- JWT signing uses HS256 with a secret of at least 32 characters, read exclusively from the `JWT_SECRET` environment variable.

---

## 11. File Map

| Layer | File | Change |
|---|---|---|
| Flutter | `lib/services/api_client.dart` | Create |
| Flutter | `lib/providers/auth_provider.dart` | Create |
| Flutter | `lib/screens/auth/login_screen.dart` | Modify |
| Flutter | `lib/screens/auth/register_screen.dart` | Modify |
| Flutter | `lib/main.dart` | Modify |
| Flutter | `lib/services/local_auth_service.dart` | Modify (add deprecation notice; remove production usage) |
| Flutter | `lib/models/user.dart` | Modify (ensure all fields from domain model are present) |
| Go | `internal/handler/auth_handler.go` | Create |
| Go | `internal/service/auth_service.go` | Create |
| Go | `internal/service/user_service.go` | Create |
| Go | `internal/repository/user_repository.go` | Create |
| Go | `internal/repository/refresh_token_repository.go` | Create |
| Go | `internal/repository/password_reset_repository.go` | Create |
| Go | `internal/model/user.go` | Create |
| Go | `internal/model/refresh_token.go` | Create |
| Go | `internal/model/password_reset_token.go` | Create |
| Go | `internal/middleware/jwt_middleware.go` | Create |
| Go | `internal/email/email_service.go` | Create |
| Go | `migrations/20260310000001_create_users_table.up.sql` | Create |
| Go | `migrations/20260310000001_create_users_table.down.sql` | Create |
| Go | `migrations/20260310000002_create_refresh_tokens_table.up.sql` | Create |
| Go | `migrations/20260310000003_create_password_reset_tokens_table.up.sql` | Create |
| Go | `main.go` | Modify (register auth routes, inject dependencies) |
| Go | `.env.example` | Create / Modify (add all variables from section 8) |
| Go | `seeds/demo_data.sql` | Create — inserts two demo accounts (see `_CONVENTIONS.md` Section 11) |

---

## 12. Acceptance Criteria

- A new user can register with name, email, and password; upon success they are navigated to /home and their profile is accessible via GET /auth/me.
- A registered user can log in with correct credentials and receive a valid JWT pair stored in FlutterSecureStorage.
- A user with wrong credentials sees a generic "Invalid email or password" message and is not logged in.
- On app restart with a valid stored token, the app automatically restores the session and navigates to /home without requiring the user to log in again.
- On app restart with an expired or missing token, the app clears storage and navigates to /login.
- A logged-in user can log out; after logout the tokens are deleted, currentUser is null, and the app is on /login.
- The token refresh mechanism transparently issues a new access token when the current one expires, without requiring the user to log in again.
- A user can request a password reset; a reset link is emailed and the UI shows a generic confirmation message regardless of whether the email is registered.
- A valid password reset link allows the user to set a new password; the link cannot be reused after the first successful use.
- The two demo accounts (demo@example.com and staff-demo@example.com) can log in successfully against the real backend when the database is seeded with their records.
- Registering with an already-used email shows a specific error ("An account with this email already exists") without crashing or navigating away.
- All auth API endpoints return the standard error envelope format `{ "error": "<code>", "message": "<human string>" }` for all non-2xx responses.
- The Privacy Policy checkbox (version v1.0) is required during registration; omitting it prevents form submission.
- The Staff tab on the login screen directs to the staff registration flow; the User tab directs to the standard user registration flow.
- After any OAuth login where `user.privacy_policy_accepted` is `false`, a non-dismissible Privacy Policy consent modal appears. The user cannot reach `/home` without tapping "Accept and Continue".
- Tapping "Accept and Continue" on the PP modal calls `POST /api/v1/auth/accept-privacy-policy`; on success the modal closes and the app navigates to `/home`.
- The two demo accounts (`demo@example.com` / `demo123` and `staff-demo@example.com` / `demo123`) are present in the database after running `seeds/demo_data.sql` and can log in against the real backend successfully.
