# [AUTH-02] Google Sign-In — Implementation Plan

**Spec:** `docs/screens/AUTH-02-google-signin.md`
**Priority:** P1 MVP
**Phụ thuộc:** AUTH-01 phải hoàn thành trước (`ApiClient`, `AuthProvider` đã có)

---

## Backend Steps

### Step 1: Thêm dependency Google idtoken

**File:** `go.mod` / `go.sum`

```
~/go/bin/go1.26.1 get google.golang.org/api/idtoken
~/go/bin/go1.26.1 mod tidy
```

### Step 2: Thêm GOOGLE_CLIENT_ID vào config

**File:** `internal/config/config.go`

Thêm field: `GoogleClientID string` — load từ env var `GOOGLE_CLIENT_ID`.

**File:** `.env.example`

Thêm: `GOOGLE_CLIENT_ID=your-web-client-id.apps.googleusercontent.com`

### Step 3: Tạo migration thêm OAuth fields vào users

**File:** `migrations/20260310000004_add_oauth_fields_to_users.up.sql`

Thêm các cột vào `users` table (chỉ cần thêm cột chưa có — check model hiện tại):
```sql
ALTER TABLE users ADD COLUMN IF NOT EXISTS google_id VARCHAR(255) UNIQUE;
ALTER TABLE users ADD COLUMN IF NOT EXISTS apple_id VARCHAR(255) UNIQUE;
ALTER TABLE users ADD COLUMN IF NOT EXISTS is_verified BOOLEAN NOT NULL DEFAULT FALSE;
ALTER TABLE users ADD COLUMN IF NOT EXISTS auth_provider VARCHAR(50) NOT NULL DEFAULT 'email';
ALTER TABLE users ADD COLUMN IF NOT EXISTS status VARCHAR(20) NOT NULL DEFAULT 'active';

CREATE INDEX IF NOT EXISTS idx_users_google_id ON users (google_id) WHERE google_id IS NOT NULL;
```

> Lưu ý: Kiểm tra `internal/model/user.go` — nhiều fields này đã có sẵn. Chỉ thêm những gì còn thiếu.

**File:** `migrations/20260310000004_add_oauth_fields_to_users.down.sql`

```sql
ALTER TABLE users DROP COLUMN IF EXISTS google_id;
-- etc.
```

### Step 4: Tạo GoogleUserInfo struct và VerifyGoogleToken method

**File:** `internal/service/auth_service.go` (thêm vào)

Struct `GoogleUserInfo`:
- `GoogleID` string (từ `sub` claim)
- `Email` string
- `Name` string
- `AvatarURL` string
- `Verified` bool

Method `VerifyGoogleToken(ctx, idToken string) (*GoogleUserInfo, error)`:
1. Gọi `idtoken.Validate(ctx, idToken, cfg.GoogleClientID)` từ `google.golang.org/api/idtoken`
2. Nếu lỗi (expired, wrong aud, tampered) → return `nil, model.ErrInvalidToken`
3. Extract từ payload:
   - `payload.Subject` → `GoogleID`
   - `payload.Claims["email"].(string)` → `Email`
   - `payload.Claims["name"].(string)` → `Name`
   - `payload.Claims["picture"].(string)` → `AvatarURL`
   - `payload.Claims["email_verified"].(bool)` → `Verified`
4. Return `&GoogleUserInfo{...}, nil`

### Step 5: Thêm UpsertGoogleUser vào UserService

**File:** `internal/service/user_service.go` (thêm method)

Method `UpsertGoogleUser(ctx, info *GoogleUserInfo) (*User, bool, error)` — trả về (user, isNew, err):

Logic theo thứ tự:
1. **FindByGoogleID:** gọi `userRepo.FindByGoogleID(ctx, info.GoogleID)`
   - Nếu tìm thấy: gọi `userRepo.UpdateLastLogin(ctx, user.ID)`, return `(user, false, nil)`
2. **FindByEmail:** gọi `userRepo.FindByEmail(ctx, info.Email)`
   - Nếu tìm thấy: gọi `userRepo.LinkGoogleProvider(ctx, user.ID, info.GoogleID, &info.AvatarURL)`, gọi `UpdateLastLogin`, return `(user, false, nil)`
3. **Create new user:**
   - Build `&model.User{ID: ulid.New(), Email: info.Email, Name: info.Name, AvatarURL: &info.AvatarURL, GoogleID: &info.GoogleID, AuthProvider: "google", Role: "user", Status: "active", IsVerified: info.Verified, Points: 0, ...}`
   - Gọi `userRepo.Create(ctx, user)`
   - Return `(user, true, nil)`

### Step 6: Thêm methods vào UserRepository

**File:** `internal/repository/user_repository.go` (thêm methods)

