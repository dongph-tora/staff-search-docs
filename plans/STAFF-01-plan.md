# [STAFF-01] Staff Profile CRUD — Implementation Plan

**Spec:** `docs/screens/STAFF-01-staff-profile-crud.md`
**Priority:** P1 MVP
**Phụ thuộc:**
- DB-01: `staff_profiles`, `staff_portfolio_photos` tables định nghĩa
- STAFF-02: `StaffNumberService` đã implement
- STAFF-03: `IsValidJobCategory()` helper và job categories endpoint đã có
- AUTH-01: `ApiClient`, `AuthProvider` đã có

---

## Backend Steps

### Step 1: Tạo StaffProfile GORM model

**File:** `internal/model/staff.go` (đã được tạo ở DB-01, verify và bổ sung)

Đảm bảo `StaffProfile` struct có đầy đủ:
```go
type StaffProfile struct {
  ID                string               `gorm:"primaryKey;type:varchar(26)"`
  UserID            string               `gorm:"uniqueIndex;type:varchar(26);not null"`
  User              User                 `gorm:"foreignKey:UserID;constraint:OnDelete:CASCADE"`
  StaffNumber       string               `gorm:"uniqueIndex;type:varchar(6);not null"`
  JobTitle          string               `gorm:"type:varchar(100);not null"`
  JobCategory       string               `gorm:"type:varchar(50);not null"`
  Location          *string              `gorm:"type:varchar(255)"`
  Bio               *string              `gorm:"type:text"`
  IntroVideoURL     *string              `gorm:"type:text"`
  IsAvailable       bool                 `gorm:"not null;default:false"`
  AcceptBookings    bool                 `gorm:"not null;default:true"`
  Rating            float32              `gorm:"type:decimal(3,2);not null;default:0"`
  ReviewCount       int                  `gorm:"not null;default:0"`
  FollowersCount    int                  `gorm:"not null;default:0"`
  TotalTipsReceived int                  `gorm:"not null;default:0"`
  PortfolioPhotos   []StaffPortfolioPhoto `gorm:"foreignKey:StaffProfileID"`
  CreatedAt         time.Time
  UpdatedAt         time.Time
}
```

### Step 2: Thêm methods vào StaffRepository

**File:** `internal/repository/staff_repository.go`

Method `FindByUserID(ctx, userID string) (*model.StaffProfile, error)`:
```go
var profile model.StaffProfile
err := r.db.WithContext(ctx).
  Preload("PortfolioPhotos", func(db *gorm.DB) *gorm.DB {
    return db.Order("display_order ASC")
  }).
  Where("user_id = ?", userID).
  First(&profile).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
  return nil, model.ErrNotFound
}
return &profile, err
```

Method `Create(ctx, profile *model.StaffProfile) error`:
```go
return r.db.WithContext(ctx).Create(profile).Error
```

Method `Update(ctx, userID string, fields map[string]any) error`:
```go
return r.db.WithContext(ctx).
  Model(&model.StaffProfile{}).
  Where("user_id = ?", userID).
  Updates(fields).Error
```

Method `ExistsByUserID(ctx, userID string) (bool, error)`:
```go
var count int64
r.db.WithContext(ctx).Model(&model.StaffProfile{}).
  Where("user_id = ?", userID).Count(&count)
return count > 0, nil
```

### Step 3: Tạo StaffService

**File:** `internal/service/staff_service.go` (tạo mới)

Struct `StaffService`:
- `staffRepo *repository.StaffRepository`
- `staffNumberService *StaffNumberService`
- `userRepo *repository.UserRepository`

Method `CreateProfile(ctx, userID string, req dto.CreateStaffProfileRequest) (*model.StaffProfile, error)`:
1. Gọi `staffRepo.ExistsByUserID(ctx, userID)`. Nếu exists → return `nil, model.ErrConflict`
2. Gọi `staffNumberService.Generate(ctx)`. Nếu lỗi → return lỗi đó
3. Build `&model.StaffProfile{ID: ulid.New(), UserID: userID, StaffNumber: number, JobTitle: req.JobTitle, JobCategory: req.JobCategory, ...AcceptBookings: req.AcceptBookings nếu có else true, ...}`
4. **Transaction:**
   ```go
   return r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
     // a. Insert staff_profiles
     if err := tx.Create(profile).Error; err != nil { return err }
     // b. Update users: is_staff_registered = true, role = 'staff'
     return tx.Model(&model.User{}).
       Where("id = ?", userID).
       Updates(map[string]any{"is_staff_registered": true, "role": "staff"}).Error
   })
   ```
