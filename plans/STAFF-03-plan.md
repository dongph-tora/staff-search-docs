# [STAFF-03] Job Type Selection (21 Categories) — Implementation Plan

**Spec:** `docs/screens/STAFF-03-job-type-selection.md`
**Priority:** P1 MVP
**Phụ thuộc:** DB-01 (`job_category` field trên `staff_profiles`), AUTH-01 (JWT auth)

---

## Backend Steps

### Step 1: Tạo static job categories config

**File:** `internal/config/job_categories.go` (tạo mới)

Struct `JobCategory`:
```go
type JobCategory struct {
  Key     string `json:"key"`
  LabelJa string `json:"label_ja"`
  LabelEn string `json:"label_en"`
  Icon    string `json:"icon"`
}
```

Slice `JobCategories []JobCategory` với 21 entries:
```
{key: "beauty",          label_ja: "美容師",              label_en: "Beautician",          icon: "💇"},
{key: "nail_art",        label_ja: "ネイリスト",           label_en: "Nail Artist",          icon: "💅"},
{key: "eyelash",         label_ja: "まつ毛エクステ",       label_en: "Eyelash Extension",    icon: "👁"},
{key: "massage",         label_ja: "マッサージ",           label_en: "Massage Therapist",    icon: "💆"},
{key: "facial",          label_ja: "フェイシャルエステ",   label_en: "Facial Esthetician",   icon: "✨"},
{key: "hair_removal",    label_ja: "脱毛",                label_en: "Hair Removal",          icon: "🌿"},
{key: "makeup",          label_ja: "メイクアップ",         label_en: "Makeup Artist",         icon: "💄"},
{key: "hair_stylist",    label_ja: "ヘアスタイリスト",     label_en: "Hair Stylist",          icon: "💈"},
{key: "barber",          label_ja: "理容師",              label_en: "Barber",                icon: "✂️"},
{key: "spa",             label_ja: "スパセラピスト",       label_en: "Spa Therapist",         icon: "🛁"},
{key: "waxing",          label_ja: "ワックス脱毛",         label_en: "Waxing",                icon: "🕯️"},
{key: "tattoo",          label_ja: "タトゥーアーティスト", label_en: "Tattoo Artist",         icon: "🎨"},
{key: "food_beverage",   label_ja: "飲食",                label_en: "Food & Beverage",       icon: "🍽️"},
{key: "bartender",       label_ja: "バーテンダー",         label_en: "Bartender",             icon: "🍸"},
{key: "sommelier",       label_ja: "ソムリエ",            label_en: "Sommelier",             icon: "🍷"},
{key: "personal_trainer",label_ja: "パーソナルトレーナー", label_en: "Personal Trainer",      icon: "💪"},
{key: "yoga",            label_ja: "ヨガインストラクター", label_en: "Yoga Instructor",       icon: "🧘"},
{key: "dance",           label_ja: "ダンスインストラクター",label_en: "Dance Instructor",     icon: "💃"},
{key: "photography",     label_ja: "フォトグラファー",     label_en: "Photographer",          icon: "📷"},
{key: "music",           label_ja: "音楽講師",            label_en: "Music Instructor",      icon: "🎵"},
{key: "other",           label_ja: "その他",              label_en: "Other",                 icon: "⭐"},
```

Helper function `IsValidJobCategory(key string) bool`:
```go
for _, cat := range JobCategories {
  if cat.Key == key { return true }
}
return false
```

### Step 2: Thêm GetJobCategories handler

**File:** `internal/handler/staff_handler.go` (thêm method hoặc tạo nếu chưa có)

Method `GetJobCategories(c fiber.Ctx) error`:
1. Read `userID` từ `c.Locals("userID")` (để satisfy JWT middleware, không dùng trong response)
2. Return `200` với `config.JobCategories` slice

### Step 3: Thêm job_category validation vào StaffHandler

**File:** `internal/handler/staff_handler.go`

Trong `CreateProfile` và `UpdateProfile` handlers, sau khi parse request body:
- Gọi `config.IsValidJobCategory(req.JobCategory)` → return `422 validation_error` với message "Invalid job category." nếu false

### Step 4: Đăng ký route

**File:** `router/router.go`

Trong protected group, thêm staff routes:
```go
staff := protected.Group("/staff")
staff.Get("/job-categories", staffHandler.GetJobCategories)
// Các routes STAFF-01 sẽ thêm vào đây sau
```

---

## Flutter Steps

### Step 1: Tạo JobCategory model

**File:** `lib/models/job_category.dart`

```dart
class JobCategory {
  final String key;
  final String labelJa;
  final String labelEn;
  final String icon;

  const JobCategory({
    required this.key,
    required this.labelJa,
    required this.labelEn,
    required this.icon,
  });

  factory JobCategory.fromJson(Map<String, dynamic> json) => JobCategory(
    key: json['key'] as String,
    labelJa: json['label_ja'] as String,
    labelEn: json['label_en'] as String,
    icon: json['icon'] as String,
  );
}
```

### Step 2: Thêm getJobCategories vào StaffService

**File:** `lib/services/staff_service.dart` (tạo mới nếu chưa có, hoặc thêm method)

