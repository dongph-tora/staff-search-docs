# [STAFF-07] Staff Profile Screen Connected to DB — Implementation Plan

**Spec:** `docs/screens/STAFF-07-staff-profile-screen-connected.md`
**Priority:** P1 MVP
**Phụ thuộc:**
- STAFF-01: `StaffService.getProfile(userID)`, `StaffProfileResponse`, `GET /api/v1/staff/:userID`
- STAFF-03: `JobCategory` model để hiển thị category label
- STAFF-05: `portfolioPhotos` trong `StaffProfile` model

> Ticket này là Flutter-only. Backend không cần thêm gì (dùng endpoints từ STAFF-01).

---

## Backend Steps

Không có backend changes mới. Chỉ cần verify:

### Verify: GET /api/v1/staff/:userID trả đủ fields

Đảm bảo response từ STAFF-01 bao gồm:
- Nested `user` object: `{name, avatar_url}` (từ JOIN với users table)
- `portfolio_photos` array sorted by `display_order ASC`
- `staff_category` key (chính là `job_category` field)
- `staff_number`, `rating`, `review_count`, `followers_count`, `bio`, `location`, `accept_bookings`

Nếu STAFF-01 chưa JOIN với users table để lấy `name` và `avatar_url`, cần update `StaffRepository.FindByUserID()` để Preload User:
```go
r.db.WithContext(ctx).
  Preload("User").
  Preload("PortfolioPhotos", func(db *gorm.DB) *gorm.DB {
    return db.Order("display_order ASC")
  }).
  Where("user_id = ?", userID).
  First(&profile)
```

Và update `StaffProfileResponse` DTO để include `user.name` và `user.avatar_url`.

---

## Flutter Steps

### Step 1: Thêm cached_network_image package

**File:** `pubspec.yaml`

```yaml
cached_network_image: ^3.3.1
```

Chạy `flutter pub get`.

### Step 2: Tạo StaffStatsRow widget

**File:** `lib/widgets/staff_stats_row.dart`

Widget nhận: `rating` double, `reviewCount` int, `followersCount` int.

Layout: `Row` với 3 items separated by dividers:
- Item 1: star icon + `rating.toStringAsFixed(1)` (e.g., "4.8")
- Item 2: `reviewCount.toString()` + " reviews"
- Item 3: formatted `followersCount` (e.g., "1.2k" nếu >= 1000) + " followers"

### Step 3: Tạo PortfolioGrid widget

**File:** `lib/widgets/portfolio_grid.dart`

Widget nhận: `photos` List<PortfolioPhoto>, `onPhotoTap` callback.

Layout: `GridView.builder` với 3 columns, aspect ratio 1:1.

Mỗi photo tile: `CachedNetworkImage` với `BoxFit.cover`.

Tap trên photo: gọi `onPhotoTap(photoUrl)` → sẽ navigate đến fullscreen viewer.

Empty state: nếu `photos.isEmpty` → hiển thị Text "No portfolio photos yet." ở center.

### Step 4: Rewrite StaffProfileScreen

**File:** `lib/screens/staff/staff_profile_screen.dart`

**State management:**
```dart
StaffProfile? _profile;
bool _isLoading = true;
String? _errorMessage;
bool _isOwner = false;
```

**Route argument:** nhận `String userID` qua `ModalRoute.of(context)!.settings.arguments`.

**initState:**
1. Gọi `_loadProfile()`
2. Compute `_isOwner = context.read<AuthProvider>().currentUser?.id == userID`

**_loadProfile() method:**
1. `setState(() { _isLoading = true; _errorMessage = null; })`
2. Gọi `StaffService.instance.getProfile(userID)`
3. Nếu thành công: `setState(() { _profile = profile; _isLoading = false; })`
4. Nếu null (404): `setState(() { _errorMessage = "This staff profile could not be found."; _isLoading = false; })`
5. Nếu network error: `setState(() { _errorMessage = "Unable to connect. Check your network and retry."; _isLoading = false; })`
6. Nếu server error: `setState(() { _errorMessage = "Something went wrong. Please try again later."; _isLoading = false; })`

