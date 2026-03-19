# [DB-05] Storage Setup for Media (Images, Videos)

| Field | Value |
|---|---|
| **Type** | Infrastructure |
| **Priority** | P1 MVP |
| **Estimate** | 1.5 days manual / 0.5 days vibe coding |
| **Dependencies** | DB-01 — `avatar_url` on `users`, `portfolio_photos` on `staff_profiles`, and all other media URL columns defined in Section 6. AUTH-01 — `ApiClient` and JWT authentication created in Sections 4.1–4.4. |
| **Platforms** | iOS · Android · Web |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format (Section 2), JWT middleware (Section 3), GORM conventions (Section 4), `ApiClient` (Section 6), error display (Section 10). |

---

## 1. Overview

This ticket sets up Cloudflare R2 (S3-compatible) object storage for all user-generated media in the staffsearch platform: profile photos, staff portfolio photos, post images and videos, stories, chat attachments, and staff intro videos. It delivers a presigned URL upload API on the Go backend and an `UploadService` on the Flutter side.

No downstream feature ticket (AUTH-09, STAFF-05, STAFF-06, FEED-01, CHAT-04) needs to implement its own file upload logic — all uploads go through the shared infrastructure established here.

The upload flow uses presigned PUT URLs so that large files are sent directly from the client device to R2, bypassing the API server entirely. The API server only generates the presigned URL; it is never a proxy for binary file data.

### End-to-end flow — File Upload

    [Flutter App]                   [Go API]                   [Cloudflare R2]
         │                               │                           │
         │── POST /api/v1/media/──────►  │                           │
         │       upload-url              │── GeneratePresignedPUT ──►│
         │   {file_name, content_type,   │◄── presigned URL ─────────│
         │    folder}                    │                           │
         │◄── {upload_url, file_key, ────│                           │
         │     public_url}               │                           │
         │                               │                           │
         │── PUT <upload_url> ──────────────────────────────────────►│
         │   raw bytes (direct to R2)    │                           │
         │◄── 200 OK ────────────────────────────────────────────────│
         │                               │                           │
         ├── store public_url in field   │                           │
         └── call domain endpoint (e.g. PATCH /users/me             │
             with avatar_url = public_url)                           │

### End-to-end flow — File Delete

    [Flutter App]                   [Go API]                   [Cloudflare R2]
         │                               │                           │
         │── DELETE /api/v1/media ──────►│                           │
         │   {file_key}                  │── DeleteObject ──────────►│
         │◄── 204 No Content ────────────│                           │

---

## 2. User Flows

### 2.1 Happy Path — Upload a Media File

1. A screen that requires media upload (e.g. profile photo, post image) calls `UploadService.uploadFile(file, folder)`.
2. `UploadService` validates the file's size and MIME type against the limits for the given folder. If validation fails, a red SnackBar is shown immediately and no API call is made.
3. `UploadService` calls `POST /api/v1/media/upload-url` via `ApiClient` with the file name, content type, and destination folder.
4. The API returns `upload_url` (a presigned R2 PUT URL valid for 15 minutes), `file_key` (the storage path), and `public_url` (the CDN-accessible URL).
5. `UploadService` performs a raw HTTP PUT request directly to `upload_url` with the file bytes as the body. This request does not go through `ApiClient` and does not include the JWT header.
6. R2 returns `200 OK`.
7. `UploadService` returns `public_url` to the caller.
8. The calling screen stores `public_url` in the relevant model field and persists it via the appropriate domain API endpoint (e.g. `PATCH /api/v1/users/me` with `avatar_url`).

### 2.2 Error Cases

| Scenario | Behaviour |
|---|---|
| File size exceeds folder limit | `UploadService` rejects before API call. Red SnackBar: `"File is too large. Maximum size is <X> MB."` |
| Unsupported MIME type | `UploadService` rejects before API call. Red SnackBar: `"Unsupported file type. Please select a JPEG, PNG, WebP, or MP4 file."` |
| 401 on presigned URL request — refresh fails | `ApiClient` clears tokens and redirects to `/login`. |
| 500 on presigned URL request | Red SnackBar: `"Something went wrong. Please try again later."` |
| Network error on presigned URL request | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| R2 PUT fails (network error) | Red SnackBar: `"Upload failed. Check your connection and try again."` |
| Presigned URL expired before PUT (R2 returns 403) | Red SnackBar: `"Upload session expired. Please try again."` |

---

## 3. UI / Screen

DB-05 provides infrastructure only — there is no dedicated upload screen. The upload interaction is embedded in existing screens (profile photo in `profile_edit_screen.dart`, portfolio photos in staff profile screens, post creation in `create_post_screen.dart`, etc.). Those screens will be wired to `UploadService` in their respective feature tickets (AUTH-09, STAFF-05, STAFF-06, FEED-01, CHAT-04).

