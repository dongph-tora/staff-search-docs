# [AUTH-02] Google Sign-In

**Type:** Feature / Auth Flow
**Priority:** P1 — Phase 1 MVP
**Estimate:** 1.5 days manual | 0.25 days vibe coding
**Dependencies:** AUTH-01 (Email/Password — must be completed first; `ApiClient` and `AuthProvider` are created there), AUTH-06 (Login/Logout flow), AUTH-08 (RBAC)
**Conventions:** See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format, JWT middleware, ApiClient, AuthProvider, navigation, and token storage rules.
**Platforms:** iOS · Android · Web

---

## 1. Overview

Adds Google OAuth as an alternative sign-in method on the Login screen. The Flutter app uses the `google_sign_in` SDK to obtain a Google `id_token` on-device. That token is sent to the Go backend, which verifies it server-side against Google's public keys, then upserts the user in PostgreSQL and returns a JWT pair — identical in shape to the AUTH-01 login response.

The `AuthProvider` (created in AUTH-01) handles token storage and state update after a successful Google sign-in, the same way it does for email/password login. No new auth state infrastructure is needed.

### End-to-end flow

```
[Flutter App]                         [Go API]                    [Google]
     │                                    │                           │
     │── tap "Sign in with Google" ──────►│                           │
     │◄─────── Google Sign-In SDK ────────┼──────── OAuth ───────────►│
     │         account picker             │                           │
     │◄─────── id_token ──────────────────┼◄──── signed JWT ──────────│
     │                                    │                           │
     │── POST /api/v1/auth/google ────────►│                           │
     │   { id_token }                     │── idtoken.Validate() ────►│
     │                                    │◄── { sub, email, name } ──│
     │                                    │── upsert users table       │
     │                                    │── GenerateTokenPair()      │
     │◄── { access_token,                 │                           │
     │      refresh_token, user } ────────│                           │
     │                                    │                           │
     │── AuthProvider stores tokens       │                           │
     │── navigate to /home                │                           │
```

---

## 2. User Flows

### 2.1 New user — first-time Google sign-in

1. User opens the app and lands on the Login screen.
2. User taps **"Sign in with Google"**.
3. The native Google account picker appears (bottom sheet on iOS/Android, popup on Web).
4. User selects a Google account.
5. The app receives an `id_token` from the SDK.
6. The app calls `POST /api/v1/auth/google` via `ApiClient`.
7. The backend verifies the token, creates a new user record, and returns a JWT pair.
8. `AuthProvider` stores the tokens and sets `currentUser`.
9. App navigates to `/home` via `pushNamedAndRemoveUntil`.
10. If `user.privacy_policy_accepted` is `false`, the Privacy Policy Consent Modal is shown before the home screen renders. Full spec for this modal — UI, backend endpoint, and accept flow — is defined in **AUTH-01 Section 3.3**.

### 2.2 Existing email/password user — first Google sign-in

- The backend extracts the email from the verified Google token and finds the matching user record.
- The `google_id` is linked to that account. No duplicate row is created.
- The backend returns a JWT for the original account, with `is_new_user: false`.

### 2.3 Returning Google user

1. User taps **"Sign in with Google"**.
2. On iOS/Android, if only one account is stored on the device, the OS may resolve it silently.
3. The app sends the `id_token` to the backend. The backend returns a fresh JWT pair.
4. `AuthProvider` updates state and navigates to `/home`.

### 2.4 Error cases

| Scenario | Behaviour |
|---|---|
| User dismisses the Google account picker | Return to Login silently — no toast, no error |
| Backend returns `400 invalid_token` | Show red SnackBar: *"Google authentication failed. Please try again."* |
| Network error (no response from backend) | Show red SnackBar: *"Unable to connect. Check your network and retry."* |
| Backend returns `403 account_disabled` | Show red SnackBar: *"This account has been suspended."* Sign the user out of Google locally to clear the cached session. |
| Web browser blocks the popup | Show orange SnackBar: *"Please allow popups from this site."* |

---

## 3. UI / Screen

### 3.1 Changes to `login_screen.dart`

A **"Sign in with Google"** button is inserted between the `or` divider and the registration button. No other layout changes are needed.

