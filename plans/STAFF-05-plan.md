# [STAFF-05] Portfolio Photo Upload — Implementation Plan

**Spec:** `docs/screens/STAFF-05-portfolio-photo-upload.md`
**Priority:** P1 MVP
**Phụ thuộc:**
- STAFF-01: `StaffProfile` model, `StaffService`, `StaffProvider` đã có
- DB-05: `UploadService`, presigned URL API, `portfolio` folder (10MB, jpg/png/webp)

---

## Backend Steps

### Step 1: Thêm StaffPortfolioPhoto methods vào StaffRepository

**File:** `internal/repository/staff_repository.go` (thêm methods)

Method `FindProfileByUserID(ctx, userID string) (*model.StaffProfile, error)`:
- Chỉ cần profile ID, không cần preload photos (để tối ưu)
- Dùng `Select("id, user_id")` nếu chỉ cần id

Method `CountPortfolioPhotos(ctx, staffProfileID string) (int64, error)`:
```go
var count int64
r.db.WithContext(ctx).Model(&model.StaffPortfolioPhoto{}).
  Where("staff_profile_id = ?", staffProfileID).
  Count(&count)
return count, nil
```

Method `AddPortfolioPhoto(ctx, photo *model.StaffPortfolioPhoto) error`:
```go
return r.db.WithContext(ctx).Create(photo).Error
```

Method `GetPortfolioPhotoByID(ctx, photoID string) (*model.StaffPortfolioPhoto, error)`:
```go
var photo model.StaffPortfolioPhoto
err := r.db.WithContext(ctx).Where("id = ?", photoID).First(&photo).Error
// map ErrRecordNotFound → model.ErrNotFound
```

Method `DeletePortfolioPhoto(ctx, photoID string) error`:
```go
return r.db.WithContext(ctx).Where("id = ?", photoID).Delete(&model.StaffPortfolioPhoto{}).Error
```

Method `MaxDisplayOrder(ctx, staffProfileID string) (int, error)`:
```go
var maxOrder *int
r.db.WithContext(ctx).Model(&model.StaffPortfolioPhoto{}).
  Where("staff_profile_id = ?", staffProfileID).
  Select("MAX(display_order)").
  Scan(&maxOrder)
if maxOrder == nil { return 0, nil }
return *maxOrder, nil
```

Method `UpdateDisplayOrders(ctx, staffProfileID string, photoOrders []dto.PhotoOrder) error`:
- Chạy trong transaction: với mỗi `{id, order}` trong photoOrders, update `display_order` cho photo có `id = ?` và `staff_profile_id = ?`

### Step 2: Tạo StaffPortfolioService

**File:** `internal/service/staff_portfolio_service.go` (tạo mới)

Struct với `staffRepo *repository.StaffRepository`, `storageClient pkg_storage.StorageClient`.

Method `AddPhoto(ctx, userID string, req dto.AddPortfolioPhotoRequest) (*model.StaffPortfolioPhoto, error)`:
1. Gọi `staffRepo.FindProfileByUserID(ctx, userID)`. Return `ErrNotFound` nếu không có profile.
2. Gọi `staffRepo.CountPortfolioPhotos(ctx, profile.ID)`. Nếu count >= 12 → return `ErrPhotoLimitReached` (custom error).
3. Validate `photo_url` bắt đầu bằng `cfg.StoragePublicURL`. Return `ErrValidation` nếu không match.
4. Xác định `display_order`: nếu `req.DisplayOrder != nil` dùng giá trị đó, else gọi `staffRepo.MaxDisplayOrder(ctx, profile.ID) + 1`.
5. Create: `&model.StaffPortfolioPhoto{ID: ulid.New(), StaffProfileID: profile.ID, PhotoURL: req.PhotoURL, DisplayOrder: displayOrder}`
6. Gọi `staffRepo.AddPortfolioPhoto(ctx, photo)`. Return photo.

Method `DeletePhoto(ctx, userID, photoID string) error`:
1. Gọi `staffRepo.GetPortfolioPhotoByID(ctx, photoID)`. Return `ErrNotFound` nếu không có.
2. Gọi `staffRepo.FindProfileByUserID(ctx, userID)`. Return `ErrNotFound` nếu không có profile.
3. Verify ownership: `photo.StaffProfileID != profile.ID` → return `model.ErrForbidden`.
4. Extract `fileKey` từ `photo.PhotoURL` (xóa `STORAGE_PUBLIC_URL` prefix).
5. Gọi `storageClient.DeleteObject(ctx, fileKey)` — không fail nếu không tìm thấy file (file đã xóa là OK).
6. Gọi `staffRepo.DeletePortfolioPhoto(ctx, photoID)`.

Method `ReorderPhotos(ctx, userID string, req dto.ReorderPhotosRequest) error`:
1. Gọi `staffRepo.FindProfileByUserID(ctx, userID)`. Return `ErrNotFound` nếu không có profile.
2. Gọi `staffRepo.UpdateDisplayOrders(ctx, profile.ID, req.PhotoOrders)`.

