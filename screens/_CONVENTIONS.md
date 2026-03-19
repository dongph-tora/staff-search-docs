# staffsearch — Development Conventions

This document defines the conventions that all contributors must follow when building the staffsearch platform. It covers the Go backend (staff-search-api) and the Flutter frontend (staff-search-app). All new code must conform to these standards. When existing code conflicts with these conventions, it should be migrated as part of the relevant ticket.

---

## 1. Go Backend — Project Structure

The backend lives under `staff-search-api/`. The directory layout below is the target structure all contributors must follow. New packages must be placed in the correct directory rather than added ad hoc.

| Path | Purpose |
|---|---|
| `main.go` | Entry point. Initializes configuration, database, and Redis connections, then wires the router and starts the Fiber server. Contains no business logic. |
| `router/router.go` | Registers all routes and applies middleware groups. All route definitions live here; no routes are registered inside handlers or anywhere else. |
| `internal/config/config.go` | Loads environment variables into a typed config struct using a library such as envconfig or os.Getenv. All environment access happens through this struct; scattered os.Getenv calls elsewhere are not permitted. |
| `internal/handler/` | One file per domain (for example, `auth_handler.go`, `user_handler.go`, `booking_handler.go`). Handlers parse HTTP requests, delegate to services, and write HTTP responses. No business logic lives here. |
| `internal/service/` | One file per domain. Contains all business logic. Services call repositories and return typed results with errors. They have no knowledge of HTTP or Fiber context. |
| `internal/repository/` | One file per domain. Contains all database interactions using GORM. Repositories accept plain Go types and return plain Go types. They have no knowledge of HTTP or business rules. All queries go through `db.WithContext(ctx)` for proper context propagation. |
| `internal/model/` | Go structs that represent database rows and shared domain types with GORM struct tags. These are the canonical data shapes passed between layers and used by GORM for auto-migration. |
| `internal/middleware/` | Fiber middleware functions. Each middleware lives in its own file (for example, `jwt_middleware.go`, `rate_limiter.go`, `cors.go`). |
| `internal/dto/` | Request and response data transfer objects. One file per domain (for example, `auth_dto.go`). DTOs decouple HTTP payloads from internal models. |
| `pkg/database/postgres.go` | PostgreSQL connection setup using GORM (gorm.io/driver/postgres). Returns `*gorm.DB`. |
| `pkg/cache/redis.go` | Redis client setup using go-redis/v9. |
| `pkg/jwt/jwt.go` | JWT token generation, validation, and hashing utilities. |
| `pkg/ulid/ulid.go` | ULID primary key generator. |
| `pkg/response/response.go` | Standard error envelope helpers (`BadRequest`, `Unauthorized`, `ServerError`, etc.). All handlers use these — never construct error JSON manually. |
| `migrations/` | SQL migration files in golang-migrate format. See Section 4 for naming rules. |
| `seeds/demo_data.sql` | Demo account seed data for development and staging. See Section 11. |

The three-layer pattern (handler → service → repository) is strictly enforced. A handler must never query the database directly, and a repository must never contain business rules.

---

## 2. Go Backend — Standard Error Response

Every endpoint in the API returns errors using a single, consistent JSON envelope. No endpoint is permitted to return a different error shape.

The envelope contains exactly two fields:

| Field | Type | Description |
|---|---|---|
| `error` | string | A machine-readable error code. Used by the Flutter client to branch on error type programmatically. |
| `message` | string | A human-readable description suitable for display to an end user. Must never contain internal Go error details, stack traces, or database messages in production. |

The full mapping of HTTP status codes to error codes is as follows:

| HTTP Status | `error` Code | When to Use |
|---|---|---|
| 400 | `bad_request` | Malformed request body or missing required fields |
| 400 | `invalid_token` | Token is structurally invalid or expired (used specifically in the refresh flow) |
| 401 | `unauthorized` | No token provided, or token fails signature verification |
| 403 | `forbidden` | Token is valid but the user lacks permission for the requested resource |
| 404 | `not_found` | The requested resource does not exist |
| 409 | `conflict` | The request conflicts with existing data (for example, duplicate email on register) |
| 422 | `validation_error` | The request is well-formed but fails business-level validation |
| 500 | `server_error` | An unexpected internal error occurred |

