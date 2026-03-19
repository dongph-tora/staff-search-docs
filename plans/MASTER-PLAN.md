# Master Implementation Plan — staffsearch

> **Quy tắc:** Sau khi hoàn thành mỗi plan, đánh dấu `[x]` vào checkbox tương ứng và fill vào ngày hoàn thành.
> Không được bắt đầu một plan nếu dependency của nó chưa được checked ✅.

---

## Tiến độ tổng quan

```
Completed:  12 / 12
Remaining:  0 / 12
```

---

## Phase 1 — Foundation

### ✅ [1] DB-01 — Database Schema

**Plan:** [DB-01-plan.md](./DB-01-plan.md)
**Depends on:** Không có
**Completed:** 2026-03-11

- [x] **[DB-01]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] App khởi động không bị lỗi `db.AutoMigrate()`
  - [x] Tất cả 10+ tables đã tồn tại trong PostgreSQL (kiểm tra bằng `\dt` trong psql)
  - [x] `staff_profiles` có column `staff_number VARCHAR(6) UNIQUE NOT NULL`
  - [x] `posts` có column `media_type VARCHAR(10)`
  - [x] `likes` có UNIQUE constraint trên `(user_id, post_id)`
  - [x] `reviews` có CHECK constraint `rating >= 1 AND rating <= 5`
  - [x] Down migrations chạy được mà không có lỗi
  - [x] Tất cả repository scaffold files đã được tạo

---

### ✅ [2] AUTH-01 — Email / Password Auth (Flutter migration)

**Plan:** [AUTH-01-plan.md](./AUTH-01-plan.md)
**Depends on:** [1] DB-01
**Completed:** 2026-03-11

- [x] **[AUTH-01]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `ApiClient` singleton tồn tại tại `lib/services/api_client.dart`, auto-refresh token khi 401
  - [x] `AuthProvider` tồn tại tại `lib/providers/auth_provider.dart`, đăng ký trong `MultiProvider`
  - [x] Tokens được lưu trong `FlutterSecureStorage`, KHÔNG phải `SharedPreferences`
  - [x] `LoginScreen` không còn import `LocalAuthService`
  - [x] `RegisterScreen` không còn import `LocalAuthService`
  - [x] Login thành công → navigate `/home`, `AuthProvider.isLoggedIn = true`
  - [x] Login sai password → hiển thị `"Invalid email or password."`
  - [x] Register với email đã tồn tại → `"An account with this email already exists."`
  - [x] App restart → `restoreSession()` tự login nếu token còn hợp lệ
  - [x] `POST /api/v1/auth/password-reset/request` → luôn trả 200
  - [x] `POST /api/v1/auth/password-reset/confirm` với expired token → 400

---

## Phase 2 — Infrastructure (làm song song)

### ✅ [3] DB-02 — Migrate LocalStorage to DB

**Plan:** [DB-02-plan.md](./DB-02-plan.md)
**Depends on:** [2] AUTH-01
**Completed:** 2026-03-11

- [x] **[DB-02]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `PATCH /api/v1/users/me` với `{name, bio, phone}` hợp lệ → 200, DB updated
  - [x] `PATCH /api/v1/users/me` với `name > 100 chars` → 422
  - [x] `PATCH /api/v1/users/me` không có JWT → 401
  - [x] `ProfileEditScreen` pre-fill từ `AuthProvider.currentUser`, không từ SharedPreferences
  - [x] Save thành công → green SnackBar `"Profile updated successfully."`
  - [x] `AppModeService` không còn đọc SharedPreferences cho profile/login state
  - [x] `role`, `points`, `password_hash` không thể thay đổi qua endpoint này
  - [x] `lib/utils/storage_helper.dart` có deprecation comment ở đầu file

---

### ✅ [4] DB-05 — Storage Setup for Media

**Plan:** [DB-05-plan.md](./DB-05-plan.md)
**Depends on:** [2] AUTH-01
**Completed:** 2026-03-11

- [x] **[DB-05]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `POST /api/v1/media/upload-url` với JPEG + folder=avatars → 200 với `upload_url`, `file_key`, `public_url`
  - [x] Raw HTTP PUT đến `upload_url` → file accessible tại `public_url` trên R2
  - [x] `POST /api/v1/media/upload-url` với unsupported content_type → 422
  - [x] `DELETE /api/v1/media` với own file key → 204, file xóa khỏi R2
  - [x] `DELETE /api/v1/media` với file key của user khác → 403
  - [x] `UploadService` singleton tồn tại tại `lib/services/upload_service.dart`
  - [x] Flutter: file > 5MB với folder=avatar → `UploadValidationException` ngay lập tức, không gọi API
  - [x] STORAGE credentials không xuất hiện trong bất kỳ API response
  - [x] `.env.example` có tất cả `STORAGE_*` variables

---

### ✅ [5] STAFF-02 — Staff Number Generation

**Plan:** [STAFF-02-plan.md](./STAFF-02-plan.md)
**Depends on:** [1] DB-01
**Completed:** 2026-03-11