The only shared UI convention is a `CircularProgressIndicator` (20×20 px, strokeWidth 2) on the control that triggered the upload while the presigned URL request and the R2 PUT are in progress. The triggering control is disabled during this period. Loading state is cleared in a finally block whether upload succeeds or fails.

Not applicable as a standalone screen.

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/services/upload_service.dart` | Handles the full presigned URL upload flow: client-side validation, fetch presigned URL via ApiClient, PUT file bytes directly to R2, return public URL. |
| `lib/models/upload_result.dart` | Simple value object containing `fileKey` and `publicUrl`. Returned by `UploadService.uploadFile()`. |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `pubspec.yaml` | Add `http: ^1.2.0` if not already present (used for the direct R2 PUT with raw byte body, separate from ApiClient's dio/http usage). |

### 4.3 UploadService Responsibilities

`UploadService` is a singleton that provides a single public method:

`uploadFile(XFile file, UploadFolder folder)` — Validates the file's MIME type and byte size against the limits table below. If validation fails, throws `UploadValidationException` with a user-facing message. If validation passes, calls `POST /api/v1/media/upload-url` via `ApiClient`. On receiving the presigned URL, performs a raw HTTP PUT to that URL with the file bytes. Returns an `UploadResult` containing `fileKey` and `publicUrl` on success, or throws `UploadNetworkException` on network failure, or `UploadExpiredException` if R2 returns 403.

The `UploadFolder` enum values and their limits:

| Enum Value | Storage Folder | Max Size | Allowed MIME Types |
|---|---|---|---|
| `avatar` | `avatars` | 5 MB | image/jpeg, image/png, image/webp |
| `portfolio` | `portfolio` | 10 MB | image/jpeg, image/png, image/webp |
| `post` | `posts` | 100 MB | image/jpeg, image/png, image/webp, video/mp4 |
| `story` | `stories` | 50 MB | image/jpeg, image/png, image/webp, video/mp4 |
| `chat` | `chat` | 10 MB | image/jpeg, image/png, image/webp |
| `introVideo` | `intro_videos` | 100 MB | video/mp4, video/quicktime |

Client-side size limits are enforced before the API call. These limits are also enforced on the R2 side via presigned URL conditions where the provider supports them.

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/media/upload-url` | Generate a presigned R2/S3 PUT URL for a file upload | Yes |
| `DELETE` | `/api/v1/media` | Delete a stored file by its file key | Yes |

### 5.2 POST /api/v1/media/upload-url

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `file_name` | string | Yes | 1–255 characters; must end with an allowed extension (.jpg, .jpeg, .png, .webp, .mp4, .mov) |
| `content_type` | string | Yes | Must be one of: `image/jpeg`, `image/png`, `image/webp`, `video/mp4`, `video/quicktime` |
| `folder` | string | Yes | Must be one of: `avatars`, `portfolio`, `posts`, `stories`, `chat`, `intro_videos` |

**Response `200`:**

| Field | Type | Description |
|---|---|---|
| `upload_url` | string | Presigned PUT URL pointing directly to R2/S3. Valid for 15 minutes. |
| `file_key` | string | The storage object key (e.g. `avatars/<userID>/<ulid>.jpg`). Store this to reference or delete the file later. |
| `public_url` | string | The CDN-accessible URL for the file after upload completes. This is the value to store in the database. |
| `expires_in` | int | Seconds until `upload_url` expires. Always 900. |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed JSON body or missing required fields |
| `401` | `unauthorized` | Missing or invalid JWT |
| `422` | `validation_error` | Unsupported `content_type`; unknown `folder`; file extension does not match `content_type`; `file_name` exceeds 255 characters |
| `500` | `server_error` | R2/S3 SDK error or unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Parse the request body into `UploadURLRequest` DTO. Return `400 bad_request` if parsing fails.
3. Validate `content_type` against the allowed list. Return `422 validation_error` if not in the list.
4. Validate `folder` against the enum. Return `422 validation_error` if unknown.
5. Validate that the `file_name` extension is consistent with `content_type` (e.g. `.jpg` with `image/jpeg`). Return `422 validation_error` if mismatched.
6. Call `MediaService.GenerateUploadURL(ctx, userID, req)`. Return `500 server_error` if the service returns an error.
7. Return `200` with the `UploadURLResponse` DTO.

### 5.3 DELETE /api/v1/media

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `file_key` | string | Yes | Non-empty; must begin with one of the allowed folder prefixes (`avatars/`, `portfolio/`, `posts/`, `stories/`, `chat/`, `intro_videos/`) |