In production, the `message` field for a 500 error must be a generic phrase such as "An internal error occurred" rather than the raw Go error string. Detailed errors may be logged server-side but must not reach the client.

---

## 3. Go Backend — JWT Middleware

The middleware responsible for authenticating requests is named `JWTMiddleware` and lives in `internal/middleware/jwt_middleware.go`.

It is applied as a group-level middleware to all API routes except the public authentication endpoints listed below. These endpoints are exempt because they are either used to obtain tokens or do not require authentication.

| Exempt Route | Method |
|---|---|
| `/api/v1/auth/login` | POST |
| `/api/v1/auth/register` | POST |
| `/api/v1/auth/google` | POST |
| `/api/v1/auth/apple` | POST |
| `/api/v1/auth/refresh` | POST |

When the middleware receives a request with a valid JWT, it attaches two values to the Fiber context locals before passing control to the next handler:

| Local Key | Type | Description |
|---|---|---|
| `userID` | string | The subject claim from the token, identifying the authenticated user |
| `role` | string | The role claim from the token (`user`, `staff`, or `admin`) |

Handlers read these values using `c.Locals("userID")` and `c.Locals("role")`. They must not re-parse or re-validate the token themselves.

When the Authorization header is missing, malformed, or contains a token that fails signature verification or has expired, the middleware immediately returns a 401 response using the `unauthorized` error code and does not call the next handler.

---

## 4. Go Backend — Database & ORM

The backend uses **GORM** (`gorm.io/gorm`) as the ORM layer with the PostgreSQL driver (`gorm.io/driver/postgres`).

### 4.1 GORM Model Conventions

| Aspect | Convention |
|---|---|
| Primary key | ULID as `VARCHAR(26)`, generated in Go via `pkg/ulid.New()`. Use `gorm:"primaryKey;type:varchar(26)"` tag. Never use PostgreSQL sequences or UUID. |
| Struct tags | All model fields must have `gorm` tags for type, constraints, and indexes. JSON tags control API serialization. |
| Timestamps | `CreatedAt` and `UpdatedAt` use `gorm:"autoCreateTime"` / `gorm:"autoUpdateTime"`. |
| Associations | Use GORM associations (`foreignKey`, `constraint:OnDelete:CASCADE`) for referential integrity. |
| Sensitive fields | Use `json:"-"` to exclude from API responses (e.g., `PasswordHash`, `GoogleID`). |

### 4.2 Auto-Migration

On startup, `main.go` calls `db.AutoMigrate()` with all model structs. This handles table creation and column additions in development. For production, explicit SQL migration files in `migrations/` should be used alongside AutoMigrate.

### 4.3 Repository Conventions

| Aspect | Convention |
|---|---|
| Context | Always use `db.WithContext(ctx)` for proper cancellation and timeout propagation. |
| Error mapping | Map `gorm.ErrRecordNotFound` to `model.ErrNotFound` — never expose GORM errors to the service layer. |
| Transactions | Use `db.WithContext(ctx).Transaction(func(tx *gorm.DB) error { ... })` for multi-step operations. |

### 4.4 SQL Migrations (for production)

Migration files live in `migrations/` for explicit schema changes. Format follows golang-migrate conventions.

| Aspect | Convention |
|---|---|
| File naming | `YYYYMMDDHHMMSS_<description>.up.sql` and `YYYYMMDDHHMMSS_<description>.down.sql` |
| Description format | Lowercase with underscores (e.g., `create_users_table`, `add_staff_ranking_column`) |
| Reversibility | Every `.up.sql` must have a corresponding `.down.sql` that fully reverses the change |
| Run command | `migrate -path migrations -database $DATABASE_URL up` |

Migrations must be written to be idempotent where practical, using `IF NOT EXISTS` and `IF EXISTS` clauses in DDL statements.

---

## 5. Go Backend — Handler Conventions

Handlers have a narrow, well-defined responsibility: parse the incoming HTTP request, call the appropriate service method, and write the HTTP response. They contain no business logic.

The following rules apply to all handlers:

