# [AUTH-01] Email / Password Authentication — Implementation Plan

**Spec:** `docs/screens/AUTH-01-email-password.md`
**Priority:** P1 MVP
**Phụ thuộc:** Không có (backend đã hoàn thành)

---

## Trạng thái hiện tại

**Backend — ĐÃ XONG:**
- `POST /api/v1/auth/login` ✅
- `POST /api/v1/auth/register` ✅
- `POST /api/v1/auth/refresh` ✅
- `POST /api/v1/auth/logout` ✅
- `GET /api/v1/auth/me` ✅
- `POST /api/v1/auth/privacy-policy/accept` ✅
- Model: `internal/model/user.go` — User, RefreshToken ✅
- Migration: `migrations/20260310000001_create_users_table.up.sql` ✅

**Backend — CÒN THIẾU:**
- `POST /api/v1/auth/password-reset/request`
- `POST /api/v1/auth/password-reset/confirm`
- `password_reset_tokens` table
- SMTP email sending

**Flutter — CHƯA LÀM:**
- `AuthProvider` (ChangeNotifier)
- `ApiClient` (HTTP wrapper với token refresh)
- Migration login/register screens từ `LocalAuthService` → `AuthProvider`
- Token storage trong `FlutterSecureStorage`

---

## Backend Steps

### Step 1: Tạo password_reset_tokens table

**File:** `migrations/20260310000003_create_password_reset_tokens.up.sql`

Tạo table với các cột:
- `id` VARCHAR(26) PK (ULID)
- `user_id` VARCHAR(26) NOT NULL → FK users.id ON DELETE CASCADE
- `token_hash` VARCHAR(64) UNIQUE NOT NULL (SHA-256 của plaintext token)
- `expires_at` TIMESTAMPTZ NOT NULL
- `used_at` TIMESTAMPTZ (null = chưa dùng)
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT NOW()

Index: `idx_password_reset_tokens_token_hash` trên `token_hash`
Index: `idx_password_reset_tokens_user_id` trên `user_id`

**File:** `migrations/20260310000003_create_password_reset_tokens.down.sql`
→ DROP TABLE password_reset_tokens

### Step 2: Tạo GORM model cho PasswordResetToken

**File:** `internal/model/user.go` (thêm vào file hiện có)

Thêm struct `PasswordResetToken`:
- ID string (ULID)
- UserID string (FK)
- User User (association)
- TokenHash string
- ExpiresAt time.Time
- UsedAt *time.Time (nullable)
- CreatedAt time.Time

Thêm error sentinel: `ErrTokenExpired`, `ErrTokenUsed`

### Step 3: Tạo PasswordResetToken repository

**File:** `internal/repository/password_reset_repository.go`

Methods:
- `Create(ctx, token *PasswordResetToken) error`
- `FindByTokenHash(ctx, hash string) (*PasswordResetToken, error)` — trả `ErrNotFound` nếu không có
- `MarkUsed(ctx, tokenID string) error` — set `used_at = NOW()`
- `DeleteExpired(ctx) error` — xóa tokens đã hết hạn (optional, chạy cleanup)

### Step 4: Tạo EmailService (SMTP)

**File:** `pkg/email/email.go`

Interface `EmailSender` với method:
- `SendPasswordReset(ctx, toEmail, resetLink string) error`

Implement với `net/smtp` hoặc package `github.com/wneessen/go-mail`.

Env vars cần dùng (thêm vào `internal/config/config.go`):
- `SMTP_HOST`
- `SMTP_PORT`
- `SMTP_USERNAME`
- `SMTP_PASSWORD`
- `SMTP_FROM`
- `APP_BASE_URL` (để build reset link)

### Step 5: Thêm 2 endpoints vào AuthService

**File:** `internal/service/auth_service.go`

Method `RequestPasswordReset(ctx, email string) error`:
1. Gọi `userRepo.FindByEmail(email)`. Nếu `ErrNotFound` → return nil (không lộ thông tin)
2. Generate reset token: `ulid.New()` (plaintext)
3. SHA-256 hash token → `tokenHash`
4. Tạo `PasswordResetToken` với `ExpiresAt = now + 1 hour`
5. Gọi `passwordResetRepo.Create(ctx, token)`
6. Build reset link: `cfg.AppBaseURL + "/reset-password?token=" + plaintextToken`
7. Gọi `emailService.SendPasswordReset(ctx, email, resetLink)`
8. Return nil