```
┌─────────────────────────────────────┐
│  🔍 スタッフサーチ                    │
│  最適なスタッフをすぐに見つける         │
│                                     │
│  [User]  [Staff]   ← Tab            │
│                                     │
│  📧 Email address                   │
│  🔒 Password                        │
│                                     │
│  [          Login          ]        │
│        Forgot your password?        │
│                                     │
│  ──────────── or ────────────       │
│                                     │
│  [G]  Sign in with Google           │  ← NEW (this feature)
│                                     │
│  ─────────────────────────────      │
│                                     │
│  [     New User Registration  ]     │
│                                     │
│  ┌── Demo Account ──────────────┐   │
│  │  [▶ User Demo Account     ]  │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

The loading spinner on the Google button must block the email/password Login button and the registration button simultaneously — a single `_isLoading` flag governs all buttons on the screen.

### 3.2 Google button specifications

| Property | Value |
|---|---|
| Height | 56px |
| Width | Full width (same as Login button) |
| Background colour | `#FFFFFF` |
| Border | 1px solid `#DADCE0` |
| Border radius | 12px |
| Icon | Official Google logo — 24px, left-aligned, 16px padding from edge |
| Label | `Sign in with Google` |
| Font | Roboto Medium (or system default medium weight), 16px, colour `#3C4043` |
| Loading state | 20×20 `CircularProgressIndicator` (strokeWidth 2) replaces label; button disabled |
| Pressed background | `#F8F9FA` |