| Rule | Detail |
|---|---|
| Naming | Handler files are named `<domain>_handler.go`. The handler struct is named `<Domain>Handler` (for example, `AuthHandler`, `BookingHandler`). |
| Request parsing | The request body must be parsed and validated before any service call is made. If parsing fails, the handler returns 400 immediately with the `bad_request` error code. |
| Service calls | Services return a `(result, error)` tuple. The handler's job is to inspect the error type and map it to the appropriate HTTP status code and error envelope. |
| No DB access | Handlers must never import or call repository or database packages directly. All data access goes through the service layer. |
| No business logic | Conditional logic based on business rules (for example, "can this user see this resource?") belongs in the service layer, not in the handler. |

---

## 6. Flutter — API Client (`ApiClient`)

All HTTP communication between the Flutter app and the Go backend goes through a single `ApiClient` class. Scattered use of raw HTTP packages elsewhere in the app is not permitted.

| Aspect | Detail |
|---|---|
| Location | `lib/services/api_client.dart` |
| Creation timeline | Must be created as part of ticket AUTH-01 |
| Pattern | Singleton — one instance is used throughout the app |
| Base URL | Injected at build time via `--dart-define=API_BASE_URL=<url>`. Defaults to `http://localhost:3000` when no value is provided. |

On every outgoing request, the client reads the `access_token` from `FlutterSecureStorage` and attaches it as an `Authorization: Bearer <token>` header. If no token is present (for example, on public endpoints before login), the header is omitted.

The client handles 401 responses from the server with a single silent retry flow:

1. On receiving a 401, the client attempts to refresh the session by calling `POST /api/v1/auth/refresh` with the stored `refresh_token`.
2. If the refresh call succeeds, the client stores the new tokens and retries the original failed request exactly once.
3. If the refresh call fails, the client clears all stored tokens from `FlutterSecureStorage` and redirects the user to `/login` by pushing a named route and clearing the back stack.

The client returns a typed `ApiResponse<T>` object for every call. This type has the following fields:

| Field | Type | Description |
|---|---|---|
| `success` | bool | True if the request completed with a 2xx status code |
| `data` | T? | The deserialized response body on success, null on error |
| `error` | String? | The machine-readable error code from the server on failure, null on success |
| `statusCode` | int | The raw HTTP status code |

---

## 7. Flutter — Auth State Management

Authentication state is owned by a single `AuthProvider` class. No screen or widget may maintain its own parallel auth state.

| Aspect | Detail |
|---|---|
| Location | `lib/providers/auth_provider.dart` |
| Creation timeline | Must be created as part of ticket AUTH-01 |
| Pattern | `ChangeNotifier` — notifies listeners on any state change |
| Registration | Registered at the root of the widget tree in `main.dart` via `MultiProvider` |

The `AuthProvider` exposes the following observable state fields:

| Field | Type | Description |
|---|---|---|
| `currentUser` | User? | The currently authenticated user, or null if not logged in |
| `isLoggedIn` | bool | True when `currentUser` is non-null and tokens are valid |
| `isLoading` | bool | True while an async auth operation is in progress |

The `AuthProvider` exposes the following methods:

| Method | Behavior |
|---|---|
| `login(email, password)` | Calls the backend, sets `currentUser` and `isLoggedIn`, persists tokens via `FlutterSecureStorage` |
| `register(...)` | Same post-success behavior as `login` |
| `logout()` | Clears `currentUser`, sets `isLoggedIn = false`, deletes tokens from `FlutterSecureStorage`, calls `POST /api/v1/auth/logout` as fire-and-forget (does not await or surface errors) |
| `restoreSession()` | Called once on app start. Reads stored tokens, validates or refreshes them, and re-hydrates `currentUser` if a valid session exists |

Screens access auth state using `context.read<AuthProvider>()` for one-off reads and `context.watch<AuthProvider>()` when the widget must rebuild on state changes.

The existing `LocalAuthService` with SharedPreferences is the legacy demo-only implementation. It must be replaced entirely by `AuthProvider` and `ApiClient` as part of AUTH-01. No new code should reference `LocalAuthService`.

---

## 8. Flutter — Token Storage

Tokens are sensitive credentials and must be stored using `flutter_secure_storage`. They must never be written to `SharedPreferences`, in-memory caches that persist across sessions, or any other unencrypted store.

