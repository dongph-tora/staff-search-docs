# [STAFF-05] Portfolio Photo Upload

| Field | Value |
|---|---|
| **Type** | Feature |
| **Priority** | P1 MVP |
| **Estimate** | 1 day manual / 0.5 days vibe coding |
| **Dependencies** | STAFF-01 — `StaffProfile` model, `staff_portfolio_photos` table, `StaffService`, `StaffProvider` created in Sections 4–5. DB-05 — `UploadService`, presigned URL API, `portfolio` upload folder and 10 MB JPEG/PNG/WebP limit defined in Sections 4.3 and 5.2. |
| **Platforms** | iOS · Android · Web |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format (Section 2), JWT middleware (Section 3), `ApiClient` (Section 6), error display (Section 10). |

---

## 1. Overview

This ticket enables staff users to upload, reorder, and delete portfolio photos displayed on their staff profile. Portfolio photos are stored in Cloudflare R2 via the presigned URL flow from DB-05, and their metadata (URL, display order) is stored in `staff_portfolio_photos`. A staff profile can have a maximum of 12 portfolio photos. The portfolio gallery is visible to all users on the staff profile screen (STAFF-07).

### End-to-end flow — Add Portfolio Photo

    [Flutter App]                   [Go API]                   [R2 Storage]
         │                               │                           │
         │── UploadService.uploadFile ──►│ POST /api/v1/media/       │
         │   (folder=portfolio)          │       upload-url          │
         │◄── {upload_url, file_key,     │                           │
         │     public_url}               │                           │
         │── PUT <upload_url> ──────────────────────────────────────►│
         │◄── 200 OK ────────────────────────────────────────────────│
         │                               │                           │
         │── POST /api/v1/staff/─────── ►│                           │
         │       portfolio/photos        │── INSERT                  │
         │   {photo_url, display_order}  │   staff_portfolio_photos ►│
         │◄── 201 {portfolio_photo} ─────│                           │

### End-to-end flow — Delete Portfolio Photo

    [Flutter App]                   [Go API]                   [R2 Storage]
         │                               │                           │
         │── DELETE /api/v1/staff/──────►│                           │
         │       portfolio/photos/:id    │── DELETE staff_portfolio  │
         │                               │   _photos row ───────────►│
         │                               │── DELETE /api/v1/media    │
         │                               │   {file_key} ─────────────│
         │◄── 204 No Content ────────────│                           │

---

## 2. User Flows

### 2.1 Happy Path — Upload a Portfolio Photo

1. Staff user opens their own profile screen and taps "Edit Portfolio".
2. App navigates to `PortfolioEditScreen`.
3. User taps the "Add Photo" tile (visible when fewer than 12 photos are present).
4. Image picker launches. User selects a JPEG, PNG, or WebP image.
5. `UploadService` validates file size (≤ 10 MB) and MIME type. If invalid, a red SnackBar is shown.
6. Upload flow begins: `UploadService.uploadFile(file, UploadFolder.portfolio)` obtains a presigned URL and PUTs the file to R2.
7. On R2 success, app calls `POST /api/v1/staff/portfolio/photos` with `photo_url` and `display_order`.
8. API creates the `staff_portfolio_photos` row and returns `201`.
9. The new photo appears in the grid. Green SnackBar: `"Photo added to portfolio."`.

### 2.2 Happy Path — Delete a Portfolio Photo

1. On `PortfolioEditScreen`, staff user long-presses (or taps a delete icon on) a photo tile.
2. Confirmation dialog: `"Remove this photo from your portfolio?"` with "Remove" and "Cancel" buttons.
3. User taps "Remove".
4. App calls `DELETE /api/v1/staff/portfolio/photos/:photoID`.
5. API deletes the `staff_portfolio_photos` row and the R2 object.
6. Photo is removed from the grid. Green SnackBar: `"Photo removed."`.

### 2.3 Happy Path — Reorder Portfolio Photos

1. On `PortfolioEditScreen`, staff user drags a photo to a new position.
2. After drag completes, app calls `PATCH /api/v1/staff/portfolio/photos/reorder` with the new ordered array of photo IDs.
3. API updates `display_order` for each photo in a single transaction.
4. Grid reflects the new order.

### 2.4 Error Cases