```dart
class StaffService {
  static final StaffService instance = StaffService._();
  StaffService._();

  // In-memory cache cho session
  List<JobCategory>? _cachedCategories;

  Future<List<JobCategory>> getJobCategories() async {
    if (_cachedCategories != null) return _cachedCategories!;

    final response = await ApiClient.instance.get('/api/v1/staff/job-categories');
    if (response.statusCode == 200) {
      final list = (response.data['data'] as List)
        .map((e) => JobCategory.fromJson(e as Map<String, dynamic>))
        .toList();
      _cachedCategories = list;
      return list;
    }
    throw Exception(response.message ?? 'Failed to load categories');
  }
}
```

Cache là in-memory (session-only). Clear khi app restart.

### Step 3: Tạo JobCategoryDropdown widget

**File:** `lib/widgets/job_category_dropdown.dart`

StatefulWidget với:
- `onChanged` callback nhận `String?` (category key)
- `initialValue` String? (để pre-fill khi edit)
- `validator` Function? cho form validation

Behavior:
- Trong `initState`: gọi `StaffService.instance.getJobCategories()`
- Trong loading state: hiển thị `CircularProgressIndicator` (20×20, strokeWidth 2) thay cho dropdown
- Sau khi load: render `DropdownButtonFormField<String>` với items từ API
  - `value`: category key (String)
  - `displayLabel`: `"${category.icon} ${category.labelJa}"` (emoji + Japanese)
  - `validator`: nếu value null → "Please select a job category."
- Error state (network error): show empty dropdown với retry button

### Step 4: Tạo JobCategoryChips widget

**File:** `lib/widgets/job_category_chips.dart`

StatefulWidget với:
- `onSelected` callback nhận `String?` (category key, null = "すべて")
- `selectedKey` String? (currently selected)

Layout: `SingleChildScrollView` với `Row` hoặc `Wrap`:
- Đầu tiên: chip "すべて" (All) — không có key
- Sau đó: 21 category chips với `"${category.icon} ${category.labelJa}"`

Chip style:
- Unselected: mặc định Flutter `FilterChip`
- Selected: primary color background, white text

Behavior:
- Load categories từ `StaffService.instance.getJobCategories()`
- Khi tap chip: gọi `onSelected(selectedKey)`, cập nhật local selected state

### Step 5: Modify các screens để dùng widgets mới

**File:** `lib/screens/staff/staff_registration_screen.dart`

Thay hardcoded dropdown items bằng `JobCategoryDropdown` widget.

**File:** `lib/screens/staff/staff_profile_edit_screen.dart`

Thay hardcoded dropdown items bằng `JobCategoryDropdown(initialValue: currentProfile?.jobCategory)`.

**File:** `lib/screens/home_screen.dart` (hoặc home/home_screen.dart)

Thêm `JobCategoryChips` widget bên dưới AppBar. Wire `onSelected` callback để filter feed theo category key.

**File:** `lib/screens/search_screen.dart`

Thêm `JobCategoryChips` widget. Wire `onSelected` callback để filter search results.

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/config/job_categories.go` | CREATE — static list + IsValidJobCategory helper |
| `internal/handler/staff_handler.go` | CREATE hoặc MODIFY — thêm GetJobCategories handler |
| `router/router.go` | MODIFY — đăng ký GET /api/v1/staff/job-categories |

### Flutter
| File | Action |
|---|---|
| `lib/models/job_category.dart` | CREATE |
| `lib/widgets/job_category_dropdown.dart` | CREATE |
| `lib/widgets/job_category_chips.dart` | CREATE |
| `lib/services/staff_service.dart` | CREATE hoặc MODIFY — thêm getJobCategories() với cache |
| `lib/screens/staff/staff_registration_screen.dart` | MODIFY — dùng JobCategoryDropdown |
| `lib/screens/staff/staff_profile_edit_screen.dart` | MODIFY — dùng JobCategoryDropdown |
| `lib/screens/home_screen.dart` | MODIFY — thêm JobCategoryChips |
| `lib/screens/search_screen.dart` | MODIFY — thêm JobCategoryChips |

---

## Verification Checklist

- [ ] `GET /api/v1/staff/job-categories` → 200, array đúng 21 items
- [ ] Tất cả 21 items có key, label_ja, label_en, icon không trống
- [ ] `GET /api/v1/staff/job-categories` không có JWT → 401
- [ ] Backend: POST staff profile với `job_category = "beauty"` → accepted
- [ ] Backend: POST staff profile với `job_category = "invalid"` → 422
- [ ] Backend: POST staff profile với `job_category = "other"` → accepted
- [ ] Flutter: Dropdown hiển thị loading indicator trong khi fetch
- [ ] Flutter: Dropdown hiển thị 21 Japanese labels sau khi load
- [ ] Flutter: Chọn "ネイリスト" → key "nail_art" được stored trong form state
- [ ] Flutter: Submit form không chọn category → "Please select a job category.", không gọi API
- [ ] Flutter: Gọi `getJobCategories()` lần 2 → trả cached data, không gọi network
- [ ] Flutter: Category chip trên Home screen filter đúng theo key