Method `FindByGoogleID(ctx, googleID string) (*model.User, error)`:
```sql
SELECT * FROM users WHERE google_id = $1 LIMIT 1
```
Trả `ErrNotFound` nếu không có record.

Method `LinkGoogleProvider(ctx, userID, googleID string, avatarURL *string) error`:
```sql
UPDATE users SET
  google_id = $2,
  avatar_url = COALESCE($3, avatar_url),
  auth_provider = CASE WHEN auth_provider = 'email' THEN 'google' ELSE 'multiple' END,
  updated_at = NOW()
WHERE id = $1
```

### Step 7: Thêm GoogleSignIn handler

**File:** `internal/handler/auth_handler.go` (thêm method)

Method `GoogleSignIn(c fiber.Ctx) error`:
1. Parse body `{ id_token: string }`. Return `400 bad_request` nếu parsing fail hoặc `id_token` trống.
2. Gọi `authService.VerifyGoogleToken(ctx, idToken)`. Return `400 invalid_token` nếu lỗi.
3. Gọi `userService.UpsertGoogleUser(ctx, googleInfo)`. Return `500` nếu DB error.
4. Check `user.Status`. Return `403 account_disabled` nếu "disabled".
5. Gọi `jwtService.GenerateTokenPair(user.ID, user.Email, user.Role)`.
6. Store refresh token hash trong `refresh_tokens` (như AUTH-01 login flow).
7. Return `201` nếu `isNew`, `200` nếu không; body giống AUTH-01 `AuthResponse`.

### Step 8: Đăng ký route

**File:** `router/router.go`

Thêm vào auth group (public, không cần JWT, có rate limiter):
```go
auth.Post("/google", authHandler.GoogleSignIn)
```

---

## Flutter Steps

### Step 1: Thêm package google_sign_in

**File:** `pubspec.yaml`

Thêm:
```yaml
google_sign_in: ^6.2.1
```

Chạy `flutter pub get`.

### Step 2: Thêm Google logo asset

**File:** `assets/icons/google_logo.svg`

Download official Google logo SVG từ Google Branding Guidelines. Đặt tại path trên.

**File:** `pubspec.yaml` — khai báo asset:
```yaml
flutter:
  assets:
    - assets/icons/google_logo.svg
```

Cần thêm package `flutter_svg: ^2.0.0` để render SVG.

### Step 3: Platform Configuration

**Android:**
1. Tải `google-services.json` từ Google Cloud Console (cần đăng ký package name + SHA-1 fingerprint)
2. Đặt tại `android/app/google-services.json`
3. File phải có entry Web Client (type 3) để `idToken` không bị null

**iOS:**
1. Tải `GoogleService-Info.plist` từ Google Cloud Console
2. Đặt tại `ios/Runner/GoogleService-Info.plist`
3. **File:** `ios/Runner/Info.plist` — thêm `CFBundleURLSchemes` với giá trị là REVERSED_CLIENT_ID (lấy từ GoogleService-Info.plist)

**Web:**
1. **File:** `web/index.html` — thêm trong `<head>`:
   ```html
   <meta name="google-signin-client_id" content="YOUR_WEB_CLIENT_ID.apps.googleusercontent.com">
   ```

### Step 4: Tạo GoogleSignInService

**File:** `lib/services/google_sign_in_service.dart`

Sealed result type:
```dart
sealed class GoogleSignInResult {}
class GoogleSignInSuccess extends GoogleSignInResult { final String idToken; }
class GoogleSignInCancelled extends GoogleSignInResult {}
class GoogleSignInError extends GoogleSignInResult { final String message; }
```

Class `GoogleSignInService`:
- Khởi tạo `GoogleSignIn` với scopes `['email', 'profile']` và với Web: `clientId: kGoogleWebClientId`
- Method `signIn()` → `Future<GoogleSignInResult>`:
  1. Gọi `_googleSignIn.signIn()`
  2. Nếu null (user cancel) → return `GoogleSignInCancelled()`
  3. Gọi `account.authentication` để lấy `idToken`
  4. Nếu `idToken == null` → return `GoogleSignInError("Google authentication failed. Please try again.")`
  5. Return `GoogleSignInSuccess(idToken: idToken)`
- Method `signOut()` → gọi `_googleSignIn.signOut()`

### Step 5: Thêm loginWithGoogle() vào AuthProvider

**File:** `lib/providers/auth_provider.dart` (thêm method)

Method `loginWithGoogle()` → `Future<void>`:
1. Gọi `GoogleSignInService.signIn()`
2. Nếu `GoogleSignInCancelled` → return (no state change)
3. Nếu `GoogleSignInError` → set `errorMessage`, notifyListeners(), return
4. Set `isLoading = true`, `errorMessage = null`, notifyListeners()
5. Gọi `ApiClient.post('/api/v1/auth/google', {id_token: idToken}, requireAuth: false)`
6. Nếu 200/201:
   - Lưu `access_token` + `refresh_token` vào `FlutterSecureStorage`
   - Set `currentUser`, `isLoggedIn = true`
