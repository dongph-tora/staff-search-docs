# [STAFF-03] Job Type Selection (20+ Categories)

| Field | Value |
|---|---|
| **Type** | Feature |
| **Priority** | P1 MVP |
| **Estimate** | 0.5 days manual / 0.25 days vibe coding |
| **Dependencies** | STAFF-01 — `job_category` field on `staff_profiles` (DB-01 Section 6.2); `CreateStaffProfileRequest` DTO (STAFF-01 Section 5.2); `StaffRegistrationScreen` job category dropdown (STAFF-01 Section 3.1). |
| **Platforms** | iOS · Android · Web |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format (Section 2), `ApiClient` (Section 6), error display (Section 10). |

---

## 1. Overview

This ticket defines the canonical list of job categories (職種) that a staff member can belong to, provides a backend endpoint to retrieve the full list, and implements the category picker widget used in the staff registration and edit forms (STAFF-01). The category list is stored as a static configuration on the backend and returned via a public API endpoint so the Flutter app does not hardcode it. Each category has a machine-readable key (used in `staff_profiles.job_category`), a Japanese display label (shown in the UI), and an English display label.

### End-to-end flow

    [Flutter App]                    [Go API]
         │                               │
         │── GET /api/v1/staff/──────── ►│
         │       job-categories          │── return static category list
         │◄── 200 [{key, label_ja,       │
         │          label_en, icon}] ────│
         │                               │
         │── User selects category ──────┤ (no API call; stored locally
         │── Stored in form state        │  until profile save)

---

## 2. User Flows

### 2.1 Happy Path — Select Job Category During Registration

1. User is on `StaffRegistrationScreen` (STAFF-01 Section 3.1).
2. On screen load, `StaffService.getJobCategories()` is called. A `CircularProgressIndicator` is shown in the dropdown while loading.
3. API returns the list of 21 categories with keys and Japanese labels.
4. The dropdown renders all 21 options with their Japanese labels.
5. User taps the dropdown and selects e.g., "ネイリスト".
6. The selected category key (`nail_art`) is stored in the form state.
7. On Save, the key is sent as `job_category` in `POST /api/v1/staff/profile`.

### 2.2 Happy Path — Filter Feed by Job Category

1. On the Home feed or Search screen, a horizontal category chip row is rendered using the same job category list.
2. User taps a chip (e.g., "美容師") to filter by `beauty`.
3. The chip becomes highlighted. The feed reloads with staff filtered by that category.

### 2.3 Error Cases

| Scenario | Behaviour |
|---|---|
| Network error fetching categories | Red SnackBar: `"Unable to connect. Check your network and retry."` Dropdown shows empty state with retry option. |
| Server error (500) fetching categories | Red SnackBar: `"Something went wrong. Please try again later."` |
| Staff submits profile with no category selected | Form validation blocks submission. Inline error: `"Please select a job category."` |

---

## 3. UI / Screen

### 3.1 Job Category Dropdown (within StaffRegistrationScreen and StaffProfileEditScreen)

> These screens already exist from STAFF-01. This ticket adds the category picker widget.

    ┌─────────────────────────────────────────┐
    │  Job Category *                         │
    │  ┌───────────────────────────────────┐  │
    │  │ ネイリスト                    ▼  │  │
    │  └───────────────────────────────────┘  │
    │                                         │
    │  (expanded dropdown list:)              │
    │  ┌───────────────────────────────────┐  │
    │  │ 💇 美容師 (Beautician)            │  │
    │  │ 💅 ネイリスト (Nail Artist)       │  │
    │  │ 👁 まつ毛エクステ (Eyelash)       │  │
    │  │ 💆 マッサージ (Massage)           │  │
    │  │  ... (17 more)                   │  │
    │  └───────────────────────────────────┘  │
    └─────────────────────────────────────────┘

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| `jobCategoryDropdown` | `DropdownButtonFormField<String>` | Displays `label_ja` for each item; stores `key` as value; shows `CircularProgressIndicator` while categories loading |
| Category chip row | `Wrap` of `FilterChip` widgets | Used on Home/Search screens for category filtering; single selection; selected chip uses primary color background |

### 3.2 Category Chip Row (Home / Search screens)