5. Return created profile

Method `UpdateProfile(ctx, userID string, req dto.UpdateStaffProfileRequest) (*model.StaffProfile, error)`:
1. Gọi `staffRepo.FindByUserID(ctx, userID)`. Return `ErrNotFound` nếu không có.
2. Build `fields map[string]any` chỉ với non-nil fields từ req
3. Gọi `staffRepo.Update(ctx, userID, fields)`
4. Gọi `staffRepo.FindByUserID(ctx, userID)` để lấy updated profile
5. Return updated profile

Method `GetByUserID(ctx, userID string) (*model.StaffProfile, error)`:
- Gọi `staffRepo.FindByUserID(ctx, userID)`. Return ErrNotFound nếu không có.

### Step 4: Tạo DTOs

**File:** `internal/dto/staff_dto.go`

`CreateStaffProfileRequest`:
- `JobTitle` string `json:"job_title"`
- `JobCategory` string `json:"job_category"`
- `Location` *string `json:"location"`
- `Bio` *string `json:"bio"`
- `AcceptBookings` *bool `json:"accept_bookings"` (optional, default true)

`UpdateStaffProfileRequest`:
- `JobTitle` *string `json:"job_title"`
- `JobCategory` *string `json:"job_category"`
- `Location` *string `json:"location"`
- `Bio` *string `json:"bio"`
- `AcceptBookings` *bool `json:"accept_bookings"`
- `IsAvailable` *bool `json:"is_available"`

`StaffProfileResponse`:
- id, user_id, staff_number, job_title, job_category, location, bio, intro_video_url, is_available, accept_bookings, rating, review_count, followers_count, total_tips_received, portfolio_photos (array of PortfolioPhotoResponse), created_at, updated_at

`PortfolioPhotoResponse`:
- id, staff_profile_id, photo_url, display_order, created_at

Helper function `ToStaffProfileResponse(profile *model.StaffProfile) StaffProfileResponse` để convert model → DTO.

### Step 5: Tạo StaffHandler

**File:** `internal/handler/staff_handler.go` (tạo mới hoặc thêm methods)

Method `CreateProfile(c fiber.Ctx) error`:
1. Read `userID` từ `c.Locals("userID").(string)`
2. Parse body vào `CreateStaffProfileRequest`. Return `400` nếu fail.
3. Validate `JobTitle` không trống, max 100 chars. Return `422` nếu fail.
4. Validate `JobCategory` qua `config.IsValidJobCategory()`. Return `422` nếu invalid.
5. Gọi `staffService.CreateProfile(ctx, userID, req)`.
6. Map: `ErrConflict` → `409` "You already have a staff profile.", `ErrStaffNumberExhausted` → `500`, khác → `500`.
7. Return `201` với `StaffProfileResponse`.

Method `UpdateProfile(c fiber.Ctx) error`:
1. Read `userID` từ JWT
2. Parse body. Return `400` nếu không có field nào (empty body).
3. Validate các fields có giá trị. Return `422` nếu invalid.
4. Gọi `staffService.UpdateProfile(ctx, userID, req)`.
5. Map: `ErrNotFound` → `404` "Staff profile not found.", khác → `500`.
6. Return `200` với `StaffProfileResponse`.

Method `GetMyProfile(c fiber.Ctx) error`:
1. Read `userID` từ JWT
2. Gọi `staffService.GetByUserID(ctx, userID)`.
3. `ErrNotFound` → `404`. Return `200` với `StaffProfileResponse`.

Method `GetProfile(c fiber.Ctx) error`:
1. Read `userID` từ `c.Params("userID")`
2. Gọi `staffService.GetByUserID(ctx, userID)`.
3. `ErrNotFound` → `404`. Return `200` với `StaffProfileResponse`.
4. **IMPORTANT:** Response không chứa `email` hay `password_hash` — chỉ dùng `StaffProfileResponse` DTO.

### Step 6: Đăng ký routes

**File:** `router/router.go`