| Scenario | Behaviour |
|---|---|
| Selected file exceeds 10 MB | Red SnackBar: `"File is too large. Maximum size is 10 MB."` No API call. |
| Unsupported file type (e.g. GIF) | Red SnackBar: `"Unsupported file type. Please select a JPEG, PNG, or WebP image."` No API call. |
| Staff already has 12 photos | "Add Photo" tile is hidden; add action is blocked. Red SnackBar: `"Portfolio is full. Maximum 12 photos allowed."` |
| DELETE on a photo belonging to another staff | `403 forbidden`. Red SnackBar: `"You do not have permission to delete this photo."` |
| R2 upload fails (network error) | Red SnackBar: `"Upload failed. Check your connection and try again."` |
| Network error on any API call | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| Server error (500) | Red SnackBar: `"Something went wrong. Please try again later."` |

---

## 3. UI / Screen

### 3.1 Portfolio Edit Screen (`portfolio_edit_screen.dart`)

> Current location: `lib/screens/staff/portfolio_edit_screen.dart`
> Status: **Create new**

    ┌─────────────────────────────────────────┐
    │  ← Portfolio (3/12)                    │
    │                                         │
    │  ┌───────┐ ┌───────┐ ┌───────┐         │
    │  │ photo │ │ photo │ │ photo │         │
    │  │  [x]  │ │  [x]  │ │  [x]  │         │
    │  └───────┘ └───────┘ └───────┘         │
    │  ┌───────┐ ┌───────┐ ┌───────┐         │
    │  │  [+]  │ │       │ │       │         │
    │  └───────┘ └───────┘ └───────┘         │
    │  (drag to reorder)                      │
    └─────────────────────────────────────────┘

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| Photo grid | `ReorderableGridView` or `GridView` with `LongPressDraggable` | 3-column grid; draggable to reorder; shows delete icon overlay |
| Add Photo tile | `InkWell` with `+` icon | Visible only when fewer than 12 photos; hidden when full; taps launch `ImagePicker` |
| Delete icon overlay | `IconButton` with `Icons.close` | Visible on each tile; taps show confirmation dialog |
| Upload progress | `CircularProgressIndicator` (20×20 px, strokeWidth 2) | Overlays the new photo tile while upload is in progress |
| Photo count label | Text in AppBar | Shows current count and max: "(3/12)" |

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/screens/staff/portfolio_edit_screen.dart` | Portfolio management screen — grid view, add, delete, reorder |
| `lib/models/portfolio_photo.dart` | `PortfolioPhoto` model with `id`, `photoUrl`, `displayOrder` |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/services/staff_service.dart` | Add `addPortfolioPhoto`, `deletePortfolioPhoto`, `reorderPortfolioPhotos` methods |
| `lib/providers/staff_provider.dart` | Add `portfolioPhotos` list to state; update on add/delete/reorder |
| `lib/screens/staff/staff_profile_screen.dart` | Add "Edit Portfolio" button visible only to the profile owner; navigate to `PortfolioEditScreen` |
| `pubspec.yaml` | Add `image_picker: ^1.1.2` if not already present |

### 4.3 PortfolioEditScreen Responsibilities