**Response `204`:** No body.

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed body or `file_key` is empty |
| `401` | `unauthorized` | Missing or invalid JWT |
| `403` | `forbidden` | The `file_key` does not contain the authenticated user's ID as a path segment |
| `404` | `not_found` | The file does not exist in storage |
| `500` | `server_error` | R2/S3 SDK error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Parse the request body. Return `400` if parsing fails or `file_key` is empty.
3. Verify that `file_key` contains `userID` as a path segment (format: `<folder>/<userID>/<ulid>.<ext>`). Return `403` if the userID segment does not match.
4. Call `MediaService.DeleteFile(ctx, fileKey)`. Return `404 not_found` if the service returns `model.ErrNotFound`. Return `500` for other errors.
5. Return `204` with no body.

### 5.4 Service Responsibilities

`MediaService`:

- `GenerateUploadURL(ctx context.Context, userID string, req UploadURLRequest) (UploadURLResponse, error)` — Constructs a unique file key by joining the folder, userID, a new ULID (via `pkg/ulid`), and the original file extension. Calls `pkg/storage.GeneratePresignedPutURL(ctx, key, contentType, 15*time.Minute)`. Prepends `STORAGE_PUBLIC_URL` to the key to form `public_url`. Returns the composed `UploadURLResponse`.

- `DeleteFile(ctx context.Context, fileKey string) error` — Calls `pkg/storage.DeleteObject(ctx, fileKey)`. Returns `model.ErrNotFound` if the object does not exist in storage.

### 5.5 Storage Package

`pkg/storage/storage.go` defines a `StorageClient` interface with two methods:

- `GeneratePresignedPutURL(ctx context.Context, key string, contentType string, expiry time.Duration) (string, error)` — Returns a presigned PUT URL.
- `DeleteObject(ctx context.Context, key string) error` — Deletes the object; returns a typed not-found error if the key does not exist.

The concrete implementation uses the AWS SDK for Go v2 (`github.com/aws/aws-sdk-go-v2`) with a custom endpoint resolver. When `STORAGE_PROVIDER=r2`, the endpoint is set to `STORAGE_ENDPOINT` (the Cloudflare R2 account endpoint). When `STORAGE_PROVIDER=s3`, standard AWS S3 routing is used. The implementation is constructed in `main.go` and injected into `MediaService`.

---

## 6. Database

Not applicable for this ticket. Media file URLs are stored as plain string columns on the relevant domain tables, as defined in DB-01. No dedicated media metadata table is required for MVP. Callers store the `public_url` returned by this ticket's endpoint on their own domain records (e.g., `avatar_url` on `users`, `photo_url` on `staff_portfolio_photos`).

---

## 7. Configuration

| Variable | Default | Description |
|---|---|---|
| `STORAGE_PROVIDER` | `r2` | Storage backend. Accepted values: `r2` (Cloudflare R2) or `s3` (AWS S3). |
| `STORAGE_BUCKET` | required | Name of the R2 or S3 bucket. |
| `STORAGE_REGION` | `auto` | AWS/R2 region. Use `auto` for Cloudflare R2. |
| `STORAGE_ACCESS_KEY_ID` | required | R2/S3 access key ID. For R2: generate in the Cloudflare dashboard under R2 → Manage R2 API Tokens. |
| `STORAGE_SECRET_ACCESS_KEY` | required | R2/S3 secret access key. |
| `STORAGE_ENDPOINT` | — | Custom endpoint URL. Required for R2; format: `https://<account_id>.r2.cloudflarestorage.com`. Leave blank for standard AWS S3. |
| `STORAGE_PUBLIC_URL` | required | CDN base URL for public file access (e.g. `https://media.staffsearch.jp`). Prepended to `file_key` to form the `public_url` in responses. |
| `STORAGE_URL_EXPIRY_SECONDS` | `900` | How long presigned PUT URLs remain valid. Defaults to 15 minutes. |

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Valid POST — `folder=avatars`, JPEG content type | `200`, `upload_url`, `file_key`, and `public_url` returned; key is `avatars/<userID>/<ulid>.jpg` |
| 2 | Valid POST — `folder=posts`, MP4 content type | `200`, valid presigned URL returned |
| 3 | Valid POST — `folder=intro_videos`, MOV content type | `200`, valid presigned URL returned |
| 4 | Missing `file_name` | `400 bad_request` |
| 5 | `content_type` is `application/pdf` | `422 validation_error` |
| 6 | Unknown `folder` value (`documents`) | `422 validation_error` |
| 7 | `file_name` is `photo.png` but `content_type` is `video/mp4` | `422 validation_error` |
| 8 | No Authorization header on upload-url request | `401 unauthorized` |
| 9 | Valid DELETE — own file key | `204`, object deleted from R2 |
| 10 | DELETE — file key belongs to a different user | `403 forbidden` |
| 11 | DELETE — file key does not exist | `404 not_found` |
| 12 | R2/S3 SDK unreachable | `500 server_error` |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Valid JPEG under 5 MB with `folder=avatar` | `UploadResult` returned with `publicUrl`; no error shown |
| 2 | JPEG file exceeding 5 MB with `folder=avatar` | Red SnackBar: `"File is too large. Maximum size is 5 MB."` No API call made. |
| 3 | PDF file selected | Red SnackBar: `"Unsupported file type. Please select a JPEG, PNG, WebP, or MP4 file."` No API call made. |
| 4 | Network error on presigned URL request | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| 5 | Network error on R2 PUT | Red SnackBar: `"Upload failed. Check your connection and try again."` |
| 6 | R2 returns 403 (URL expired) | Red SnackBar: `"Upload session expired. Please try again."` |
| 7 | Upload in progress | Triggering control shows `CircularProgressIndicator` 20×20 px and is disabled |
| 8 | Upload completes (success or failure) | Loading state always cleared; control re-enabled |