- [x] **[STAFF-02]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `StaffNumberService` tồn tại tại `internal/service/staff_number_service.go`
  - [x] `Generate()` dùng `crypto/rand` (không phải `math/rand`)
  - [x] Kết quả luôn đúng 6 ký tự, zero-padded (e.g. `"000042"`)
  - [x] `StaffNumberExists()` trả `true` cho number đã có trong DB
  - [x] Khi 10 attempts đều collide → `ErrStaffNumberExhausted` trả về
  - [x] `StaffService.CreateProfile()` đã inject và gọi `StaffNumberService`

---

### ✅ [6] STAFF-03 — Job Type Selection

**Plan:** [STAFF-03-plan.md](./STAFF-03-plan.md)
**Depends on:** [1] DB-01
**Completed:** 2026-03-11

- [x] **[STAFF-03]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `GET /api/v1/staff/job-categories` → 200, đúng 21 items, mỗi item có `key`, `label_ja`, `label_en`, `icon`
  - [x] `GET /api/v1/staff/job-categories` không có JWT → 401
  - [x] `config.IsValidJobCategory("beauty")` trả `true`
  - [x] `config.IsValidJobCategory("invalid")` trả `false`
  - [x] `JobCategoryDropdown` widget tồn tại tại `lib/widgets/job_category_dropdown.dart`
  - [x] `JobCategoryChips` widget tồn tại tại `lib/widgets/job_category_chips.dart`
  - [x] `StaffService.getJobCategories()` cache in-memory, không gọi network lần 2
  - [x] Dropdown hiển thị `CircularProgressIndicator` khi đang load

---

## Phase 3 — Core Staff

### ✅ [7] STAFF-01 — Staff Profile CRUD

**Plan:** [STAFF-01-plan.md](./STAFF-01-plan.md)
**Depends on:** [5] STAFF-02, [6] STAFF-03, [1] DB-01
**Completed:** 2026-03-11

- [x] **[STAFF-01]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `POST /api/v1/staff/profile` → 201, `staff_number` là 6 digits, `users.is_staff_registered = true`, `users.role = 'staff'`
  - [x] `POST /api/v1/staff/profile` lần 2 (duplicate) → 409
  - [x] `POST /api/v1/staff/profile` với `job_category = "invalid"` → 422
  - [x] `PATCH /api/v1/staff/profile` với chỉ `bio` → 200, chỉ bio thay đổi
  - [x] `PATCH /api/v1/staff/profile` body rỗng → 400
  - [x] `GET /api/v1/staff/me` → 200, full profile có `portfolio_photos` array
  - [x] `GET /api/v1/staff/:userID` → không có `email` hay `password_hash` trong response
  - [x] `GET /api/v1/staff/:invalidID` → 404
  - [x] `StaffRegistrationScreen` validate trước khi gọi API
  - [x] `StaffProfileScreen` hiển thị data từ API, không còn mock data
  - [x] `StaffProvider` đăng ký trong `MultiProvider`
  - [x] Transaction trong CreateProfile: users + staff_profiles update atomically

---

## Phase 4 — Staff Complete

### ✅ [8] STAFF-05 — Portfolio Photo Upload

**Plan:** [STAFF-05-plan.md](./STAFF-05-plan.md)
**Depends on:** [7] STAFF-01, [4] DB-05
**Completed:** 2026-03-11

- [x] **[STAFF-05]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `POST /api/v1/staff/portfolio/photos` → 201, photo thêm vào `staff_portfolio_photos`
  - [x] `POST` khi đã có 12 photos → 422 `"Portfolio is full. Maximum 12 photos allowed."`
  - [x] `DELETE /api/v1/staff/portfolio/photos/:id` (own photo) → 204, file xóa khỏi R2
  - [x] `DELETE` photo của staff khác → 403
  - [x] `PATCH /api/v1/staff/portfolio/photos/reorder` → 200, `display_order` updated trong DB
  - [x] `PortfolioEditScreen` tồn tại với grid 3 cột
  - [x] Flutter: file > 10MB → SnackBar `"File is too large. Maximum size is 10 MB."`
  - [x] Flutter: "Add Photo" tile ẩn khi count = 12
  - [x] Flutter: Delete confirmation dialog xuất hiện, Cancel không xóa
  - [x] Flutter: Owner thấy "Edit Portfolio" button, visitor không thấy

---

### ✅ [9] AUTH-02 — Google Sign-In

**Plan:** [AUTH-02-plan.md](./AUTH-02-plan.md)
**Depends on:** [2] AUTH-01
**Completed:** 2026-03-11

- [x] **[AUTH-02]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `POST /api/v1/auth/google` với valid idToken (new user) → 201, `auth_provider = 'google'`
  - [x] `POST /api/v1/auth/google` với email đã tồn tại → 200, `google_id` linked, không tạo duplicate row
  - [x] `POST /api/v1/auth/google` với expired token → 400 `invalid_token`
  - [x] `POST /api/v1/auth/google` với disabled account → 403
  - [x] Google button render đúng branding (không bị resize/recolor)
  - [x] Dismiss account picker → Login screen không thay đổi, không có error
  - [x] Sign-in thành công → tokens trong `FlutterSecureStorage`
  - [x] New user có `privacy_policy_accepted = false` → Privacy Policy modal trước `/home`
  - [x] Flow hoạt động trên iOS, Android, và Web

