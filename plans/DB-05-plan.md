# [DB-05] Storage Setup for Media — Implementation Plan

**Spec:** `docs/screens/DB-05-storage-setup-for-media.md`
**Priority:** P1 MVP
**Phụ thuộc:** AUTH-01 (`ApiClient`, JWT auth đã có)

---

## Mục tiêu

Tạo infrastructure upload file cho toàn bộ app:
- Backend: 2 endpoints (generate presigned URL + delete file)
- Flutter: `UploadService` singleton dùng cho mọi feature upload (portfolio, post, avatar, ...)
- Storage: Cloudflare R2 (S3-compatible)

---

## Backend Steps

### Step 1: Thêm AWS SDK Go v2 dependency

```bash
~/go/bin/go1.26.1 get github.com/aws/aws-sdk-go-v2
~/go/bin/go1.26.1 get github.com/aws/aws-sdk-go-v2/config
~/go/bin/go1.26.1 get github.com/aws/aws-sdk-go-v2/service/s3
~/go/bin/go1.26.1 get github.com/aws/aws-sdk-go-v2/credentials
~/go/bin/go1.26.1 mod tidy
```

### Step 2: Thêm Storage config vào Config struct

**File:** `internal/config/config.go`

Thêm fields:
```go
StorageProvider        string // "r2" hoặc "s3"
StorageBucket          string
StorageRegion          string // default "auto" cho R2
StorageAccessKeyID     string
StorageSecretAccessKey string
StorageEndpoint        string // custom endpoint cho R2: https://<account_id>.r2.cloudflarestorage.com
StoragePublicURL       string // CDN base URL: https://media.staffsearch.jp
StorageURLExpirySeconds int   // default 900 (15 minutes)
```

Load từ env vars: `STORAGE_PROVIDER`, `STORAGE_BUCKET`, `STORAGE_REGION`, `STORAGE_ACCESS_KEY_ID`, `STORAGE_SECRET_ACCESS_KEY`, `STORAGE_ENDPOINT`, `STORAGE_PUBLIC_URL`, `STORAGE_URL_EXPIRY_SECONDS`.

**File:** `.env.example`

Thêm tất cả STORAGE_* vars với placeholder values.

### Step 3: Tạo Storage package

**File:** `pkg/storage/storage.go`

Interface `StorageClient`:
```go
type StorageClient interface {
  GeneratePresignedPutURL(ctx context.Context, key string, contentType string, expiry time.Duration) (string, error)
  DeleteObject(ctx context.Context, key string) error
}
```

Struct `S3StorageClient` implement interface:
- Constructor `NewS3StorageClient(cfg *config.Config) (*S3StorageClient, error)`:
  1. Load AWS config với `aws.Config{Credentials: credentials.NewStaticCredentialsProvider(...)}`
  2. Nếu `cfg.StorageProvider == "r2"`: set custom `EndpointResolver` trỏ đến `cfg.StorageEndpoint`, set `Region: cfg.StorageRegion`
  3. Tạo `s3.Client` từ config
  4. Tạo `s3.PresignClient` từ s3 client

Method `GeneratePresignedPutURL(ctx, key, contentType string, expiry time.Duration) (string, error)`:
- Dùng `presignClient.PresignPutObject()` với `s3.PutObjectInput{Bucket: &bucket, Key: &key, ContentType: &contentType}`
- Return presigned URL string

Method `DeleteObject(ctx, key string) error`:
- Dùng `s3Client.DeleteObject(ctx, &s3.DeleteObjectInput{Bucket: &bucket, Key: &key})`
- Nếu object không tồn tại (NoSuchKey error) → return `model.ErrNotFound`

### Step 4: Tạo Media DTOs

**File:** `internal/dto/media_dto.go`

Structs:
- `UploadURLRequest` — `FileName` string, `ContentType` string, `Folder` string
- `UploadURLResponse` — `UploadURL` string, `FileKey` string, `PublicURL` string, `ExpiresIn` int
- `DeleteMediaRequest` — `FileKey` string

### Step 5: Tạo MediaService

**File:** `internal/service/media_service.go`

Struct `MediaService` với fields `storage StorageClient`, `cfg *config.Config`.

Allowed content types:
```go
var allowedContentTypes = map[string]bool{
  "image/jpeg": true, "image/png": true, "image/webp": true,
  "video/mp4": true, "video/quicktime": true,
}
```

Allowed folders:
```go
var allowedFolders = map[string]bool{
  "avatars": true, "portfolio": true, "posts": true,
  "stories": true, "chat": true, "intro_videos": true,
}
```