Trong staff protected group:
```go
staff := protected.Group("/staff")
staff.Get("/job-categories", staffHandler.GetJobCategories)  // từ STAFF-03
staff.Post("/profile", staffHandler.CreateProfile)
staff.Patch("/profile", staffHandler.UpdateProfile)
staff.Get("/me", staffHandler.GetMyProfile)
staff.Get("/:userID", staffHandler.GetProfile)
```

### Step 7: Update AutoMigrate trong main.go

**File:** `main.go`

Đảm bảo `StaffProfile` và `StaffPortfolioPhoto` đã trong `db.AutoMigrate()` call (đã làm ở DB-01).

Wire StaffHandler vào router.

---

## Flutter Steps

### Step 1: Tạo StaffProfile model

**File:** `lib/models/staff_profile.dart`

```dart
class PortfolioPhoto {
  final String id;
  final String staffProfileId;
  final String photoUrl;
  final int displayOrder;
  final DateTime createdAt;

  factory PortfolioPhoto.fromJson(Map<String, dynamic> json) { ... }
}

class StaffProfile {
  final String id;
  final String userId;
  final String staffNumber;
  final String jobTitle;
  final String jobCategory;
  final String? location;
  final String? bio;
  final String? introVideoUrl;
  final bool isAvailable;
  final bool acceptBookings;
  final double rating;
  final int reviewCount;
  final int followersCount;
  final int totalTipsReceived;
  final List<PortfolioPhoto> portfolioPhotos;
  final DateTime createdAt;
  final DateTime updatedAt;

  factory StaffProfile.fromJson(Map<String, dynamic> json) { ... }
}
```

### Step 2: Tạo StaffService (methods cho CRUD)

**File:** `lib/services/staff_service.dart` (thêm methods vào file đã có từ STAFF-03)

Methods:
- `createProfile(Map<String, dynamic> req)` → `Future<StaffProfile>`:
  - Gọi `ApiClient.post('/api/v1/staff/profile', req)`
  - Nếu 201: return `StaffProfile.fromJson(response.data['data'])`
  - Nếu 409: throw `Exception("You already have a staff profile.")`

- `updateProfile(Map<String, dynamic> req)` → `Future<StaffProfile>`:
  - Gọi `ApiClient.patch('/api/v1/staff/profile', req)`
  - Nếu 200: return parsed StaffProfile

- `getProfile(String userID)` → `Future<StaffProfile?>`:
  - Gọi `ApiClient.get('/api/v1/staff/$userID')`
  - Nếu 200: return StaffProfile
  - Nếu 404: return null

- `getMyProfile()` → `Future<StaffProfile?>`:
  - Gọi `ApiClient.get('/api/v1/staff/me')`
  - Nếu 200: return StaffProfile
  - Nếu 404: return null

### Step 3: Tạo StaffProvider

**File:** `lib/providers/staff_provider.dart`

```dart
class StaffProvider extends ChangeNotifier {
  StaffProfile? currentProfile;
  bool isLoading = false;
  String? error;

  Future<void> loadMyProfile() async { ... }
  Future<void> createProfile(Map<String, dynamic> req) async { ... }
  Future<void> updateProfile(Map<String, dynamic> req) async { ... }
  void setProfile(StaffProfile profile) { currentProfile = profile; notifyListeners(); }
}
```

Trong mỗi async method:
1. Set `isLoading = true`, `error = null`, `notifyListeners()`
2. Gọi StaffService method
3. Update state
4. `finally: isLoading = false, notifyListeners()`

Trong `loadMyProfile()`: nếu `AuthProvider.currentUser?.isStaffRegistered == true` mới gọi API.

### Step 4: Đăng ký StaffProvider trong main.dart

**File:** `lib/main.dart`

Thêm vào `MultiProvider`:
```dart
ChangeNotifierProvider(create: (_) => StaffProvider()),
```

Trong `restoreSession()` flow: sau khi `AuthProvider` load user, nếu `user.isStaffRegistered == true` thì gọi `StaffProvider.loadMyProfile()`.

### Step 5: Tạo StaffRegistrationScreen

**File:** `lib/screens/staff/staff_registration_screen.dart` (tạo mới)