> **Brand requirement:** The Google logo must not be recoloured, stretched, or obscured. Follow the [Google Sign-In Branding Guidelines](https://developers.google.com/identity/branding-guidelines). Place the official SVG asset at `assets/icons/google_logo.svg`.

---

## 4. Frontend — Flutter

### 4.1 Required packages

| Package | Version | Purpose |
|---|---|---|
| `google_sign_in` | `^6.2.1` | Native Google OAuth flow on all three platforms |
| `flutter_secure_storage` | `^9.2.2` | Already required by AUTH-01; listed here for completeness |

### 4.2 New service: `GoogleSignInService`

Create: `lib/services/google_sign_in_service.dart`

This service is responsible for the Google-side half of the flow only. It does not call the backend — that is the responsibility of `AuthProvider`.

**Responsibilities:**

1. Initialise `GoogleSignIn` with scopes `['email', 'profile']` and, for Web, the Web Client ID.
2. Expose a `signIn()` method that triggers the native account picker and returns either a `GoogleCredential` (containing `idToken` and `accessToken`) or a `cancelled` indicator if the user dismisses the picker.
3. Expose a `signOut()` method that disconnects the Google session on the device. This is called by `AuthProvider.logout()` in addition to clearing JWT tokens.
4. Handle the case where `idToken` comes back null (possible on some Android configurations) and surface it as an error.

**Return type:** A sealed result type (or equivalent) with three states:
- `GoogleSignInSuccess` — carries the `idToken` string
- `GoogleSignInCancelled` — user dismissed the picker
- `GoogleSignInError` — carries an error message string

### 4.3 Changes to `AuthProvider`

`AuthProvider` (created in AUTH-01, at `lib/providers/auth_provider.dart`) gains one new method:

**`loginWithGoogle()`**

Responsibilities (in order):
1. Call `GoogleSignInService.signIn()`.
2. If result is `cancelled`, return immediately — no state change, no error.
3. If result is `error`, set the error message on the provider and return.
4. Set `isLoading = true`.
5. Call `ApiClient.post('/api/v1/auth/google', { 'id_token': idToken })`.
6. On `200`: store tokens in `FlutterSecureStorage`, set `currentUser` from response, set `isLoggedIn = true`, clear any error.
7. On `400` or `403`: call `GoogleSignInService.signOut()` to clean up the Google session, then set the error message from the response body.
8. On network error: set a generic connection error message.
9. Always set `isLoading = false` when done.

### 4.4 Changes to `LoginScreen`

- Add a `_handleGoogleSignIn()` method that calls `context.read<AuthProvider>().loginWithGoogle()`, then listens for `AuthProvider.isLoggedIn` to navigate, or shows a SnackBar if `AuthProvider.errorMessage` is set.
- The Google button's `onPressed` calls `_handleGoogleSignIn()` and is disabled when `authProvider.isLoading` is true.

### 4.5 Platform configuration

#### Android
- The app package name and **SHA-1 fingerprint** (both debug and release keystores) must be registered in Google Cloud Console as an Android OAuth client.
- `google-services.json` downloaded from Google Cloud Console must be placed at `android/app/google-services.json`.
- The file must include a Web Client entry (type `3`) — this is what enables `id_token` generation. If only an Android entry (type `1`) is present, `idToken` will be null.

#### iOS
- The **Reversed Client ID** from `GoogleService-Info.plist` must be registered as a custom URL scheme in `ios/Runner/Info.plist` under `CFBundleURLSchemes`. This allows Google to redirect back to the app after the browser-based auth step.
- `GoogleService-Info.plist` must be placed at `ios/Runner/GoogleService-Info.plist`.

#### Web
- The **Web Client ID** must be declared as a `<meta name="google-signin-client_id">` tag in `web/index.html`.
- The authorised JavaScript origin for each environment (local, staging, production) must be added in Google Cloud Console.
- `GoogleSignIn` must be initialised with `clientId: 'YOUR_WEB_CLIENT_ID'` explicitly because the meta tag alone is not sufficient for the Flutter Web SDK.

---

## 5. Backend — Go + Fiber

### 5.1 New endpoint

| Method | Route | Auth required |
|---|---|---|
| POST | `/api/v1/auth/google` | No — exempt from `JWTMiddleware` |

This endpoint is added to the existing `/api/v1/auth` group in `router/router.go`.

### 5.2 POST /api/v1/auth/google — full contract

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `id_token` | string | Yes | Google ID Token obtained by the Flutter app from the `google_sign_in` SDK |

**Response `201 Created`** — when a new user is created:

| Field | Type | Description |
|---|---|---|
| `access_token` | string | JWT (HS256, 1-hour expiry) |
| `refresh_token` | string | Long-lived token (30-day expiry) |
| `token_type` | string | Always `"Bearer"` |
| `expires_in` | integer | `3600` |
| `user.id` | string | ULID |
| `user.email` | string | From Google profile |
| `user.name` | string | From Google profile |
| `user.avatar_url` | string | Profile picture URL from Google |
| `user.role` | string | `"user"` (new Google users are always `user` by default) |
| `user.is_staff` | boolean | `false` for new users |
| `user.is_new_user` | boolean | `true` |
| `user.points` | integer | `0` |
| `user.privacy_policy_accepted` | boolean | `false` for new users |
| `user.created_at` | string | ISO 8601 |

**Response `200 OK`** — when an existing user is found (by `google_id` or email):

Same shape as above, with `is_new_user: false`. `points`, `role`, `is_staff` reflect the existing account values.

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Request body is missing or `id_token` field is absent |
| `400` | `invalid_token` | Token fails Google verification (expired, wrong audience, tampered) |
| `403` | `account_disabled` | User record has `status = 'disabled'` |
| `500` | `server_error` | Unexpected internal error — DB failure, token generation failure |

### 5.3 Handler responsibilities (`auth_handler.go` — `GoogleSignIn` method)

Steps performed in order:

1. Parse the request body. Return `400 bad_request` if parsing fails or `id_token` is empty.
2. Call `AuthService.VerifyGoogleToken(ctx, idToken)`. Return `400 invalid_token` on any error — do not expose the internal verification error message.
3. Call `UserService.UpsertGoogleUser(ctx, googleUserInfo)`. Return `500` on DB error.
4. Check `user.Status`. Return `403 account_disabled` if `"disabled"`.
5. Call `JWTService.GenerateTokenPair(userID, role)`. Return `500` on failure.
6. Store the refresh token hash in the `refresh_tokens` table (same as AUTH-01 login).
7. Return `201` if `isNew`, else `200`, with the full response body.

### 5.4 AuthService — `VerifyGoogleToken`

New method added to the existing `AuthService` in `internal/service/auth_service.go`.

**Responsibilities:**

1. Call `idtoken.Validate(ctx, idToken, cfg.GoogleClientID)` from the `google.golang.org/api/idtoken` library.
2. The `audience` parameter must exactly match `GOOGLE_CLIENT_ID` env var. Any mismatch causes rejection.
3. Extract from the validated payload: `Subject` → `GoogleID`, `"email"` claim, `"name"` claim, `"picture"` claim → `AvatarURL`, `"email_verified"` claim → `Verified`.
4. Return a `GoogleUserInfo` struct. Return an error for any validation failure without revealing specifics.

**New Go dependency:** `go get google.golang.org/api/idtoken`

### 5.5 UserService — `UpsertGoogleUser`

New method added to `internal/service/user_service.go`.

Lookup precedence (stops at first match):

1. **By `google_id`** — call `UserRepository.FindByGoogleID(googleID)`. If found: call `UpdateLastLogin(userID)`, return `(user, isNew=false, nil)`.
2. **By `email`** — call `UserRepository.FindByEmail(email)`. If found: call `UserRepository.LinkGoogleProvider(userID, googleID, avatarURL)`, call `UpdateLastLogin(userID)`, return `(user, isNew=false, nil)`.
3. **Create new** — build a `User` struct with a generated ULID, `auth_provider = "google"`, `role = "user"`, `status = "active"`, `points = 0`, `is_verified = googleUserInfo.Verified`. Call `UserRepository.Create(user)`. Return `(user, isNew=true, nil)`.

### 5.6 UserRepository — new methods

These methods are added to `internal/repository/user_repository.go`:

| Method | Query description |
|---|---|
| `FindByGoogleID(googleID string)` | `SELECT * FROM users WHERE google_id = $1 LIMIT 1` |
| `LinkGoogleProvider(userID, googleID, avatarURL string)` | `UPDATE users SET google_id = $2, avatar_url = $3, auth_provider = CASE WHEN auth_provider = 'email' THEN 'google' ELSE 'multiple' END, updated_at = NOW() WHERE id = $1` |

---

## 6. Database

### 6.1 Migration

**File:** `migrations/20260310000002_add_oauth_fields_to_users.up.sql`
**Reverse:** `migrations/20260310000002_add_oauth_fields_to_users.down.sql` (drops the added columns and index)

New columns added to `users` table:

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `google_id` | `VARCHAR(255) UNIQUE` | Yes | `NULL` | Google's `sub` claim — unique user identifier from Google |
| `apple_id` | `VARCHAR(255) UNIQUE` | Yes | `NULL` | Reserved for AUTH-03 (Apple Sign-In) |
| `avatar_url` | `TEXT` | Yes | `NULL` | Profile picture URL from OAuth provider |
| `is_verified` | `BOOLEAN` | No | `FALSE` | Whether the email is verified by the identity provider |
| `auth_provider` | `VARCHAR(50)` | No | `'email'` | `'email'` \| `'google'` \| `'apple'` \| `'multiple'` |
| `status` | `VARCHAR(20)` | No | `'active'` | `'active'` \| `'disabled'` \| `'pending'` |

New index: partial index on `google_id WHERE google_id IS NOT NULL` for fast lookups.

### 6.2 Full `users` table schema (after this migration)

This is the canonical definition going forward. AUTH-01 and AUTH-02 migrations together produce this table.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `VARCHAR(26)` PK | No | ULID, generated in Go |
| `email` | `VARCHAR(255)` UNIQUE | No | |
| `password_hash` | `VARCHAR(255)` | Yes | `NULL` for pure-OAuth accounts |
| `name` | `VARCHAR(255)` | No | |
| `phone_number` | `VARCHAR(20)` | Yes | |
| `avatar_url` | `TEXT` | Yes | |
| `bio` | `TEXT` | Yes | |
| `role` | `VARCHAR(20)` | No | `'user'` \| `'staff'` \| `'admin'` · default `'user'` |
| `is_staff` | `BOOLEAN` | No | Default `false` |
| `is_verified` | `BOOLEAN` | No | Default `false` |
| `google_id` | `VARCHAR(255)` UNIQUE | Yes | |
| `apple_id` | `VARCHAR(255)` UNIQUE | Yes | Reserved |
| `auth_provider` | `VARCHAR(50)` | No | Default `'email'` |
| `status` | `VARCHAR(20)` | No | Default `'active'` |
| `points` | `INTEGER` | No | Default `0` |
| `privacy_policy_accepted` | `BOOLEAN` | No | Default `false` |
| `privacy_policy_version` | `VARCHAR(20)` | Yes | |
| `last_login_at` | `TIMESTAMPTZ` | Yes | |
| `created_at` | `TIMESTAMPTZ` | No | Default `NOW()` |
| `updated_at` | `TIMESTAMPTZ` | No | Default `NOW()` |

---

## 7. JWT Token Contract

Identical to AUTH-01. Reproduced here for reference.

### 7.1 Access token payload

| Claim | Value |
|---|---|
| `sub` | Internal user ID (ULID string) |
| `email` | User email |
| `role` | `"user"` \| `"staff"` \| `"admin"` |
| `iat` | Issued-at (Unix timestamp) |
| `exp` | `iat + JWT_ACCESS_EXPIRY` (default `iat + 3600`) |

Algorithm: `HS256`. Secret: `JWT_SECRET` env var.

### 7.2 Refresh token

Same policy as AUTH-01: 30-day expiry, stored as SHA-256 hash in `refresh_tokens` table, rotation on every use.

---

## 8. Configuration

### 8.1 New backend environment variable

| Variable | Description |
|---|---|
| `GOOGLE_CLIENT_ID` | Web Client ID from Google Cloud Console. Used as the `audience` in `idtoken.Validate`. Must match the client ID embedded in the Flutter app for Web. |

> `GOOGLE_CLIENT_SECRET` is **not** required for server-side `id_token` verification.

All other env vars (JWT, DB, SMTP) are defined in AUTH-01.

### 8.2 Google Cloud Console setup checklist

1. Create or select a project.
2. Enable the **Google People API**.
3. Create OAuth 2.0 Client IDs — one per platform:
   - **Android:** package name + SHA-1 fingerprint (debug and release)
   - **iOS:** bundle identifier
   - **Web:** authorised JS origins per environment (`http://localhost:PORT` for dev, production domain for prod)
4. Download `google-services.json` → `android/app/google-services.json`
5. Download `GoogleService-Info.plist` → `ios/Runner/GoogleService-Info.plist`
6. Copy the **Web Client ID** → `GOOGLE_CLIENT_ID` env var on the backend

### 8.3 Flutter asset

Place the official Google logo SVG at `assets/icons/google_logo.svg` and declare it in `pubspec.yaml` under `flutter.assets`.

---

## 9. Testing

### 9.1 Backend test cases

| # | Scenario | Expected |
|---|---|---|
| 1 | Valid `id_token`, email not in DB | `201`, new user row, `is_new_user: true`, `auth_provider = 'google'` |
| 2 | Valid `id_token`, email exists from email/pw signup | `200`, `google_id` linked on existing row, `is_new_user: false` |
| 3 | Valid `id_token`, `google_id` already linked | `200`, `last_login_at` updated, `is_new_user: false` |
| 4 | Expired `id_token` | `400 invalid_token` |
| 5 | Random string as `id_token` | `400 invalid_token` |
| 6 | Valid token, wrong `aud` claim | `400 invalid_token` |
| 7 | Valid token, `user.status = 'disabled'` | `403 account_disabled` |
| 8 | Missing `id_token` field in body | `400 bad_request` |
| 9 | Empty body | `400 bad_request` |
| 10 | DB unreachable during upsert | `500 server_error` |

### 9.2 Flutter test cases

| # | Scenario | Expected |
|---|---|---|
| 1 | User taps Back on Google account picker | Login screen unchanged, no toast |
| 2 | Successful sign-in, new user | Navigate to `/home`, `AuthProvider.isLoggedIn = true` |
| 3 | Successful sign-in, returning user | Navigate to `/home` |
| 4 | New user with `privacy_policy_accepted = false` | Privacy Policy modal shown before home |
| 5 | Network unreachable | Red SnackBar: "Unable to connect…" |
| 6 | Backend `400` | Red SnackBar: "Google authentication failed. Please try again." |
| 7 | Backend `403` | Red SnackBar: "This account has been suspended." + Google session cleared |
| 8 | `idToken` is null from SDK (Android misconfiguration) | Red SnackBar: "Google authentication failed. Please try again." |

---

## 10. Security Checklist

- [ ] `id_token` is verified server-side using `google.golang.org/api/idtoken` — client-supplied claims are never trusted directly
- [ ] The `aud` claim is validated against `GOOGLE_CLIENT_ID`; mismatches are rejected
- [ ] `id_token` is never stored in the database — only `google_id` (`sub` claim) is persisted
- [ ] `google_id` is treated as sensitive; not exposed in any API response
- [ ] JWT tokens are stored in `FlutterSecureStorage`, not `SharedPreferences`
- [ ] Refresh token stored as SHA-256 hash; plaintext only exists in memory during issuance
- [ ] Refresh token rotation enforced — previous token invalidated on each use
- [ ] `/api/v1/auth/google` is rate-limited (10 requests per minute per IP)
- [ ] Failed verification attempts are logged at WARN level; token values are never written to logs
- [ ] HTTPS enforced on all production environments

---

## 11. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — UI | `lib/screens/login_screen.dart` | Add Google button widget; add `_handleGoogleSignIn()` that calls `AuthProvider.loginWithGoogle()` |
| Flutter — Service | `lib/services/google_sign_in_service.dart` | **Create new** — wraps `google_sign_in` SDK |
| Flutter — Provider | `lib/providers/auth_provider.dart` | Add `loginWithGoogle()` method (file created in AUTH-01) |
| Flutter — Model | `lib/models/user.dart` | Add `googleId` (String?) and `authProvider` (String) fields |
| Flutter — Assets | `assets/icons/google_logo.svg` | **Add asset** |
| Flutter — Config | `pubspec.yaml` | Add `google_sign_in ^6.2.1`; declare new asset |
| Android | `android/app/google-services.json` | Replace with download from Google Console |
| iOS | `ios/Runner/GoogleService-Info.plist` | Replace with download from Google Console |
| iOS | `ios/Runner/Info.plist` | Add `CFBundleURLSchemes` entry (reversed client ID) |
| Web | `web/index.html` | Add `google-signin-client_id` meta tag |
| Go — Handler | `internal/handler/auth_handler.go` | Add `GoogleSignIn()` method |
| Go — Service | `internal/service/auth_service.go` | Add `VerifyGoogleToken()` method |
| Go — Service | `internal/service/user_service.go` | Add `UpsertGoogleUser()` method |
| Go — Repository | `internal/repository/user_repository.go` | Add `FindByGoogleID()` and `LinkGoogleProvider()` methods |
| Go — Router | `router/router.go` | Register `POST /api/v1/auth/google` in auth group |
| Go — Config | `internal/config/config.go` | Add `GoogleClientID` field loaded from `GOOGLE_CLIENT_ID` env var |
| Go — Dep | `go.mod` / `go.sum` | Add `google.golang.org/api` |
| DB Migration | `migrations/20260310000002_add_oauth_fields_to_users.up.sql` | **Create new** |
| DB Migration | `migrations/20260310000002_add_oauth_fields_to_users.down.sql` | **Create new** |
| Flutter — UI | *(no new file)* Privacy Policy modal is triggered in the same post-login navigation guard. Spec in AUTH-01 Section 3.3. | Reuse existing logic |
| Config | `.env.example` | Add `GOOGLE_CLIENT_ID` |

---

## 12. Acceptance Criteria

- [ ] "Sign in with Google" button renders with correct Google branding on all three platforms
- [ ] Full flow succeeds on a real iOS device (not simulator)
- [ ] Full flow succeeds on a real Android device (not emulator)
- [ ] Full flow succeeds on Web in Chrome
- [ ] New user is created in DB with `auth_provider = 'google'`, `status = 'active'`, `points = 0`
- [ ] Pre-existing email/password user has `google_id` linked — no duplicate row
- [ ] Returning Google user receives a fresh JWT without creating a new account
- [ ] Dismissing the account picker returns to Login with no error or state change
- [ ] Access and refresh tokens are stored in `FlutterSecureStorage`
- [ ] `AuthProvider.isLoggedIn` is `true` and `AuthProvider.currentUser` is populated after success
- [ ] Backend rejects a tampered token with HTTP `400`
- [ ] Suspended account receives HTTP `403` and Google session is cleared locally
- [ ] Privacy Policy modal appears for new users before entering `/home`