`PortfolioEditScreen` reads `StaffProvider.currentProfile.portfolioPhotos` on load to render the initial grid. It calls `StaffService.addPortfolioPhoto` after a successful R2 upload, and `StaffService.deletePortfolioPhoto` after user confirmation. On drag-drop completion, it calls `StaffService.reorderPortfolioPhotos` with the new ID order. All operations update `StaffProvider` state so the grid reflects changes immediately (optimistic update with rollback on error).

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/staff/portfolio/photos` | Add a portfolio photo to the authenticated staff's profile | Yes |
| `DELETE` | `/api/v1/staff/portfolio/photos/:photoID` | Delete a portfolio photo by ID | Yes |
| `PATCH` | `/api/v1/staff/portfolio/photos/reorder` | Update display_order for all portfolio photos | Yes |

### 5.2 POST /api/v1/staff/portfolio/photos

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `photo_url` | string | Yes | Non-empty URL; must begin with `STORAGE_PUBLIC_URL` value |
| `display_order` | int | No | Non-negative integer; defaults to current max + 1 if omitted |

**Response `201`:**

| Field | Type | Description |
|---|---|---|
| `id` | string | ULID of the new `staff_portfolio_photos` row |
| `staff_profile_id` | string | The staff's profile ID |
| `photo_url` | string | R2 CDN URL |
| `display_order` | int | Final display order assigned |
| `created_at` | string | ISO 8601 timestamp |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed body or missing `photo_url` |
| `401` | `unauthorized` | Missing or invalid JWT |
| `404` | `not_found` | Authenticated user has no staff profile |
| `422` | `validation_error` | Staff already has 12 photos; `photo_url` does not start with `STORAGE_PUBLIC_URL` |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Parse request body. Return `400` if parsing fails.
3. Call `StaffPortfolioService.AddPhoto(ctx, userID, req)`. Return `404` if staff profile not found. Return `422` if photo limit reached. Return `500` for other errors.
4. Return `201` with `PortfolioPhotoResponse`.

### 5.3 DELETE /api/v1/staff/portfolio/photos/:photoID

**Response `204`:** No body.

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `401` | `unauthorized` | Missing or invalid JWT |
| `403` | `forbidden` | The photo does not belong to the authenticated user's staff profile |
| `404` | `not_found` | Photo with given ID does not exist |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Call `StaffPortfolioService.DeletePhoto(ctx, userID, photoID)`. Return `403` if ownership check fails. Return `404` if not found. Return `500` for other errors.
3. Return `204`.

### 5.4 PATCH /api/v1/staff/portfolio/photos/reorder

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `photo_ids` | array of strings | Yes | Non-empty array; all IDs must belong to the authenticated user's staff profile |

**Response `200`:**

Returns the full updated list of `PortfolioPhotoResponse` objects sorted by `display_order`.

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed body or empty `photo_ids` |
| `401` | `unauthorized` | Missing or invalid JWT |
| `403` | `forbidden` | Any photo ID in the array does not belong to the authenticated user's staff profile |
| `404` | `not_found` | Authenticated user has no staff profile |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")`.
2. Parse request body. Return `400` if `photo_ids` is empty.
3. Call `StaffPortfolioService.ReorderPhotos(ctx, userID, photoIDs)`. Returns `403` if any photo ID does not belong to the user's profile. Return `500` for other errors.
4. Return `200` with updated photo list.

### 5.5 Service Responsibilities

`StaffPortfolioService`:

- `AddPhoto(ctx, userID string, req AddPortfolioPhotoRequest) (PortfolioPhoto, error)` — Looks up the staff profile by userID (ErrNotFound if absent). Counts existing photos; returns `ErrPhotoLimitExceeded` if count ≥ 12. Validates `photo_url` prefix. Inserts a new `staff_portfolio_photos` row with the given `display_order` (or `maxOrder + 1` if not provided).
- `DeletePhoto(ctx, userID string, photoID string) error` — Loads the photo by ID; verifies its `staff_profile_id` matches the user's profile (ErrForbidden if not). Deletes the DB row. Calls `MediaService.DeleteFile(ctx, fileKey)` to clean up R2 (fire-and-forget — R2 errors are logged but do not fail the response).
- `ReorderPhotos(ctx, userID string, photoIDs []string) ([]PortfolioPhoto, error)` — Verifies all photoIDs belong to the user's profile in one query (ErrForbidden if any are foreign). Updates `display_order` for each ID matching its index in the slice. Uses a transaction.

---

## 6. Database

Not applicable for this ticket. The `staff_portfolio_photos` table is defined in DB-01 Section 6.2. No migration changes are required.

---

## 7. Configuration

Not applicable for this ticket. No new environment variables are required beyond those defined in DB-05 Section 7.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Valid POST — first photo | `201`, row created, `display_order = 0` |
| 2 | Valid POST — second photo | `201`, `display_order = 1` |
| 3 | POST — staff already has 12 photos | `422 validation_error` |
| 4 | POST — no staff profile for this user | `404 not_found` |
| 5 | POST — no JWT | `401 unauthorized` |
| 6 | DELETE — own photo | `204`, DB row deleted |
| 7 | DELETE — another user's photo | `403 forbidden` |
| 8 | DELETE — non-existent photo ID | `404 not_found` |
| 9 | PATCH reorder — valid own photo IDs | `200`, display_order updated in correct sequence |
| 10 | PATCH reorder — contains foreign photo ID | `403 forbidden` |
| 11 | PATCH reorder — empty array | `400 bad_request` |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Add photo — valid JPEG under 10 MB | Photo appears in grid; green SnackBar: `"Photo added to portfolio."` |
| 2 | Add photo — file exceeds 10 MB | Red SnackBar: `"File is too large. Maximum size is 10 MB."` No API call. |
| 3 | Add photo when grid is full (12 photos) | "Add Photo" tile hidden; add action blocked |
| 4 | Delete photo — user confirms | Photo removed from grid; green SnackBar: `"Photo removed."` |
| 5 | Delete photo — user cancels dialog | No action taken; grid unchanged |
| 6 | Upload in progress | `CircularProgressIndicator` shown on the new photo tile; "Add Photo" disabled |
| 7 | Network error during upload | Red SnackBar: `"Upload failed. Check your connection and try again."` |
| 8 | Drag to reorder | Grid reflects new order; reorder API called after drop |