### Step 3: Tạo DTOs

**File:** `internal/dto/staff_dto.go` (thêm vào)

```go
type AddPortfolioPhotoRequest struct {
  PhotoURL     string `json:"photo_url"`
  DisplayOrder *int   `json:"display_order"`
}

type PhotoOrder struct {
  ID    string `json:"id"`
  Order int    `json:"order"`
}

type ReorderPhotosRequest struct {
  PhotoOrders []PhotoOrder `json:"photo_orders"`
}

type PortfolioPhotoResponse struct {
  ID             string    `json:"id"`
  StaffProfileID string    `json:"staff_profile_id"`
  PhotoURL       string    `json:"photo_url"`
  DisplayOrder   int       `json:"display_order"`
  CreatedAt      time.Time `json:"created_at"`
}
```

Error sentinels thêm vào `internal/model/error.go`:
- `ErrPhotoLimitReached = errors.New("photo_limit_reached")`
- `ErrForbidden = errors.New("forbidden")`

### Step 4: Thêm handlers vào StaffHandler

**File:** `internal/handler/staff_handler.go` (thêm methods)

Method `AddPortfolioPhoto(c fiber.Ctx) error`:
1. Read `userID` từ JWT
2. Parse body. Return `400` nếu `photo_url` trống.
3. Gọi `portfolioService.AddPhoto(ctx, userID, req)`.
4. Map: `ErrNotFound` → 404, `ErrPhotoLimitReached` → 422 "Portfolio is full. Maximum 12 photos allowed.", `ErrValidation` → 422.
5. Return `201` với `PortfolioPhotoResponse`.

Method `DeletePortfolioPhoto(c fiber.Ctx) error`:
1. Read `userID` từ JWT, `photoID` từ `c.Params("photoID")`
2. Gọi `portfolioService.DeletePhoto(ctx, userID, photoID)`.
3. Map: `ErrNotFound` → 404, `ErrForbidden` → 403 "You do not have permission to delete this photo.".
4. Return `204`.

Method `ReorderPortfolioPhotos(c fiber.Ctx) error`:
1. Read `userID` từ JWT
2. Parse body `ReorderPhotosRequest`. Return `400` nếu `photo_orders` rỗng.
3. Gọi `portfolioService.ReorderPhotos(ctx, userID, req)`.
4. Return `200 { message: "Order updated." }`.

### Step 5: Đăng ký routes

**File:** `router/router.go`

```go
portfolio := staff.Group("/portfolio")
portfolio.Post("/photos", staffHandler.AddPortfolioPhoto)
portfolio.Delete("/photos/:photoID", staffHandler.DeletePortfolioPhoto)
portfolio.Patch("/photos/reorder", staffHandler.ReorderPortfolioPhotos)
```

---

## Flutter Steps

### Step 1: Tạo PortfolioPhoto model

**File:** `lib/models/portfolio_photo.dart`

```dart
class PortfolioPhoto {
  final String id;
  final String staffProfileId;
  final String photoUrl;
  final int displayOrder;
  final DateTime createdAt;

  factory PortfolioPhoto.fromJson(Map<String, dynamic> json) { ... }
}
```

### Step 2: Thêm portfolio methods vào StaffService

**File:** `lib/services/staff_service.dart` (thêm)

- `addPortfolioPhoto(String photoUrl, int? displayOrder)` → `Future<PortfolioPhoto>`:
  - POST `/api/v1/staff/portfolio/photos` với `{photo_url, display_order?}`

- `deletePortfolioPhoto(String photoID)` → `Future<void>`:
  - DELETE `/api/v1/staff/portfolio/photos/$photoID`

- `reorderPortfolioPhotos(List<{id, order}> orders)` → `Future<void>`:
  - PATCH `/api/v1/staff/portfolio/photos/reorder` với `{photo_orders: [...]}`

### Step 3: Cập nhật StaffProvider

**File:** `lib/providers/staff_provider.dart` (thêm)

Thêm `List<PortfolioPhoto> portfolioPhotos` vào state (sync với `currentProfile.portfolioPhotos`).

Method `addPortfolioPhoto(XFile file)` → `Future<void>`:
1. `isLoading = true`, notifyListeners()
2. Gọi `UploadService.instance.uploadFile(file, UploadFolder.portfolio)`
3. Sau khi upload thành công: gọi `StaffService.addPortfolioPhoto(result.publicUrl)`
4. Optimistic update: thêm photo mới vào `portfolioPhotos` list
5. Rollback nếu API fail
6. `finally: isLoading = false, notifyListeners()`

Method `deletePortfolioPhoto(String photoID)` → `Future<void>`:
1. Confirm dialog trước khi gọi (xử lý ở UI layer)
2. Optimistic remove từ list
3. Gọi API, rollback nếu fail