Method `ConfirmPasswordReset(ctx, plaintext, newPassword string) error`:
1. SHA-256 hash plaintext token
2. Gọi `passwordResetRepo.FindByTokenHash(ctx, hash)`
3. Nếu `ErrNotFound` → return `ErrInvalidToken`
4. Nếu `token.ExpiresAt.Before(now)` → return `ErrTokenExpired`
5. Nếu `token.UsedAt != nil` → return `ErrTokenUsed`
6. Hash newPassword với bcrypt cost 12
7. Update `users.password_hash` cho `token.UserID`
8. Gọi `passwordResetRepo.MarkUsed(ctx, token.ID)`
9. Return nil

### Step 6: Thêm 2 handler methods vào AuthHandler

**File:** `internal/handler/auth_handler.go`

Method `RequestPasswordReset(c fiber.Ctx) error`:
- Parse body `{ email: string }`
- Gọi `authService.RequestPasswordReset(ctx, email)`
- Luôn return `200 { message: "If that email is registered, you will receive a reset link shortly." }` (không phân biệt email có tồn tại hay không)

Method `ConfirmPasswordReset(c fiber.Ctx) error`:
- Parse body `{ token: string, new_password: string }`
- Validate `new_password` min 6 ký tự
- Gọi `authService.ConfirmPasswordReset(ctx, token, newPassword)`
- Map errors:
  - `ErrInvalidToken` → `400 invalid_token` "This reset link is invalid."
  - `ErrTokenExpired` → `400 invalid_token` "This reset link has expired. Please request a new one."
  - `ErrTokenUsed` → `400 invalid_token` "This reset link has already been used."
- Return `200 { message: "Password updated successfully." }`

### Step 7: Đăng ký routes mới

**File:** `router/router.go`

Thêm vào auth group (public, không cần JWT):
```
auth.Post("/password-reset/request", authHandler.RequestPasswordReset)
auth.Post("/password-reset/confirm", authHandler.ConfirmPasswordReset)
```

### Step 8: Update AutoMigrate trong main.go

**File:** `main.go`

Thêm `&model.PasswordResetToken{}` vào `db.AutoMigrate()` call.
Khởi tạo `passwordResetRepo`, `emailService`, update wire-up.

---

## Flutter Steps

### Step 1: Thêm packages vào pubspec.yaml

Thêm:
- `flutter_secure_storage: ^9.2.2`
- `http: ^1.2.0` (nếu chưa có)
- `provider: ^6.1.2` (nếu chưa có)

Chạy `flutter pub get`.

### Step 2: Tạo ApiResponse model

**File:** `lib/models/api_response.dart`

Generic class `ApiResponse<T>`:
- `data` T? (null khi error)
- `error` String? — machine code từ backend (bad_request, unauthorized, ...)
- `message` String? — human readable message từ backend
- `statusCode` int
- factory `ApiResponse.success(T data, int statusCode)`
- factory `ApiResponse.error(String error, String message, int statusCode)`

### Step 3: Tạo User model (Dart)

**File:** `lib/models/user.dart` (modify hoặc tạo mới nếu chưa có đủ fields)

Fields:
- `id` String
- `email` String
- `name` String
- `avatarUrl` String?
- `role` String (`user` | `staff` | `admin`)
- `isStaff` bool
- `isStaffRegistered` bool
- `points` int
- `privacyPolicyAccepted` bool
- `createdAt` DateTime

Factory `User.fromJson(Map<String, dynamic> json)`.
Method `toJson()`.

### Step 4: Tạo ApiClient

**File:** `lib/services/api_client.dart`

Singleton class `ApiClient`:

**Constructor:** nhận `baseUrl` (từ const config hoặc env).

**Headers mặc định:** `Content-Type: application/json`

**Method `_getHeaders({bool requireAuth = true})`:**
- Nếu `requireAuth`: đọc `access_token` từ `FlutterSecureStorage`, thêm `Authorization: Bearer <token>`
- Return Map<String, String>

**Method `post(String path, Map<String, dynamic> body, {bool requireAuth = true})`:**
- Gọi `http.post(Uri.parse(baseUrl + path), headers: ..., body: jsonEncode(body))`
- Nếu response 401 VÀ `requireAuth`:
  - Thử refresh token (gọi `_tryRefresh()`)
  - Nếu refresh thành công: retry request một lần
  - Nếu refresh thất bại: clear tokens, ném `UnauthorizedException`
- Parse response → `ApiResponse<Map<String, dynamic>>`

**Method `get(String path)`:**
- Tương tự `post` nhưng HTTP GET, không có body

**Method `patch(String path, Map<String, dynamic> body)`:**
- HTTP PATCH

**Method `delete(String path, {Map<String, dynamic>? body})`:**
- HTTP DELETE

