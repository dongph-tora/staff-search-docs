# [DB-02] Migrate LocalStorage to Real DB — Implementation Plan

**Spec:** `docs/screens/DB-02-migrate-localstorage-to-db.md`
**Priority:** P1 MVP
**Phụ thuộc:** AUTH-01 phải hoàn thành trước (`ApiClient`, `AuthProvider`, `GET /api/v1/auth/me` đã có)

---

## Mục tiêu

1. Tạo `PATCH /api/v1/users/me` endpoint trên backend
2. Migrate `profile_edit_screen.dart` và `profile_settings_screen.dart` dùng `AuthProvider` thay cho `StorageHelper`/SharedPreferences
3. Refactor `AppModeService` để đọc role từ `AuthProvider` thay vì SharedPreferences

---

## Backend Steps

### Step 1: Tạo UpdateUser DTO

**File:** `internal/dto/user_dto.go` (tạo mới hoặc thêm vào file có sẵn)

Struct `UpdateUserRequest`:
- `Name` *string `json:"name"` — optional, nil nếu không gửi
- `Bio` *string `json:"bio"`
- `Phone` *string `json:"phone"`

Struct `UserResponse` (đã có trong `auth_dto.go`, move sang đây hoặc tái sử dụng):
- `id`, `email`, `name`, `bio` (*string), `phone` (*string), `avatar_url` (*string), `role`, `points`, `is_staff_registered`, `created_at`

### Step 2: Thêm Update method vào UserRepository

**File:** `internal/repository/user_repository.go` (thêm method)

Method `Update(ctx context.Context, userID string, fields map[string]any) error`:
```go
return r.db.WithContext(ctx).
  Model(&model.User{}).
  Where("id = ?", userID).
  Updates(fields).Error
```

Chỉ update các fields được truyền vào (không dùng `Save` để tránh zero-value override).

Method `FindByID` đã có — dùng để return updated user sau khi update.

### Step 3: Thêm UpdateProfile method vào UserService

**File:** `internal/service/user_service.go` (thêm method)

Method `UpdateProfile(ctx, userID string, req dto.UpdateUserRequest) (*model.User, error)`:

Validation:
1. Nếu `req.Name != nil`:
   - Nếu `*req.Name == ""` → return `model.ErrValidation` + message "Name cannot be empty."
   - Nếu `len(*req.Name) > 100` → return `model.ErrValidation` + message "Name must be 100 characters or fewer."
2. Nếu `req.Bio != nil && len(*req.Bio) > 500` → return `model.ErrValidation` + "Bio must be 500 characters or fewer."
3. Nếu `req.Phone != nil && *req.Phone != ""`:
   - Validate regex `^\+?[0-9]{7,15}$`
   - Nếu không match → return `model.ErrValidation` + "Enter a valid phone number."

Build `fields map[string]any`:
- Chỉ include fields không nil từ request
- Trim whitespace từ Name, Bio, Phone trước khi lưu
- Luôn thêm `"updated_at": time.Now()`

Gọi `userRepo.Update(ctx, userID, fields)`.
Gọi `userRepo.FindByID(ctx, userID)` để lấy updated user.
Return updated user.

### Step 4: Tạo UserHandler

**File:** `internal/handler/user_handler.go` (tạo mới)

Struct `UserHandler` với field `userService *service.UserService`.
Constructor `NewUserHandler(userService *service.UserService) *UserHandler`.

Method `UpdateProfile(c fiber.Ctx) error`:
1. Read `userID` từ `c.Locals("userID").(string)`
2. Parse body vào `dto.UpdateUserRequest`. Return `400 bad_request` nếu malformed JSON.
3. Gọi `userService.UpdateProfile(ctx, userID, req)`.
4. Map errors:
   - `model.ErrValidation` → `422 validation_error` với message từ error
5. Return `200` với `dto.UserResponse` của updated user.

### Step 5: Đăng ký route

**File:** `router/router.go`

Trong protected group (JWT required):
```go
users := protected.Group("/users")
users.Patch("/me", userHandler.UpdateProfile)
```

### Step 6: Update wire-up trong main.go

**File:** `main.go`

Tạo `userHandler := handler.NewUserHandler(userService)` và pass vào `router.Setup()`.
Cập nhật `router.Setup()` signature để nhận `userHandler`.

---

## Flutter Steps

### Step 1: Tạo UserService

**File:** `lib/services/user_service.dart` (tạo mới)

Class `UserService` singleton:

Method `updateProfile(Map<String, dynamic> fields)` → `Future<ApiResponse<User>>`:
1. Loại bỏ null values và empty strings khỏi `fields` map (chỉ gửi những gì user thực sự thay đổi)
2. Gọi `ApiClient.patch('/api/v1/users/me', fields)`
3. Parse response body thành `User` từ `response.data['data']` (hoặc response body trực tiếp tùy ApiClient convention)
4. Return `ApiResponse<User>`

### Step 2: Thêm updateCurrentUser vào AuthProvider

**File:** `lib/providers/auth_provider.dart` (thêm method)

Method `updateCurrentUser(User user)`:
```dart
currentUser = user;
notifyListeners();
```

### Step 3: Migrate ProfileEditScreen

**File:** `lib/screens/profile_edit_screen.dart`

