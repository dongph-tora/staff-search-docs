# [FEED-01] Create Post (Image / Video, Caption)

| Field | Value |
|---|---|
| **Type** | Feature |
| **Priority** | P1 MVP |
| **Estimate** | 2 days manual / 0.75 days vibe coding |
| **Dependencies** | DB-01 — `posts` table defined in Section 6.3. DB-05 — `UploadService`, presigned URL API, `post` upload folder (100 MB, image/jpeg, image/png, image/webp, video/mp4) defined in Sections 4.3 and 5.2. AUTH-01 — `ApiClient` and `AuthProvider` in Sections 4.1–4.4. |
| **Platforms** | iOS · Android · Web |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format (Section 2), JWT middleware (Section 3), GORM conventions (Section 4), `ApiClient` (Section 6), error display (Section 10). |

---

## 1. Overview

This ticket enables any authenticated user or staff to create a post consisting of an optional image or video file and an optional caption. Posts are the foundation of the TikTok-style vertical feed (FEED-02). After creating a post, the author is redirected to the feed where the new post is visible. All media uploads use the presigned URL flow from DB-05. Both image and video posts are supported; the `media_type` field distinguishes them and drives the correct player in FEED-02.

### End-to-end flow

    [Flutter App]                   [Go API]                   [R2 Storage]
         │                               │                           │
         │── UploadService.uploadFile ──►│ POST /api/v1/media/       │
         │   (folder=post)               │       upload-url          │
         │◄── {upload_url, file_key,     │                           │
         │     public_url}               │                           │
         │── PUT <upload_url> ──────────────────────────────────────►│
         │◄── 200 OK ────────────────────────────────────────────────│
         │                               │                           │
         │── POST /api/v1/posts ────────►│                           │
         │   {content, media_url,        │── INSERT posts ──────────►│
         │    media_type}                │◄── row created ───────────│
         │◄── 201 {post} ────────────────│                           │
         │── navigate to feed            │                           │

---

## 2. User Flows

### 2.1 Happy Path — Create an Image Post

1. Authenticated user taps the "Create Post" button (+ icon in navigation bar).
2. App navigates to `CreatePostScreen`.
3. User taps the media area and selects an image from the gallery.
4. `UploadService` validates the file (≤ 100 MB; JPEG/PNG/WebP).
5. A preview thumbnail is shown in the media area.
6. User enters a caption (optional).
7. User taps "Share".
8. Upload begins: `UploadService.uploadFile(file, UploadFolder.post)` → R2 PUT.
9. On R2 success, `POST /api/v1/posts` is called with `media_url`, `media_type = "image"`, and `content`.
10. API returns `201` with the new post.
11. Green SnackBar: `"Post shared successfully."` App navigates back to the home feed (FEED-02).

### 2.2 Happy Path — Create a Video Post

1–3. Same as 2.1, but user selects a video file.
4. `UploadService` validates: ≤ 100 MB, video/mp4. If a `.mov` file is selected, the app shows a SnackBar: `"Unsupported file type. Please select a JPEG, PNG, WebP, or MP4 file."` (MOV is not accepted for posts per DB-05 post folder limits).
5. A video thumbnail is generated client-side and shown as preview.
6–11. Same as 2.1, with `media_type = "video"`.

### 2.3 Happy Path — Create a Text-Only Post

1. User navigates to `CreatePostScreen`.
2. User skips media selection and types a caption.
3. User taps "Share".
4. `POST /api/v1/posts` is called with only `content`; `media_url` and `media_type` are omitted.
5. API returns `201`. App navigates to feed.

### 2.4 Error Cases

| Scenario | Behaviour |
|---|---|
| Both caption and media are empty | Share button validation blocks submission. Inline message: `"Add a photo, video, or caption to share."` |
| File exceeds 100 MB | Red SnackBar: `"File is too large. Maximum size is 100 MB."` No API call. |
| Unsupported file type (GIF, MOV) | Red SnackBar: `"Unsupported file type. Please select a JPEG, PNG, WebP, or MP4 file."` |
| R2 upload fails | Red SnackBar: `"Upload failed. Check your connection and try again."` |
| 500 from POST /posts | Red SnackBar: `"Something went wrong. Please try again later."` |
| Network error | Red SnackBar: `"Unable to connect. Check your network and retry."` |