**Private method `_tryRefresh()`:**
1. Đọc `refresh_token` từ `FlutterSecureStorage`
2. Gọi `POST /api/v1/auth/refresh` với `{ refresh_token: ... }` (không cần auth header)
3. Nếu 200: lưu tokens mới vào `FlutterSecureStorage`, return true
4. Nếu thất bại: return false

**Constant:** `lib/config/api_config.dart` với `kApiBaseUrl = 'http://localhost:3000'` (dev) — cần environment config.

### Step 5: Tạo AuthProvider

**File:** `lib/providers/auth_provider.dart`

Class `AuthProvider extends ChangeNotifier`:

**State:**
- `User? currentUser`
- `bool isLoggedIn`
- `bool isLoading`
- `String? errorMessage`

**FlutterSecureStorage:** instance private `_storage = FlutterSecureStorage()`

**Method `restoreSession()` → Future<void>:**
1. Đọc `access_token` từ `_storage`
2. Nếu null: set `isLoggedIn = false`, notifyListeners(), return
3. Gọi `ApiClient.get('/api/v1/auth/me')`
4. Nếu 200: parse user, set `currentUser`, `isLoggedIn = true`
5. Nếu 401: xóa cả 2 tokens khỏi storage, set `isLoggedIn = false`
6. notifyListeners()

**Method `login(String email, String password)` → Future<void>:**
1. Set `isLoading = true`, `errorMessage = null`, notifyListeners()
2. Gọi `ApiClient.post('/api/v1/auth/login', {email, password}, requireAuth: false)`
3. Nếu 200: lưu `access_token` + `refresh_token` vào `_storage`, parse `user`, set `currentUser`, `isLoggedIn = true`
4. Nếu lỗi: set `errorMessage` theo status code (401 → "Invalid email or password.", 403 → "Your account has been disabled. Contact support.", 429 → "Too many attempts. Please wait before trying again.", khác → "Something went wrong. Please try again later.")
5. Set `isLoading = false`, notifyListeners()

**Method `register(String name, String email, String? phone, String password, String role, bool privacyPolicyAccepted)` → Future<void>:**
1. Set `isLoading = true`, `errorMessage = null`, notifyListeners()
2. Gọi `ApiClient.post('/api/v1/auth/register', {name, email, phone, password, role, privacy_policy_accepted}, requireAuth: false)`
3. Nếu 201: lưu tokens, set user, `isLoggedIn = true`
4. Nếu 409: `errorMessage = "An account with this email already exists."`
5. Nếu lỗi khác: map tương tự login
6. Set `isLoading = false`, notifyListeners()

**Method `logout()` → Future<void>:**
1. Gọi `ApiClient.post('/api/v1/auth/logout', {})`
2. Xóa `access_token` + `refresh_token` khỏi `_storage`
3. Set `currentUser = null`, `isLoggedIn = false`, notifyListeners()

**Method `updateCurrentUser(User user)`:**
- Set `currentUser = user`, notifyListeners()

### Step 6: Đăng ký AuthProvider trong main.dart

**File:** `lib/main.dart`

Wrap `MaterialApp` với `MultiProvider`:
```dart
MultiProvider(
  providers: [
    ChangeNotifierProvider(create: (_) => AuthProvider()),
    // ... providers khác sau này
  ],
  child: MaterialApp(...)
)
```

Trong `initState` của root widget hoặc `main()`: gọi `AuthProvider.restoreSession()` để auto-login.

Thêm route guard: nếu `authProvider.isLoggedIn == false` → redirect về `/login`.

### Step 7: Migrate LoginScreen

**File:** `lib/screens/login_screen.dart`

Thay đổi:
1. Xóa toàn bộ import và usage của `LocalAuthService`
2. Thay `LocalAuthService.login()` bằng `context.read<AuthProvider>().login(email, password)`
3. Dùng `Consumer<AuthProvider>` hoặc `context.watch<AuthProvider>()` để:
   - Disable Login button khi `authProvider.isLoading == true`
   - Show `CircularProgressIndicator` (20×20, strokeWidth 2) bên trong button khi loading
   - Hiển thị `authProvider.errorMessage` dưới nút Login (màu đỏ)
4. Sau khi login thành công (`authProvider.isLoggedIn == true`): `Navigator.pushNamedAndRemoveUntil('/home', (_) => false)`
5. Handler "Forgot Password?": gọi `_handlePasswordReset()`:
   - Show dialog nhập email
   - Gọi `ApiClient.post('/api/v1/auth/password-reset/request', {email})`
   - Luôn show SnackBar: "If that email is registered, you will receive a reset link shortly."
6. Giữ nguyên demo pre-fill buttons (chỉ pre-fill fields, không gọi LocalAuthService)