Extension-contentType mapping:
```go
var extToContentType = map[string]string{
  ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
  ".png": "image/png", ".webp": "image/webp",
  ".mp4": "video/mp4", ".mov": "video/quicktime",
}
```

Method `GenerateUploadURL(ctx, userID string, req dto.UploadURLRequest) (dto.UploadURLResponse, error)`:
1. Validate `req.ContentType` trong `allowedContentTypes` → `model.ErrValidation` nếu không có
2. Validate `req.Folder` trong `allowedFolders` → `model.ErrValidation` nếu không có
3. Lấy extension từ `req.FileName`. Check extension match với `req.ContentType` qua `extToContentType` → `model.ErrValidation` nếu mismatch
4. Build file key: `fmt.Sprintf("%s/%s/%s%s", req.Folder, userID, ulid.New(), filepath.Ext(req.FileName))`
5. Gọi `storage.GeneratePresignedPutURL(ctx, key, req.ContentType, time.Duration(cfg.StorageURLExpirySeconds)*time.Second)`
6. Build `publicURL = cfg.StoragePublicURL + "/" + key`
7. Return `UploadURLResponse{UploadURL: presignedURL, FileKey: key, PublicURL: publicURL, ExpiresIn: cfg.StorageURLExpirySeconds}`

Method `DeleteFile(ctx, fileKey string) error`:
- Gọi `storage.DeleteObject(ctx, fileKey)`
- Return lỗi trực tiếp (ErrNotFound sẽ được map bởi handler)

### Step 6: Tạo MediaHandler

**File:** `internal/handler/media_handler.go`

Struct `MediaHandler` với field `mediaService *service.MediaService`.

Method `GenerateUploadURL(c fiber.Ctx) error`:
1. Read `userID` từ `c.Locals("userID").(string)`
2. Parse body vào `dto.UploadURLRequest`. Return `400` nếu fail hoặc fields trống.
3. Validate: `file_name` max 255 chars, `content_type` không trống, `folder` không trống.
4. Gọi `mediaService.GenerateUploadURL(ctx, userID, req)`.
5. Map `model.ErrValidation` → `422 validation_error`.
6. Return `200` với `UploadURLResponse`.

Method `DeleteFile(c fiber.Ctx) error`:
1. Read `userID` từ `c.Locals("userID")`
2. Parse body `dto.DeleteMediaRequest`. Return `400` nếu `file_key` trống.
3. Verify: `fileKey` phải chứa `userID` như một path segment (`strings.Contains(fileKey, "/"+userID+"/")`). Return `403 forbidden` nếu không match.
4. Gọi `mediaService.DeleteFile(ctx, fileKey)`.
5. Map `model.ErrNotFound` → `404 not_found`.
6. Return `204` (no body).

### Step 7: Đăng ký routes

**File:** `router/router.go`

Trong protected group:
```go
media := protected.Group("/media")
media.Post("/upload-url", mediaHandler.GenerateUploadURL)
media.Delete("", mediaHandler.DeleteFile)
```

### Step 8: Update wire-up trong main.go

**File:** `main.go`

```go
storageClient, err := pkg_storage.NewS3StorageClient(cfg)
if err != nil { log.Fatal(err) }
mediaService := service.NewMediaService(storageClient, cfg)
mediaHandler := handler.NewMediaHandler(mediaService)
```

Pass `mediaHandler` vào `router.Setup()`.

---

## Flutter Steps

### Step 1: Thêm package http nếu chưa có

**File:** `pubspec.yaml`

```yaml
http: ^1.2.0
image_picker: ^1.1.2  # nếu chưa có
```

### Step 2: Tạo UploadFolder enum và constants

**File:** `lib/models/upload_result.dart`

```dart
enum UploadFolder {
  avatar,   // → "avatars", max 5MB, jpg/png/webp
  portfolio, // → "portfolio", max 10MB, jpg/png/webp
  post,     // → "posts", max 100MB, jpg/png/webp/mp4
  story,    // → "stories", max 50MB, jpg/png/webp/mp4
  chat,     // → "chat", max 10MB, jpg/png/webp
  introVideo // → "intro_videos", max 100MB, mp4/mov
}
```

Class `UploadResult`:
- `fileKey` String
- `publicUrl` String

Constants per folder (maxSizeBytes, allowedMimeTypes, folderName String).

### Step 3: Tạo Custom Exceptions

**File:** `lib/models/upload_result.dart` (thêm vào cùng file)

```dart
class UploadValidationException implements Exception {
  final String message;
  UploadValidationException(this.message);
}

class UploadNetworkException implements Exception {}

class UploadExpiredException implements Exception {}
```

### Step 4: Tạo UploadService

**File:** `lib/services/upload_service.dart`