---

## 3. UI / Screen

### 3.1 Create Post Screen (`create_post_screen.dart`)

> Current location: `lib/screens/feed/create_post_screen.dart`
> Status: **Create new**

    ┌─────────────────────────────────────────┐
    │  ×  New Post               [Share]      │
    ├─────────────────────────────────────────┤
    │                                         │
    │  ┌─────────────────────────────────┐   │
    │  │                                 │   │
    │  │   [Tap to add photo or video]   │   │
    │  │   📷  🎥                        │   │
    │  │   (preview fills this area)     │   │
    │  │                                 │   │
    │  └─────────────────────────────────┘   │
    │                                         │
    │  [Write a caption...]                   │
    │  (multi-line text field, max 500 chars) │
    │                                         │
    │  Progress bar (shown during upload)     │
    └─────────────────────────────────────────┘

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| Media picker area | `GestureDetector` wrapping `Container` | Tapping shows bottom sheet with "Photo Library" and "Video" options |
| Media preview | `Image.file` or `VideoPlayerWidget` | Replaces picker area once media is selected; tap to change |
| Caption field | `TextFormField` | Optional; multiline; max 500 characters; character counter shown |
| Share button | `TextButton` in AppBar | Shows `CircularProgressIndicator` (20×20, strokeWidth 2) while upload + API call are in progress; disabled during request |
| Cancel button | `IconButton` with `Icons.close` | Pops the screen without saving |
| Upload progress bar | `LinearProgressIndicator` | Shown below caption field during R2 PUT; hidden otherwise |

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/screens/feed/create_post_screen.dart` | Post creation form with media picker, preview, and caption input |
| `lib/models/post.dart` | `Post` model with `id`, `authorId`, `content`, `mediaUrl`, `mediaType`, `likesCount`, `commentsCount`, `isActive`, `createdAt` |
| `lib/services/post_service.dart` | API calls: `createPost`, `getFeed`, `getPost` |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/main.dart` | Add named route for `CreatePostScreen` |
| `pubspec.yaml` | Add `video_player: ^2.9.2` and `image_picker: ^1.1.2` if not already present; add `video_thumbnail` if needed for video preview generation |

### 4.3 PostService Responsibilities

`PostService` wraps all post-related API calls:

- `createPost(CreatePostRequest req)` — Calls `POST /api/v1/posts`. Returns `Post` on `201`.
- `getFeed(String? cursor, int limit, String? categoryFilter)` — Calls `GET /api/v1/posts/feed` with cursor pagination. Returns `FeedResponse` containing `posts` array and `nextCursor`.
- `getPost(String postID)` — Calls `GET /api/v1/posts/:postID`. Returns `Post` or null if 404.

### 4.4 Form Validation Rules