---

## 9. Security Checklist

- [ ] Presigned URLs include the authenticated user's ID as a path segment in the file key, namespacing uploads per user.
- [ ] The DELETE endpoint verifies the file key contains the authenticated user's ID before deleting; users cannot delete files belonging to others.
- [ ] Allowed `content_type` values and file extensions are validated server-side; the client-side check is supplementary.
- [ ] Presigned PUT URLs expire in 15 minutes, limiting the window for misuse if intercepted.
- [ ] `STORAGE_ACCESS_KEY_ID` and `STORAGE_SECRET_ACCESS_KEY` are loaded from environment variables and are never included in any API response, log line, or error message returned to clients.
- [ ] The R2/S3 bucket is private. Public file access is only via the CDN (`STORAGE_PUBLIC_URL`), not via direct R2 presigned GET URLs or bucket-level public access.
- [ ] Error responses do not expose the bucket name, account ID, or raw SDK error messages.
- [ ] File keys are constructed server-side using a server-generated ULID; clients cannot specify an arbitrary storage path.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Service | `lib/services/upload_service.dart` | Create — full presigned URL upload flow |
| Flutter — Model | `lib/models/upload_result.dart` | Create — UploadResult value object |
| Flutter — Config | `pubspec.yaml` | Modify — add `http` package if not present |
| Go — Handler | `internal/handler/media_handler.go` | Create — POST /api/v1/media/upload-url and DELETE /api/v1/media |
| Go — Service | `internal/service/media_service.go` | Create — GenerateUploadURL, DeleteFile |
| Go — DTO | `internal/dto/media_dto.go` | Create — UploadURLRequest, UploadURLResponse, DeleteMediaRequest |
| Go — Package | `pkg/storage/storage.go` | Create — StorageClient interface and R2/S3 implementation |
| Go — Config | `internal/config/config.go` | Modify — add all STORAGE_* fields to the Config struct |
| Go — Router | `router/router.go` | Modify — register media routes under the JWT-protected group |
| Config | `.env.example` | Modify — add all STORAGE_* variables with placeholder values |

---

## 11. Acceptance Criteria

- [ ] `POST /api/v1/media/upload-url` with a valid JPEG request returns `200` containing a non-empty `upload_url`, `file_key`, and `public_url`.
- [ ] A raw HTTP PUT to the returned `upload_url` successfully stores the file in R2 and the file is accessible at `public_url`.
- [ ] `POST /api/v1/media/upload-url` with an unsupported `content_type` returns `422 validation_error`.
- [ ] `POST /api/v1/media/upload-url` with a mismatched extension and content type returns `422 validation_error`.
- [ ] `POST /api/v1/media/upload-url` without a valid JWT returns `401 unauthorized`.
- [ ] `DELETE /api/v1/media` with a valid own file key returns `204` and the file is removed from R2.
- [ ] `DELETE /api/v1/media` with another user's file key returns `403 forbidden`.
- [ ] `DELETE /api/v1/media` with a non-existent file key returns `404 not_found`.
- [ ] `UploadService.uploadFile()` rejects files over the folder-specific size limit before any API call is made.
- [ ] `UploadService.uploadFile()` rejects unsupported MIME types before any API call is made.
- [ ] Upload in-progress state shows `CircularProgressIndicator` on the triggering control; loading is always cleared in a finally block.
- [ ] All `STORAGE_*` environment variables are present in `.env.example` with placeholder values.
- [ ] Storage credentials are never present in any API response body or server log at INFO level or below.
- [ ] All API endpoints return the standard error envelope for non-2xx responses.