7. Nếu 400 hoặc 403:
   - Gọi `GoogleSignInService.signOut()` để clear Google session
   - Set `errorMessage`:
     - 400 → "Google authentication failed. Please try again."
     - 403 → "This account has been suspended."
8. Nếu network error → "Unable to connect. Check your network and retry."
9. Set `isLoading = false`, notifyListeners()

### Step 6: Thêm Google button vào LoginScreen

**File:** `lib/screens/login_screen.dart`

Thêm Google sign-in button giữa OR divider và registration button:

Specs của button:
- Height: 56px, full width
- Background: `#FFFFFF`, border: 1px solid `#DADCE0`, border radius: 12px
- Icon: `google_logo.svg` 24px, left-aligned, 16px padding
- Label: "Sign in with Google", font 16px, màu `#3C4043`
- Pressed: background `#F8F9FA`
- Loading: `CircularProgressIndicator` (20×20, strokeWidth 2) thay label; disabled

Method `_handleGoogleSignIn()`:
1. Gọi `context.read<AuthProvider>().loginWithGoogle()`
2. Sau khi complete, check `authProvider.isLoggedIn` → navigate to `/home`
3. Hoặc check `authProvider.errorMessage` → show red SnackBar

Button disabled khi `authProvider.isLoading == true` (block tất cả buttons trên screen trong khi loading).

### Step 7: Cập nhật User model

**File:** `lib/models/user.dart`

Thêm fields:
- `authProvider` String (email | google | apple | multiple)
- `isVerified` bool

Cập nhật `fromJson()` và `toJson()`.

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `go.mod` / `go.sum` | MODIFY — thêm google.golang.org/api |
| `internal/config/config.go` | MODIFY — thêm GoogleClientID |
| `.env.example` | MODIFY — thêm GOOGLE_CLIENT_ID |
| `migrations/20260310000004_add_oauth_fields_to_users.up.sql` | CREATE |
| `migrations/20260310000004_add_oauth_fields_to_users.down.sql` | CREATE |
| `internal/service/auth_service.go` | MODIFY — thêm VerifyGoogleToken |
| `internal/service/user_service.go` | MODIFY — thêm UpsertGoogleUser |
| `internal/repository/user_repository.go` | MODIFY — thêm FindByGoogleID, LinkGoogleProvider |
| `internal/handler/auth_handler.go` | MODIFY — thêm GoogleSignIn handler |
| `router/router.go` | MODIFY — thêm POST /api/v1/auth/google |

### Flutter
| File | Action |
|---|---|
| `pubspec.yaml` | MODIFY — thêm google_sign_in, flutter_svg |
| `assets/icons/google_logo.svg` | CREATE — official Google logo |
| `lib/services/google_sign_in_service.dart` | CREATE |
| `lib/providers/auth_provider.dart` | MODIFY — thêm loginWithGoogle() |
| `lib/models/user.dart` | MODIFY — thêm authProvider, isVerified |
| `lib/screens/login_screen.dart` | MODIFY — thêm Google button |
| `android/app/google-services.json` | REPLACE — download từ Google Console |
| `ios/Runner/GoogleService-Info.plist` | REPLACE — download từ Google Console |
| `ios/Runner/Info.plist` | MODIFY — thêm CFBundleURLSchemes |
| `web/index.html` | MODIFY — thêm google-signin-client_id meta tag |

---

## Verification Checklist

- [ ] `POST /api/v1/auth/google` với valid idToken (new user) → 201, user created, `auth_provider = 'google'`
- [ ] `POST /api/v1/auth/google` với valid idToken (email đã tồn tại) → 200, `google_id` linked, no duplicate
- [ ] `POST /api/v1/auth/google` với expired/tampered token → 400 `invalid_token`
- [ ] `POST /api/v1/auth/google` với account disabled → 403 `account_disabled`
- [ ] Flutter Android: Google button flow hoàn chỉnh trên real device
- [ ] Flutter iOS: Google button flow hoàn chỉnh trên real device
- [ ] Flutter Web: Google popup flow hoàn chỉnh trong Chrome
- [ ] User cancel account picker → Login screen không thay đổi, không có toast
- [ ] Tokens được lưu trong FlutterSecureStorage sau khi Google sign-in
- [ ] New user có `privacy_policy_accepted = false` → Privacy Policy modal xuất hiện trước home screen
- [ ] Google session được clear khi account bị suspended (403)
- [ ] Google logo đúng branding (không bị resize/recolor)