**build() layout:**
```
Scaffold
├── AppBar: title = _profile?.user?.name ?? "Profile", back button
└── body:
    ├── if _isLoading: Center(CircularProgressIndicator)
    ├── if _errorMessage != null: _buildErrorState()
    └── else: _buildProfileContent()
```

**_buildErrorState():**
- Column: icon + message text + (Retry button nếu `_errorMessage != "...not found..."`)
- Retry taps → gọi `_loadProfile()`
- 404 → không có Retry, chỉ có back button

**_buildProfileContent():**

```
SingleChildScrollView
└── Column
    ├── Header section:
    │   ├── CircleAvatar + CachedNetworkImage (80px, fallback: initials từ name)
    │   ├── Text: "Staff ID: #${_profile!.staffNumber}"
    │   ├── Text: _profile!.jobTitle
    │   └── Chip: category label_ja (lookup từ StaffService.cachedCategories)
    │
    ├── StaffStatsRow(rating, reviewCount, followersCount)
    │
    ├── if location != null: Row(📍 icon + location text)
    │
    ├── if bio != null && bio!.isNotEmpty: "About" section với expandable text
    │
    ├── "Portfolio" section:
    │   └── PortfolioGrid(photos: _profile!.portfolioPhotos, onPhotoTap: _openPhotoViewer)
    │
    └── Action buttons:
        if _isOwner:
          ├── ElevatedButton("Edit Profile") → navigate to StaffProfileEditScreen
          └── OutlinedButton("Edit Portfolio") → navigate to PortfolioEditScreen
        else:
          ├── OutlinedButton("Follow") → stub (SOCIAL-01)
          ├── ElevatedButton("Book Now", disabled if !acceptBookings) → stub (BOOK-01)
          └── TextButton("Tip") → stub (PAY-07)
```

**Photo viewer:**
Khi tap portfolio photo: navigate đến fullscreen image viewer (dùng `showDialog` với `InteractiveViewer` hoặc package `photo_view`).

### Step 5: Update route trong main.dart

**File:** `lib/main.dart`

Đảm bảo route `/staff-profile` nhận `arguments: userID`:
```dart
'/staff-profile': (context) {
  final userID = ModalRoute.of(context)!.settings.arguments as String;
  return StaffProfileScreen(userID: userID);
},
```

Hoặc nếu dùng `Navigator.pushNamed('/staff-profile', arguments: userID)` ở mọi nơi trong app.

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/repository/staff_repository.go` | VERIFY/MODIFY — Preload User trong FindByUserID |
| `internal/dto/staff_dto.go` | VERIFY/MODIFY — StaffProfileResponse include user.name, user.avatar_url |

### Flutter
| File | Action |
|---|---|
| `pubspec.yaml` | MODIFY — thêm cached_network_image |
| `lib/widgets/staff_stats_row.dart` | CREATE |
| `lib/widgets/portfolio_grid.dart` | CREATE |
| `lib/screens/staff/staff_profile_screen.dart` | REWRITE — remove all mock data, add API call, loading/error/owner states |
| `lib/main.dart` | MODIFY — update StaffProfileScreen route |

---

## Verification Checklist

- [ ] Screen hiển thị CircularProgressIndicator khi đang load
- [ ] Tất cả profile fields (name, staff number, job title, category badge, rating, reviews, followers, bio, portfolio) hiển thị từ API
- [ ] Không còn mock data nào trong screen
- [ ] Visitor mode: thấy Follow + Book Now + Tip buttons
- [ ] Owner mode (`userID == AuthProvider.currentUser.id`): thấy Edit Profile + Edit Portfolio buttons
- [ ] 404 → full-screen empty state "This staff profile could not be found." (không có Retry)
- [ ] Network error → full-screen error state với Retry button
- [ ] Retry button gọi lại API
- [ ] Portfolio grid sorted by display_order
- [ ] Portfolio photos dùng CachedNetworkImage
- [ ] Profile không có bio → bio section ẩn hoàn toàn
- [ ] Profile không có portfolio photos → "No portfolio photos yet."
- [ ] response không chứa email/password_hash
- [ ] `isOwner` sử dụng `AuthProvider.currentUser.id`, không từ profile response