> Used in feed and search filtering. Rendered as a horizontally scrollable chip list.

    ┌────────────────────────────────────────────────────┐
    │  すべて  美容師  ネイリスト  マッサージ  飲食  ›  │
    └────────────────────────────────────────────────────┘

Changes required to existing screens:
- `lib/screens/home/home_screen.dart` — add category chip row below the AppBar; wire to feed filtering.
- `lib/screens/search/search_screen.dart` — add category chip row; wire to search filtering.

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/models/job_category.dart` | `JobCategory` value object with `key`, `labelJa`, `labelEn`, `icon` fields |
| `lib/widgets/job_category_dropdown.dart` | Reusable `DropdownButtonFormField` that fetches and renders job categories |
| `lib/widgets/job_category_chips.dart` | Horizontal scrollable chip row for filtering; accepts `onSelected` callback |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/services/staff_service.dart` | Add `getJobCategories()` method — calls `GET /api/v1/staff/job-categories`; caches result in memory for the session |
| `lib/screens/staff/staff_registration_screen.dart` | Replace hardcoded dropdown items with `JobCategoryDropdown` widget |
| `lib/screens/staff/staff_profile_edit_screen.dart` | Replace hardcoded dropdown items with `JobCategoryDropdown` widget |
| `lib/screens/home/home_screen.dart` | Add `JobCategoryChips` widget below AppBar |
| `lib/screens/search/search_screen.dart` | Add `JobCategoryChips` widget for category filtering |

### 4.3 JobCategory Caching

`StaffService.getJobCategories()` fetches the category list once per app session and caches the result in memory. On subsequent calls within the same session, the cached list is returned without a network request. The cache is cleared when the app is restarted. This avoids repeated network calls every time a dropdown or chip row is rendered.

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| `GET` | `/api/v1/staff/job-categories` | Returns the complete list of job categories | Yes |

### 5.2 GET /api/v1/staff/job-categories

**Response `200`:**

Returns an array of category objects. Each object has the following fields:

| Field | Type | Description |
|---|---|---|
| `key` | string | Machine-readable identifier used in `staff_profiles.job_category` |
| `label_ja` | string | Japanese display label shown in the UI |
| `label_en` | string | English display label |
| `icon` | string | Emoji icon for the category |

The full list of 21 categories is:

| key | label_ja | label_en | icon |
|---|---|---|---|
| `beauty` | 美容師 | Beautician | 💇 |
| `nail_art` | ネイリスト | Nail Artist | 💅 |
| `eyelash` | まつ毛エクステ | Eyelash Extension | 👁 |
| `massage` | マッサージ | Massage Therapist | 💆 |
| `facial` | フェイシャルエステ | Facial Esthetician | ✨ |
| `hair_removal` | 脱毛 | Hair Removal | 🌿 |
| `makeup` | メイクアップ | Makeup Artist | 💄 |
| `hair_stylist` | ヘアスタイリスト | Hair Stylist | 💈 |
| `barber` | 理容師 | Barber | ✂️ |
| `spa` | スパセラピスト | Spa Therapist | 🛁 |
| `waxing` | ワックス脱毛 | Waxing | 🕯️ |
| `tattoo` | タトゥーアーティスト | Tattoo Artist | 🎨 |
| `food_beverage` | 飲食 | Food & Beverage | 🍽️ |
| `bartender` | バーテンダー | Bartender | 🍸 |
| `sommelier` | ソムリエ | Sommelier | 🍷 |
| `personal_trainer` | パーソナルトレーナー | Personal Trainer | 💪 |
| `yoga` | ヨガインストラクター | Yoga Instructor | 🧘 |
| `dance` | ダンスインストラクター | Dance Instructor | 💃 |
| `photography` | フォトグラファー | Photographer | 📷 |
| `music` | 音楽講師 | Music Instructor | 🎵 |
| `other` | その他 | Other | ⭐ |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `401` | `unauthorized` | Missing or invalid JWT |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Read `userID` from `c.Locals("userID")` (to satisfy the JWT middleware; not used in the response).
2. Return `200` with the static category list defined in `internal/config/job_categories.go`.

### 5.3 Service Responsibilities