Method `reorderPortfolioPhotos(List<PortfolioPhoto> newOrder)` → `Future<void>`:
1. Optimistic update local list
2. Gọi `StaffService.reorderPortfolioPhotos(...)` với `newOrder.map((p, i) => {id: p.id, order: i})`
3. Rollback nếu fail

### Step 4: Tạo PortfolioEditScreen

**File:** `lib/screens/staff/portfolio_edit_screen.dart` (tạo mới)

Layout: `Scaffold` với AppBar title "(3/12)" (count hiện tại):
- `ReorderableGridView` (hoặc `GridView.builder` với drag support) — 3 columns
- Mỗi photo tile: `CachedNetworkImage` + delete icon overlay (IconButton top-right)
- "Add Photo" tile: chỉ hiển thị khi count < 12, tapping launch `ImagePicker`

Init: đọc từ `StaffProvider.portfolioPhotos`.

**Add photo flow:**
1. Mở `ImagePicker.pickImage(source: ImageSource.gallery)`
2. Nếu null → return (user cancel)
3. Show loading overlay trên new tile area
4. Gọi `StaffProvider.addPortfolioPhoto(file)`
5. Nếu `UploadValidationException`: show red SnackBar
6. Nếu thành công: show green SnackBar "Photo added to portfolio."

**Delete photo flow:**
1. Tap delete icon → show `AlertDialog`:
   - Title: "Remove Photo"
   - Content: "Remove this photo from your portfolio?"
   - Actions: "Cancel" + "Remove" (màu đỏ)
2. Nếu confirm: gọi `StaffProvider.deletePortfolioPhoto(photoID)`
3. Nếu thành công: show green SnackBar "Photo removed."
4. Nếu 403: show red SnackBar "You do not have permission to delete this photo."

**Reorder flow:**
1. User drag-drop photo
2. `ReorderableGridView.onReorder` callback cập nhật local list order
3. Gọi `StaffProvider.reorderPortfolioPhotos(newOrder)`

### Step 5: Thêm "Edit Portfolio" button vào StaffProfileScreen

**File:** `lib/screens/staff/staff_profile_screen.dart` (modify)

Thêm logic phân biệt owner vs visitor:
```dart
final isOwner = AuthProvider.of(context).currentUser?.id == profile.userId;
```

Nếu `isOwner`:
- Hiển thị "Edit Profile" và "Edit Portfolio" buttons
- "Edit Portfolio" → navigate to `PortfolioEditScreen`

Nếu không phải owner:
- Hiển thị "Follow", "Book Now", "Tip" buttons

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/model/error.go` | MODIFY — thêm ErrPhotoLimitReached, ErrForbidden |
| `internal/repository/staff_repository.go` | MODIFY — thêm portfolio methods |
| `internal/service/staff_portfolio_service.go` | CREATE |
| `internal/dto/staff_dto.go` | MODIFY — thêm portfolio DTOs |
| `internal/handler/staff_handler.go` | MODIFY — thêm 3 portfolio handlers |
| `router/router.go` | MODIFY — thêm portfolio routes |
| `main.go` | MODIFY — wire StaffPortfolioService |

### Flutter
| File | Action |
|---|---|
| `lib/models/portfolio_photo.dart` | CREATE |
| `lib/services/staff_service.dart` | MODIFY — thêm portfolio methods |
| `lib/providers/staff_provider.dart` | MODIFY — thêm portfolio state + methods |
| `lib/screens/staff/portfolio_edit_screen.dart` | CREATE |
| `lib/screens/staff/staff_profile_screen.dart` | MODIFY — thêm owner/visitor mode, Edit Portfolio button |
| `pubspec.yaml` | MODIFY — thêm image_picker nếu chưa có |

---

## Verification Checklist

- [ ] `POST /api/v1/staff/portfolio/photos` → 201, photo thêm vào DB
- [ ] `POST` khi đã có 12 photos → 422 "Portfolio is full..."
- [ ] `POST` với photo_url không bắt đầu từ STORAGE_PUBLIC_URL → 422
- [ ] `POST` không có staff profile → 404
- [ ] `DELETE /api/v1/staff/portfolio/photos/:id` (own photo) → 204, file xóa khỏi R2
- [ ] `DELETE` photo của staff khác → 403
- [ ] `PATCH /api/v1/staff/portfolio/photos/reorder` → 200, display_order updated
- [ ] Flutter: File > 10MB → SnackBar "File is too large. Maximum size is 10 MB."
- [ ] Flutter: GIF file → SnackBar "Unsupported file type..."
- [ ] Flutter: Add photo thành công → grid cập nhật, SnackBar "Photo added to portfolio."
- [ ] Flutter: Delete confirmation dialog xuất hiện, cancel không xóa
- [ ] Flutter: Drag reorder → thứ tự lưu lại sau reload
- [ ] Flutter: "Add Photo" tile ẩn khi count = 12
- [ ] Flutter: Owner thấy Edit Portfolio button, visitor không thấy