---

## Phase 5 — Profile Screen + Feed

### ✅ [10] STAFF-07 — Staff Profile Screen Connected

**Plan:** [STAFF-07-plan.md](./STAFF-07-plan.md)
**Depends on:** [7] STAFF-01, [8] STAFF-05
**Completed:** 2026-03-11

- [x] **[STAFF-07]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `StaffProfileScreen` fetch data từ `GET /api/v1/staff/:userID`, không còn mock data
  - [x] `CircularProgressIndicator` hiển thị khi đang load
  - [x] Tất cả fields (name, staff number, job title, category, rating, reviews, followers, bio, portfolio) render từ API
  - [x] Visitor mode: Follow + Book Now + Tip buttons visible
  - [x] Owner mode (`userID == AuthProvider.currentUser.id`): Edit Profile + Edit Portfolio buttons visible, Follow/Book/Tip ẩn
  - [x] 404 → full-screen empty state `"This staff profile could not be found."` (không có Retry)
  - [x] Network error → full-screen error state với Retry button hoạt động
  - [x] Portfolio photos dùng `CachedNetworkImage`, sorted by `display_order`
  - [x] Profile không có bio → bio section ẩn hoàn toàn
  - [x] `email`, `password_hash` không có trong response và không được log/display

---

### ✅ [11] FEED-01 — Create Post

**Plan:** [FEED-01-plan.md](./FEED-01-plan.md)
**Depends on:** [4] DB-05, [2] AUTH-01
**Completed:** 2026-03-11

- [x] **[FEED-01]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] `POST /api/v1/posts` với image media_url + caption → 201, `media_type = "image"`
  - [x] `POST /api/v1/posts` với caption only → 201, `media_url = null`
  - [x] `POST /api/v1/posts` không có content lẫn media → 400
  - [x] `POST /api/v1/posts` với `media_type = "gif"` → 422
  - [x] `POST /api/v1/posts` với caption > 500 chars → 422
  - [x] `GET /api/v1/posts/feed` → 200 với `posts`, `next_cursor`, `has_more`
  - [x] `GET /api/v1/posts/feed?category=nail_art` → chỉ posts của nail art staff
  - [x] `GET /api/v1/posts/feed?limit=51` → 422
  - [x] `CreatePostScreen` tồn tại với media picker, caption, upload progress bar
  - [x] Flutter: Tap Share không có media/caption → `"Add a photo, video, or caption to share."`
  - [x] Flutter: Upload thành công → green SnackBar `"Post shared successfully."` + navigate to feed
  - [x] Flutter: Share button disabled + loading khi đang upload/submit

---

## Phase 6 — Feed UI

### ✅ [12] FEED-02 — TikTok Feed

**Plan:** [FEED-02-plan.md](./FEED-02-plan.md)
**Depends on:** [11] FEED-01, [6] STAFF-03
**Completed:** 2026-03-11

- [x] **[FEED-02]** Hoàn thành khi tất cả criteria sau đều đúng:
  - [x] Home tab render full-screen vertical `PageView`, one post per page
  - [x] Swipe up/down snap đúng post boundaries
  - [x] Video post auto-play muted khi enter viewport, pause khi leave
  - [x] Chỉ 1 video play tại 1 thời điểm
  - [x] Tap video → mute toggle với icon flash
  - [x] Image post fill screen với `BoxFit.cover`
  - [x] Author overlay ở bottom-left, action column ở right
  - [x] Khi còn 2 posts đến cuối → auto fetch next page, append mượt mà
  - [x] `has_more = false` → không fetch thêm
  - [x] Category chip select → feed reload với filtered posts
  - [x] Tap author name/avatar → navigate to `StaffProfileScreen`
  - [x] Initial load error → full-screen error state với Retry button
  - [x] Empty feed → `"No posts yet. Check back soon!"`
  - [x] Không còn mock/hardcoded posts
  - [x] Video controller disposed khi widget unmount (kiểm tra bằng Flutter DevTools memory)

---

## Hướng dẫn sử dụng

**Khi bắt đầu một plan:**
1. Mở file `<TICKET>-plan.md` tương ứng
2. Đọc từ đầu đến cuối trước khi code
3. Làm theo thứ tự Backend Steps → Flutter Steps
4. Dùng "Files to Create/Modify" table như checklist

**Khi hoàn thành một plan:**
1. Chạy qua toàn bộ Verification Checklist trong plan file
2. Tick từng item trong "Completion Criteria" của plan đó ở file này
3. Nếu tất cả checked → điền ngày vào `Completed: ___________`
4. Update "Tiến độ tổng quan" ở trên (`Completed: X / 12`)
5. Đổi `🔲` thành `✅` ở header của plan đó

**Nếu một item chưa pass:**
- Không được mark plan là hoàn thành
- Fix issue trước, test lại, rồi mới check

---

*Last updated: 2026-03-10*