No `StaffService` method is needed. The handler returns a static list defined in `internal/config/job_categories.go` as a Go slice of `JobCategory` structs. The list is a compile-time constant and requires no database query.

---

## 6. Database

Not applicable for this ticket. The `job_category` column is `VARCHAR(50)` as defined in DB-01 Section 6.2. No new columns or tables are added. The category values stored are the `key` strings defined in Section 5.2.

---

## 7. Configuration

Not applicable for this ticket. The job category list is hardcoded in `internal/config/job_categories.go`. No environment variables are needed.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | GET /api/v1/staff/job-categories with valid JWT | `200`, array of exactly 21 category objects returned |
| 2 | Every category object has `key`, `label_ja`, `label_en`, `icon` | All four fields present and non-empty for all 21 entries |
| 3 | GET without JWT | `401 unauthorized` |
| 4 | POST /api/v1/staff/profile with `job_category = "beauty"` | Accepted; `staff_profiles.job_category = "beauty"` stored |
| 5 | POST /api/v1/staff/profile with `job_category = "invalid_key"` | `422 validation_error` |
| 6 | The `other` category key is in the allowlist | `POST /api/v1/staff/profile` with `job_category = "other"` succeeds |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Registration screen loads | Dropdown shows `CircularProgressIndicator` while categories fetch |
| 2 | Categories loaded successfully | Dropdown renders all 21 Japanese labels |
| 3 | User selects "ネイリスト" | Dropdown shows "ネイリスト"; `nail_art` key stored in form state |
| 4 | Form submitted without category selected | Inline error: `"Please select a job category."` No API call made |
| 5 | Network error fetching categories | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| 6 | Categories already cached | Second call to `getJobCategories()` returns cached list; no network request |
| 7 | Category chip selected on Home screen | Chip highlighted; feed filters by selected category |

---

## 9. Security Checklist

- [ ] `job_category` is validated server-side against the explicit 21-key allowlist before being stored; clients cannot inject arbitrary string values.
- [ ] The category list endpoint requires authentication; anonymous access is not permitted.
- [ ] No sensitive data is included in the category list response.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Model | `lib/models/job_category.dart` | Create — JobCategory value object |
| Flutter — Widget | `lib/widgets/job_category_dropdown.dart` | Create — reusable dropdown widget |
| Flutter — Widget | `lib/widgets/job_category_chips.dart` | Create — horizontal filter chip row |
| Flutter — Service | `lib/services/staff_service.dart` | Modify — add getJobCategories() with in-memory cache |
| Flutter — Screen | `lib/screens/staff/staff_registration_screen.dart` | Modify — use JobCategoryDropdown widget |
| Flutter — Screen | `lib/screens/staff/staff_profile_edit_screen.dart` | Modify — use JobCategoryDropdown widget |
| Flutter — Screen | `lib/screens/home/home_screen.dart` | Modify — add JobCategoryChips below AppBar |
| Flutter — Screen | `lib/screens/search/search_screen.dart` | Modify — add JobCategoryChips for filtering |
| Go — Handler | `internal/handler/staff_handler.go` | Modify — add GetJobCategories handler |
| Go — Config | `internal/config/job_categories.go` | Create — static JobCategory list (21 entries) |
| Go — Router | `router/router.go` | Modify — register GET /api/v1/staff/job-categories |

---

## 11. Acceptance Criteria

- [ ] `GET /api/v1/staff/job-categories` returns exactly 21 categories each with `key`, `label_ja`, `label_en`, and `icon`.
- [ ] `POST /api/v1/staff/profile` with any of the 21 valid category keys succeeds.
- [ ] `POST /api/v1/staff/profile` with an unknown category key returns `422 validation_error`.
- [ ] The `JobCategoryDropdown` widget renders all 21 Japanese labels loaded from the API.
- [ ] The dropdown shows a `CircularProgressIndicator` while categories are loading.
- [ ] Submitting the staff registration form without selecting a category shows inline error `"Please select a job category."` and makes no API call.
- [ ] `StaffService.getJobCategories()` makes only one network request per session; subsequent calls use the cache.
- [ ] All API endpoints return the standard error envelope for non-2xx responses.
- [ ] The category chip row on the Home screen correctly filters staff by the selected category key.