| Field | Required | Rule | Error Message |
|---|---|---|---|
| At least one of `mediaUrl` or `content` | Yes | Both cannot be empty | `"Add a photo, video, or caption to share."` |
| `content` | No | Max 500 characters | `"Caption must be 500 characters or less."` |
| Media file | No | Max 100 MB; JPEG/PNG/WebP/MP4 only | See error cases in Section 2.4 |

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/posts` | Create a new post | Yes |
| `GET` | `/api/v1/posts/feed` | Get paginated feed posts | Yes |
| `GET` | `/api/v1/posts/:postID` | Get a single post by ID | Yes |

### 5.2 POST /api/v1/posts

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `content` | string | No | Max 500 characters |
| `media_url` | string | No | Must begin with `STORAGE_PUBLIC_URL` if provided |
| `media_type` | string | No (required if `media_url` provided) | Must be `"image"` or `"video"` if provided |

**Response `201`:**

| Field | Type | Description |
|---|---|---|
| `id` | string | ULID |
| `author_id` | string | The creating user's ID |
| `author` | object | `{id, name, avatar_url}` — the author's public info |
| `content` | string\|null | Caption text |
| `media_url` | string\|null | R2 CDN URL |
| `media_type` | string\|null | "image" or "video" |
| `likes_count` | int | 0 initially |
| `comments_count` | int | 0 initially |
| `is_liked` | bool | false for the creating user |
| `created_at` | string | ISO 8601 timestamp |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed body; both `content` and `media_url` are absent; `media_url` provided without `media_type` |
| `401` | `unauthorized` | Missing or invalid JWT |
| `422` | `validation_error` | `media_type` is not `"image"` or `"video"`; `content` exceeds 500 chars; `media_url` does not start with `STORAGE_PUBLIC_URL` |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Parse request body into `CreatePostRequest` DTO. Return `400` if parsing fails.
3. Validate that at least one of `content` or `media_url` is provided. Return `400` if both are absent.
4. If `media_url` is provided, validate that `media_type` is also provided and is `"image"` or `"video"`. Return `422` on validation failure.
5. Validate `content` length ≤ 500. Return `422` if exceeded.
6. Call `PostService.CreatePost(ctx, userID, req)`. Return `500` for errors.
7. Return `201` with `PostResponse`.

### 5.3 GET /api/v1/posts/feed

**Query parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `cursor` | string | No | ULID of the last post seen; used for cursor pagination |
| `limit` | int | No | Number of posts to return; default 20, max 50 |
| `category` | string | No | Job category key to filter by staff category |

**Response `200`:**

| Field | Type | Description |
|---|---|---|
| `posts` | array | Array of `PostResponse` objects |
| `next_cursor` | string\|null | ULID to use as `cursor` in the next request; null when no more posts |
| `has_more` | bool | True if more posts exist beyond this page |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `401` | `unauthorized` | Missing or invalid JWT |
| `422` | `validation_error` | `limit` exceeds 50 |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Parse `cursor`, `limit`, and `category` from query params. Default `limit` to 20 if absent; cap at 50.
3. Call `PostService.GetFeed(ctx, userID, cursor, limit, category)`.
4. Return `200` with `FeedResponse`.

### 5.4 GET /api/v1/posts/:postID

**Response `200`:** Single `PostResponse` object.

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `401` | `unauthorized` | Missing or invalid JWT |
| `404` | `not_found` | Post does not exist or `is_active = false` |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Call `PostService.GetByID(ctx, c.Params("postID"))`. Return `404` if `ErrNotFound`.
2. Return `200` with `PostResponse`.

### 5.5 Service Responsibilities

`PostService`:

- `CreatePost(ctx, userID string, req CreatePostRequest) (Post, error)` — Generates a new ULID. Inserts a `posts` row with `author_id = userID`, `content`, `media_url`, `media_type`. Returns the created row with author info preloaded.
- `GetFeed(ctx context.Context, userID string, cursor string, limit int, category string) ([]Post, string, bool, error)` — Queries `posts` joined with `users` and `staff_profiles` (when `category` filter is present). Uses `WHERE posts.id < cursor` for cursor pagination ordered by `id DESC` (ULIDs are monotonically increasing). Checks if a next page exists. Returns posts, nextCursor, hasMore.
- `GetByID(ctx, postID string) (Post, error)` — Finds post by ID where `is_active = true`. Preloads `author`. Returns `model.ErrNotFound` if absent.

---

## 6. Database

Not applicable for this ticket. The `posts` table is defined in DB-01 Section 6.3. The `Post` GORM model is created in `internal/model/post.go` and registered in `main.go`'s `db.AutoMigrate()` call.

---

## 7. Configuration

Not applicable for this ticket. No new environment variables are required.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Valid POST — image with caption | `201`, post returned with `media_type = "image"` |
| 2 | Valid POST — video, no caption | `201`, post returned with `media_type = "video"`, `content = null` |
| 3 | Valid POST — caption only, no media | `201`, `media_url = null`, `media_type = null` |
| 4 | POST — both `content` and `media_url` absent | `400 bad_request` |
| 5 | POST — `media_url` provided without `media_type` | `400 bad_request` |
| 6 | POST — `media_type = "gif"` | `422 validation_error` |
| 7 | POST — `content` of 501 characters | `422 validation_error` |
| 8 | POST — no JWT | `401 unauthorized` |
| 9 | GET /posts/feed — first page | `200`, up to 20 posts, `has_more = true/false` |
| 10 | GET /posts/feed — with `cursor` | `200`, posts older than cursor returned |
| 11 | GET /posts/feed — `limit = 51` | `422 validation_error` |
| 12 | GET /posts/:postID — valid ID | `200`, post returned |
| 13 | GET /posts/:postID — non-existent | `404 not_found` |
| 14 | GET /posts/:postID — `is_active = false` | `404 not_found` |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Select image and add caption; tap Share | Upload shown; on success, green SnackBar + navigate to feed |
| 2 | Select video and tap Share | Upload shown; video preview generated; on success, navigate to feed |
| 3 | Tap Share with no media and no caption | Inline message: `"Add a photo, video, or caption to share."` No API call |
| 4 | Select file exceeding 100 MB | Red SnackBar: `"File is too large. Maximum size is 100 MB."` |
| 5 | R2 upload fails | Red SnackBar: `"Upload failed. Check your connection and try again."` |
| 6 | Network error | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| 7 | Caption exceeds 500 chars | Character counter turns red; Share button disabled |
| 8 | Share button tapped | `CircularProgressIndicator` (20×20 px, strokeWidth 2) shown in AppBar; button disabled |

---

## 9. Security Checklist

- [ ] `author_id` is read from JWT claims (`c.Locals("userID")`), never from the request body; users cannot create posts on behalf of others.
- [ ] `media_url` is validated server-side to start with `STORAGE_PUBLIC_URL`; users cannot reference external arbitrary URLs as media.
- [ ] `media_type` is validated against an explicit allowlist (`"image"`, `"video"`).
- [ ] Caption length is validated server-side, not only client-side.
- [ ] Input validated server-side, not only client-side.
- [ ] No sensitive data returned in error messages.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Screen | `lib/screens/feed/create_post_screen.dart` | Create — post creation form |
| Flutter — Model | `lib/models/post.dart` | Create — Post, FeedResponse models |
| Flutter — Service | `lib/services/post_service.dart` | Create — createPost, getFeed, getPost |
| Flutter — App | `lib/main.dart` | Modify — add CreatePostScreen route |
| Flutter — Config | `pubspec.yaml` | Modify — add video_player, image_picker if absent |
| Go — Model | `internal/model/post.go` | Create — Post, Comment, Like, SavedPost GORM models |
| Go — Handler | `internal/handler/post_handler.go` | Create — CreatePost, GetFeed, GetPostByID handlers |
| Go — Service | `internal/service/post_service.go` | Create — CreatePost, GetFeed, GetByID |
| Go — Repository | `internal/repository/post_repository.go` | Modify — add Create, GetFeed, GetByID methods to scaffolded file from DB-01 |
| Go — DTO | `internal/dto/post_dto.go` | Create — CreatePostRequest, PostResponse, FeedResponse |
| Go — Router | `router/router.go` | Modify — register /api/v1/posts routes |
| Go — main | `main.go` | Modify — register Post model in AutoMigrate call |

---

## 11. Acceptance Criteria

- [ ] `POST /api/v1/posts` with a valid image `media_url` and caption returns `201` with `media_type = "image"`.
- [ ] `POST /api/v1/posts` with only a caption (no media) returns `201` with `media_url = null`.
- [ ] `POST /api/v1/posts` with both `content` and `media_url` absent returns `400 bad_request`.
- [ ] `POST /api/v1/posts` without a valid JWT returns `401 unauthorized`.
- [ ] `GET /api/v1/posts/feed` returns up to 20 posts by default, sorted newest-first, with `next_cursor` for pagination.
- [ ] `GET /api/v1/posts/feed` with `cursor` returns posts older than the cursor post.
- [ ] `GET /api/v1/posts/:postID` returns `404` for a non-existent or inactive post.
- [ ] `CreatePostScreen` validates that at least one of media or caption is present before making any API call.
- [ ] Selecting a file larger than 100 MB shows red SnackBar without making any API call.
- [ ] Share button shows `CircularProgressIndicator` (20×20 px, strokeWidth 2) during upload and is disabled until complete.
- [ ] After successful post creation, the user is navigated to the home feed.
- [ ] All API endpoints return the standard error envelope for non-2xx responses.