Class `UploadService` singleton:

Method `uploadFile(XFile file, UploadFolder folder)` → `Future<UploadResult>`:

**Validation step:**
1. Đọc `mimeType` từ file (dùng `mime` package hoặc check extension)
2. Check `mimeType` trong allowedMimeTypes của folder → throw `UploadValidationException("Unsupported file type. Please select a JPEG, PNG, WebP, or MP4 file.")` nếu không hợp lệ
3. Đọc file bytes, check size ≤ maxSizeBytes → throw `UploadValidationException("File is too large. Maximum size is X MB.")` nếu quá lớn

**Get presigned URL step:**
4. Gọi `ApiClient.post('/api/v1/media/upload-url', {file_name: fileName, content_type: mimeType, folder: folderName})`
5. Nếu network error → throw `UploadNetworkException()`
6. Parse `upload_url`, `file_key`, `public_url` từ response

**Upload to R2 step:**
7. Tạo `http.Request('PUT', Uri.parse(uploadUrl))`
8. Set `request.headers['Content-Type'] = mimeType`
9. Set `request.bodyBytes = await file.readAsBytes()`
10. Send request (KHÔNG dùng ApiClient — direct HTTP, không có JWT header)
11. Nếu response status 403 → throw `UploadExpiredException()`
12. Nếu network error → throw `UploadNetworkException()`
13. Return `UploadResult(fileKey: fileKey, publicUrl: publicUrl)`

**Folder spec table** (hardcoded trong UploadService):
| Enum | folderName | maxMB | allowedTypes |
|---|---|---|---|
| avatar | avatars | 5 | jpg, png, webp |
| portfolio | portfolio | 10 | jpg, png, webp |
| post | posts | 100 | jpg, png, webp, mp4 |
| story | stories | 50 | jpg, png, webp, mp4 |
| chat | chat | 10 | jpg, png, webp |
| introVideo | intro_videos | 100 | mp4, mov |

### Step 5: Cách dùng UploadService trong screens

Các screen sau này dùng `UploadService` theo pattern:
```dart
try {
  setState(() => _isUploading = true);
  final result = await UploadService.instance.uploadFile(file, UploadFolder.portfolio);
  // dùng result.publicUrl để lưu vào API
} on UploadValidationException catch (e) {
  // show red SnackBar với e.message
} on UploadExpiredException {
  // show red SnackBar: "Upload session expired. Please try again."
} on UploadNetworkException {
  // show red SnackBar: "Upload failed. Check your connection and try again."
} finally {
  setState(() => _isUploading = false);
}
```

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `go.mod` / `go.sum` | MODIFY — thêm aws-sdk-go-v2 |
| `internal/config/config.go` | MODIFY — thêm Storage* fields |
| `.env.example` | MODIFY — thêm STORAGE_* vars |
| `pkg/storage/storage.go` | CREATE — StorageClient interface + S3/R2 implementation |
| `internal/dto/media_dto.go` | CREATE |
| `internal/service/media_service.go` | CREATE |
| `internal/handler/media_handler.go` | CREATE |
| `router/router.go` | MODIFY — đăng ký media routes |
| `main.go` | MODIFY — wire StorageClient, MediaService, MediaHandler |

### Flutter
| File | Action |
|---|---|
| `pubspec.yaml` | MODIFY — thêm http, image_picker |
| `lib/models/upload_result.dart` | CREATE — UploadResult, UploadFolder, exceptions |
| `lib/services/upload_service.dart` | CREATE |

---

## Verification Checklist

- [ ] `POST /api/v1/media/upload-url` với JPEG + folder=avatars → 200, upload_url + file_key + public_url
- [ ] Raw HTTP PUT đến upload_url thành công, file accessible tại public_url
- [ ] `POST /api/v1/media/upload-url` với unsupported content_type → 422
- [ ] `POST /api/v1/media/upload-url` với mismatch extension/content_type → 422
- [ ] `POST /api/v1/media/upload-url` không có JWT → 401
- [ ] `DELETE /api/v1/media` với own file key → 204, file deleted từ R2
- [ ] `DELETE /api/v1/media` với file key của user khác → 403
- [ ] `DELETE /api/v1/media` với non-existent key → 404
- [ ] file_key format đúng: `<folder>/<userID>/<ulid>.<ext>`
- [ ] Flutter: File > 5MB với folder=avatar → UploadValidationException, không gọi API
- [ ] Flutter: PDF file → UploadValidationException "Unsupported file type..."
- [ ] Flutter: Network error trên R2 PUT → UploadNetworkException
- [ ] STORAGE credentials không xuất hiện trong bất kỳ API response hoặc log