---

## 9. Security Checklist

- [ ] Only the authenticated staff user can add or delete photos from their own portfolio; `userID` is read from JWT, never from the request body.
- [ ] The DELETE endpoint verifies that the photo's `staff_profile_id` matches the authenticated user's profile before deleting.
- [ ] The PATCH reorder endpoint verifies all provided photo IDs belong to the authenticated user's profile before updating.
- [ ] `photo_url` is validated to begin with `STORAGE_PUBLIC_URL` to prevent storage of arbitrary external URLs.
- [ ] Photo count limit (12) is enforced server-side; clients cannot bypass it.
- [ ] Input validated server-side, not only client-side.
- [ ] No sensitive data returned in error messages.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Model | `lib/models/portfolio_photo.dart` | Create — PortfolioPhoto model |
| Flutter — Screen | `lib/screens/staff/portfolio_edit_screen.dart` | Create — portfolio grid with add, delete, reorder |
| Flutter — Service | `lib/services/staff_service.dart` | Modify — add addPortfolioPhoto, deletePortfolioPhoto, reorderPortfolioPhotos |
| Flutter — Provider | `lib/providers/staff_provider.dart` | Modify — add portfolioPhotos state; update on mutations |
| Flutter — Screen | `lib/screens/staff/staff_profile_screen.dart` | Modify — add "Edit Portfolio" button for profile owner |
| Flutter — Config | `pubspec.yaml` | Modify — add image_picker if not already present |
| Go — Handler | `internal/handler/staff_handler.go` | Modify — add AddPortfolioPhoto, DeletePortfolioPhoto, ReorderPortfolioPhotos handlers |
| Go — Service | `internal/service/staff_portfolio_service.go` | Create — AddPhoto, DeletePhoto, ReorderPhotos |
| Go — Repository | `internal/repository/staff_repository.go` | Modify — add portfolio photo CRUD and reorder methods |
| Go — DTO | `internal/dto/staff_dto.go` | Modify — add AddPortfolioPhotoRequest, PortfolioPhotoResponse, ReorderRequest |
| Go — Router | `router/router.go` | Modify — register /api/v1/staff/portfolio/photos routes |

---

## 11. Acceptance Criteria

- [ ] `POST /api/v1/staff/portfolio/photos` with a valid `photo_url` creates a `staff_portfolio_photos` row and returns `201`.
- [ ] `POST /api/v1/staff/portfolio/photos` when the staff already has 12 photos returns `422 validation_error`.
- [ ] `DELETE /api/v1/staff/portfolio/photos/:photoID` deletes the DB row and returns `204`.
- [ ] `DELETE /api/v1/staff/portfolio/photos/:photoID` for a photo belonging to another user returns `403 forbidden`.
- [ ] `PATCH /api/v1/staff/portfolio/photos/reorder` updates `display_order` for all photos in the provided array order.
- [ ] Uploading a file larger than 10 MB shows red SnackBar `"File is too large. Maximum size is 10 MB."` without making any API call.
- [ ] `PortfolioEditScreen` displays all current portfolio photos in a 3-column grid.
- [ ] The "Add Photo" tile is hidden when the staff already has 12 photos.
- [ ] Upload progress shows `CircularProgressIndicator` (20×20 px, strokeWidth 2) while upload is in progress.
- [ ] Deleting a photo requires user confirmation via a dialog before proceeding.
- [ ] All API endpoints return the standard error envelope for non-2xx responses.
- [ ] All endpoints return `401` when called without a valid JWT.