Thay đổi:
1. **Xóa:** tất cả import và usage của `AppModeService.saveUserProfile()`, `StorageHelper.setString('user_profile', ...)`, `StorageHelper.getString(...)`
2. **Thêm:** `final authProvider = context.read<AuthProvider>();`
3. **Pre-fill fields** từ `authProvider.currentUser`:
   - `_nameController.text = authProvider.currentUser?.name ?? ''`
   - `_bioController.text = authProvider.currentUser?.bio ?? ''`
   - `_phoneController.text = authProvider.currentUser?.phone ?? ''`
4. **Validation inline:**
   - Name: nếu > 100 chars → `"Name must be 100 characters or fewer."`
   - Bio: nếu > 500 chars → `"Bio must be 500 characters or fewer."`
   - Phone: nếu không match pattern → `"Enter a valid phone number."`
5. **Save button handler:**
   ```
   a. Validate form → if fail: show inline errors, return
   b. Show loading (CircularProgressIndicator 20×20, button disabled)
   c. Gọi UserService.updateProfile({name: ..., bio: ..., phone: ...})
   d. Nếu success (200):
      - Gọi authProvider.updateCurrentUser(updatedUser)
      - Show green SnackBar: "Profile updated successfully."
      - Navigator.pop()
   e. Nếu lỗi: show red SnackBar với message tương ứng
   f. Finally: clear loading state
   ```
6. **Avatar photo area:** tap → show SnackBar "Profile photo upload coming soon." (chưa implement, sẽ làm ở AUTH-09/DB-05)

### Step 4: Migrate ProfileSettingsScreen

**File:** `lib/screens/profile_settings_screen.dart`

Thay đổi:
1. **Xóa:** tất cả `StorageHelper` reads cho user name, avatar URL, role
2. **Thay bằng:** `context.watch<AuthProvider>().currentUser` để đọc thông tin hiển thị
3. Layout không thay đổi — chỉ thay data source

### Step 5: Refactor AppModeService

**File:** `lib/services/app_mode_service.dart`

Thay đổi:
1. **Xóa:** tất cả SharedPreferences reads/writes cho:
   - `user_profile` key
   - `staff_profile` key
   - `user_logged_in` key
   - `staff_logged_in` key
   - `app_mode` key
2. **Giữ lại:** in-memory UI mode toggle nếu cần cho "switch to staff view" feature
3. **Thay:** role detection bằng cách inject `AuthProvider` và đọc `authProvider.currentUser?.role`
4. Nếu `AppModeService` là singleton, cần thêm method để set `AuthProvider` reference sau khi init

### Step 6: Thêm deprecation comment vào StorageHelper

**File:** `lib/utils/storage_helper.dart`

Thêm comment block ở đầu file:
```dart
// DEPRECATED: Do not use this class for new user-facing persistent data.
// Use AuthProvider for user state and ApiClient for API calls instead.
// This class is kept for backward compatibility during migration.
// See docs/screens/DB-02-migrate-localstorage-to-db.md
```

Không thay đổi logic.

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/dto/user_dto.go` | CREATE — UpdateUserRequest, UserResponse (hoặc move từ auth_dto.go) |
| `internal/repository/user_repository.go` | MODIFY — thêm Update(ctx, id, fields) method |
| `internal/service/user_service.go` | MODIFY — thêm UpdateProfile(ctx, userID, req) method |
| `internal/handler/user_handler.go` | CREATE — UpdateProfile handler |
| `router/router.go` | MODIFY — đăng ký PATCH /api/v1/users/me |
| `main.go` | MODIFY — wire UserHandler |

### Flutter
| File | Action |
|---|---|
| `lib/services/user_service.dart` | CREATE — wraps PATCH /api/v1/users/me |
| `lib/providers/auth_provider.dart` | MODIFY — thêm updateCurrentUser(User) |
| `lib/screens/profile_edit_screen.dart` | MODIFY — remove StorageHelper, dùng AuthProvider + UserService |
| `lib/screens/profile_settings_screen.dart` | MODIFY — remove StorageHelper, đọc từ AuthProvider |
| `lib/services/app_mode_service.dart` | MODIFY — remove SharedPreferences profile/login state |
| `lib/utils/storage_helper.dart` | MODIFY — thêm deprecation comment |

---

## Verification Checklist

- [ ] `PATCH /api/v1/users/me` với `{name: "New Name", bio: "Test"}` → 200, fields updated trong DB
- [ ] `PATCH /api/v1/users/me` với empty body `{}` → 200, user unchanged
- [ ] `PATCH /api/v1/users/me` với `name` > 100 chars → 422 validation_error
- [ ] `PATCH /api/v1/users/me` với `name: ""` → 422 validation_error
- [ ] `PATCH /api/v1/users/me` với `bio` > 500 chars → 422 validation_error
- [ ] `PATCH /api/v1/users/me` với `phone: "abc"` (invalid) → 422 validation_error
- [ ] `PATCH /api/v1/users/me` với `phone: ""` → 200, phone cleared
- [ ] `PATCH /api/v1/users/me` không có JWT → 401
- [ ] Flutter: ProfileEditScreen pre-fill từ AuthProvider, không từ SharedPreferences
- [ ] Flutter: Save thành công → green SnackBar "Profile updated successfully.", navigate back
- [ ] Flutter: Name > 100 chars → inline validation error, không gọi API
- [ ] Flutter: ProfileSettingsScreen hiển thị từ AuthProvider.currentUser
- [ ] Flutter: AppModeService không đọc SharedPreferences cho profile/login state
- [ ] Flutter: `role` field không thể thay đổi qua PATCH endpoint