### Step 8: Migrate RegisterScreen

**File:** `lib/screens/register_screen.dart`

Thay đổi:
1. Xóa import `LocalAuthService`
2. Thay `LocalAuthService.register()` bằng `context.read<AuthProvider>().register(...)`
3. Client-side validation trước khi gọi API:
   - Name: required, không trống
   - Email: required, format hợp lệ
   - Password: min 6 ký tự
   - Confirm password: phải match → "Passwords do not match."
   - Privacy Policy checkbox: phải tick
4. Loading state trên Save button như LoginScreen
5. Show `authProvider.errorMessage` sau khi submit
6. Sau khi register thành công: `Navigator.pushNamedAndRemoveUntil('/home', (_) => false)`

### Step 9: Tạo PasswordResetConfirmScreen (nếu dùng deep link)

**File:** `lib/screens/password_reset_confirm_screen.dart`

Screen nhận `token` từ query param (deep link `/reset-password?token=...`):
1. Form nhập `newPassword` + `confirmPassword`
2. Validate: min 6 ký tự, match
3. Gọi `ApiClient.post('/api/v1/auth/password-reset/confirm', {token, new_password})`
4. Nếu 200: show SnackBar "Password updated successfully." → navigate to `/login`
5. Error cases per spec Section 2.3

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `migrations/20260310000003_create_password_reset_tokens.up.sql` | CREATE |
| `migrations/20260310000003_create_password_reset_tokens.down.sql` | CREATE |
| `internal/model/user.go` | MODIFY — thêm PasswordResetToken struct + error sentinels |
| `internal/repository/password_reset_repository.go` | CREATE |
| `pkg/email/email.go` | CREATE |
| `internal/service/auth_service.go` | MODIFY — thêm RequestPasswordReset, ConfirmPasswordReset |
| `internal/handler/auth_handler.go` | MODIFY — thêm 2 handlers |
| `router/router.go` | MODIFY — thêm 2 routes |
| `internal/config/config.go` | MODIFY — thêm SMTP + APP_BASE_URL fields |
| `main.go` | MODIFY — thêm PasswordResetToken vào AutoMigrate, wire deps |
| `.env.example` | MODIFY — thêm SMTP_*, APP_BASE_URL |

### Flutter
| File | Action |
|---|---|
| `pubspec.yaml` | MODIFY — thêm packages |
| `lib/config/api_config.dart` | CREATE |
| `lib/models/api_response.dart` | CREATE |
| `lib/models/user.dart` | MODIFY — đảm bảo có đủ fields từ backend |
| `lib/services/api_client.dart` | CREATE |
| `lib/providers/auth_provider.dart` | CREATE |
| `lib/main.dart` | MODIFY — MultiProvider, AuthProvider, route guard |
| `lib/screens/login_screen.dart` | MODIFY — remove LocalAuthService, dùng AuthProvider |
| `lib/screens/register_screen.dart` | MODIFY — remove LocalAuthService, dùng AuthProvider |
| `lib/screens/password_reset_confirm_screen.dart` | CREATE |

---

## Verification Checklist

- [ ] `POST /api/v1/auth/login` với đúng credentials → 200 + JWT pair
- [ ] `POST /api/v1/auth/login` với sai password → 401
- [ ] `POST /api/v1/auth/register` với email mới → 201 + user object
- [ ] `POST /api/v1/auth/register` với email đã có → 409
- [ ] `POST /api/v1/auth/refresh` với valid refresh_token → 200 + tokens mới
- [ ] `GET /api/v1/auth/me` với valid JWT → 200 + user
- [ ] `POST /api/v1/auth/password-reset/request` → 200 (bất kể email có tồn tại hay không)
- [ ] `POST /api/v1/auth/password-reset/confirm` với token hợp lệ → 200
- [ ] `POST /api/v1/auth/password-reset/confirm` với token hết hạn → 400 invalid_token
- [ ] Flutter: Login thành công → navigate to /home, `AuthProvider.isLoggedIn = true`
- [ ] Flutter: Login sai password → hiển thị "Invalid email or password."
- [ ] Flutter: Tokens lưu trong FlutterSecureStorage (không phải SharedPreferences)
- [ ] Flutter: App restart → `restoreSession()` tự động login nếu có token hợp lệ
- [ ] Flutter: Access token hết hạn → `ApiClient` auto-refresh, retry request
- [ ] Flutter: Refresh token hết hạn → clear session, navigate to /login
- [ ] Flutter: Register thành công → navigate to /home
- [ ] Flutter: Register với email đã tồn tại → "An account with this email already exists."