| Aspect | Detail |
|---|---|
| Package | `flutter_secure_storage ^9.2.2` |
| Key for access token | `access_token` |
| Key for refresh token | `refresh_token` |

`SharedPreferences` remains available in the app but is restricted to non-sensitive user preferences such as whether the onboarding flow has been completed or the user's selected theme. If there is any doubt about whether a value is sensitive, it must go into `FlutterSecureStorage`.

---

## 9. Flutter — Navigation Conventions

Named routes are the canonical navigation mechanism in the app and are all defined in `main.dart`. The current named routes are `/`, `/login`, `/home`, and `/admin`.

| Scenario | Navigation Call |
|---|---|
| After successful login or registration | `Navigator.of(context).pushNamedAndRemoveUntil('/home', (_) => false)` — the entire back stack is cleared so the user cannot navigate back to the auth flow |
| After logout | `Navigator.of(context).pushNamedAndRemoveUntil('/login', (_) => false)` — the entire back stack is cleared |
| Opening the login screen from within the app | `Navigator.push(...)` using a non-named route, so the user can pop back to wherever they came from |

The `/` route renders the home experience. Unauthenticated users may browse a limited view; full features such as booking and tipping require being logged in. The `/` route must never itself require authentication — any gating is handled at the feature level within the screen.

---

## 10. Flutter — Error Display Convention

User-facing feedback from API calls and other async operations is displayed using `ScaffoldMessenger` SnackBars. Dialogs, custom overlays, or other patterns are not used for routine success and error feedback.

The color coding for SnackBars is as follows:

| Severity | Background Color |
|---|---|
| Success | `Colors.green` |
| Error | `Colors.red` |
| Warning or informational | `Colors.orange` |

While an async action is in progress, the button or control that triggered it must display a `CircularProgressIndicator` sized at 20 by 20 logical pixels with a `strokeWidth` of 2, and the control must be disabled to prevent duplicate submissions. The `isLoading` flag that drives this state is always set back to false in a `finally`-equivalent block, regardless of whether the action succeeded or failed. Loading state must never be left active after an action completes.

---

## 11. Demo Data Seeding

Seed data provides pre-built accounts for development and testing. It must never be applied to the production database.

### What it is

A SQL seed file that inserts two fixed user accounts into the `users` table. These accounts have known credentials so developers can log in immediately without going through the registration flow during local development.

### Location and format

| Item | Detail |
|---|---|
| File location | `staff-search-api/seeds/demo_data.sql` |
| Format | Plain SQL INSERT statements with `ON CONFLICT DO NOTHING` so the file is safe to run multiple times |
| When to run | Only on `development` and `staging` environments, never on production |
| Run command | `psql $DATABASE_URL -f seeds/demo_data.sql` |

### Demo accounts

These are the two fixed accounts. All other fields not listed below use their column defaults.

**Regular user demo account:**

| Field | Value |
|---|---|
| `id` | `demo_user_00000000000000001` (fixed ULID-length string for predictability) |
| `email` | `demo@example.com` |
| `password_hash` | bcrypt hash of `demo123` at cost factor 12 |
| `name` | `Demo User` |
| `role` | `user` |
| `points` | `1000` |
| `is_active` | `true` |
| `privacy_policy_accepted` | `true` |
| `privacy_policy_version` | `v1.0` |
| `is_staff_registered` | `false` |

**Staff demo account:**

| Field | Value |
|---|---|
| `id` | `demo_staff_0000000000000001` (fixed ULID-length string) |
| `email` | `staff-demo@example.com` |
| `password_hash` | bcrypt hash of `demo123` at cost factor 12 |
| `name` | `Demo Staff` |
| `role` | `staff` |
| `points` | `5000` |
| `is_active` | `true` |
| `privacy_policy_accepted` | `true` |
| `privacy_policy_version` | `v1.0` |
| `is_staff_registered` | `true` |

### Flutter behaviour for demo accounts

The login screen pre-fills the email and password fields when the user taps the demo account button. It does **not** call any special endpoint — it simply populates the form fields and submits the regular `POST /api/v1/auth/login` call. The backend treats demo accounts identically to any other account.