Form elements:
- `jobTitleField`: TextFormField, required, max 100 chars, validator: "Job title is required."
- `jobCategoryDropdown`: `JobCategoryDropdown` widget từ STAFF-03
- `locationField`: TextFormField, optional, max 255 chars
- `bioField`: TextFormField, optional, multiline max 5 rows, max 1000 chars
- `acceptBookingsSwitch`: Switch, default true

Save button handler:
1. `_formKey.currentState!.validate()` — nếu false: return
2. Show loading (CircularProgressIndicator 20×20, button disabled)
3. Gọi `context.read<StaffProvider>().createProfile({job_title, job_category, location, bio, accept_bookings})`
4. Nếu thành công:
   - Show green SnackBar: "Staff profile created successfully."
   - Navigate to staff home tab
5. Nếu lỗi: show red SnackBar với message từ exception
6. Finally: clear loading

### Step 6: Tạo StaffProfileEditScreen

**File:** `lib/screens/staff/staff_profile_edit_screen.dart` (tạo mới)

Giống StaffRegistrationScreen nhưng:
- Pre-fill tất cả fields từ `StaffProvider.currentProfile`
- Save button gọi `StaffProvider.updateProfile()` thay vì `createProfile()`
- On success: green SnackBar "Profile updated.", pop về profile screen

### Step 7: Modify StaffProfileScreen

**File:** `lib/screens/staff/staff_profile_screen.dart` (modify)

Thay đổi:
1. Nhận `userID` parameter
2. Trong `initState`: gọi `StaffService.instance.getProfile(userID)` và store result trong local state
3. Khi đang load: hiển thị `CircularProgressIndicator` fullscreen
4. Khi có lỗi 404: hiển thị message "Staff profile not found."
5. Khi có data: render toàn bộ từ `StaffProfile` object (không còn mock data)
6. Xóa tất cả hardcoded/mock data imports

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/model/staff.go` | VERIFY/MODIFY — đảm bảo StaffProfile struct đầy đủ |
| `internal/repository/staff_repository.go` | MODIFY — thêm FindByUserID, Create, Update, ExistsByUserID |
| `internal/service/staff_service.go` | CREATE — CreateProfile, UpdateProfile, GetByUserID |
| `internal/dto/staff_dto.go` | CREATE — request/response DTOs |
| `internal/handler/staff_handler.go` | CREATE/MODIFY — 4 handlers |
| `router/router.go` | MODIFY — đăng ký 4 staff routes |
| `main.go` | MODIFY — wire StaffHandler, StaffService |

### Flutter
| File | Action |
|---|---|
| `lib/models/staff_profile.dart` | CREATE — StaffProfile + PortfolioPhoto models |
| `lib/services/staff_service.dart` | CREATE/MODIFY — CRUD methods |
| `lib/providers/staff_provider.dart` | CREATE |
| `lib/screens/staff/staff_registration_screen.dart` | CREATE |
| `lib/screens/staff/staff_profile_edit_screen.dart` | CREATE |
| `lib/screens/staff/staff_profile_screen.dart` | MODIFY — replace mock data |
| `lib/main.dart` | MODIFY — register StaffProvider |

---

## Verification Checklist

- [ ] `POST /api/v1/staff/profile` → 201, `staff_number` là 6 digits, `users.is_staff_registered = true`, `users.role = 'staff'`
- [ ] `POST /api/v1/staff/profile` lần 2 → 409 conflict
- [ ] `PATCH /api/v1/staff/profile` với chỉ `bio` → 200, chỉ bio thay đổi
- [ ] `PATCH /api/v1/staff/profile` không có fields → 400
- [ ] `PATCH /api/v1/staff/profile` không có staff profile → 404
- [ ] `GET /api/v1/staff/me` → 200, full profile với portfolio_photos
- [ ] `GET /api/v1/staff/:userID` → 200, không có email/password_hash trong response
- [ ] `GET /api/v1/staff/:invalidID` → 404
- [ ] Tất cả endpoints không có JWT → 401
- [ ] Flutter: StaffRegistrationScreen validate required fields trước khi gọi API
- [ ] Flutter: Submit hợp lệ → 201, green SnackBar, navigate to staff home
- [ ] Flutter: StaffProfileScreen hiển thị live data, không có mock data
- [ ] Flutter: 404 trên StaffProfileScreen → "Staff profile not found."
- [ ] Transaction trong CreateProfile: nếu UPDATE users fail → staff_profiles row cũng được rollback
