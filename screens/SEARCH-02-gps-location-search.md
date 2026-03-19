# SEARCH-02 — GPS / Location-based Search

| Field | Value |
|---|---|
| **Type** | Feature |
| **Priority** | P1 MVP |
| **Estimate** | 1.5 days manual / 0.5 days vibe coding |
| **Dependencies** | `STAFF-01-staff-profile-crud.md` Sections 5.2–5.4 — `StaffProfileResponse` DTO and `UpdateStaffProfileRequest` defined there; `DB-01-database-schema.md` — `staff_profiles` table exists |
| **Platforms** | iOS · Android |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format, JWT, ApiClient, AuthProvider, navigation, and token storage rules. |

---

## 1. Overview

This feature adds GPS-based distance calculation to the staff discovery experience. Staff members can optionally store their latitude and longitude in their profile. When a customer opens the home screen on a device with location permission granted, the app reads their current position and computes the distance from each staff card that has coordinates. A distance label is shown on those cards. Staff with no coordinates are included in results without a label — they are never excluded. The Filter Settings screen already exposes a maximum-distance slider (1–100 km, default 50 km) stored in SharedPreferences; once the backend returns coordinates, that filter becomes active without further Flutter changes.

The backend changes are additive: two nullable columns are added to `staff_profiles`, the `UpdateStaffProfileRequest` accepts optional latitude and longitude, and `StaffProfileResponse` includes both fields. No new endpoints are required for MVP; all distance calculation and filtering happen client-side after the staff list is fetched.

### End-to-end flow

```
[Flutter App]                      [Go API]                [PostgreSQL]
     │                                  │                        │
     │── GET /api/v1/staff ────────────►│                        │
     │                                  │── SELECT staff_profiles│
     │                                  │   (incl. lat, lng) ───►│
     │                                  │◄── rows ───────────────│
     │◄── 200 StaffProfileResponse[] ───│                        │
     │    (latitude, longitude per row) │                        │
     │                                  │                        │
     │── geolocator.getCurrentPosition()│                        │
     │◄── Position (user lat/lng)       │                        │
     │                                  │                        │
     │── _applyFilters()                │                        │
     │   calculates distance per staff  │                        │
     │   hides staff beyond maxDistance │                        │
     │   (staff with no coords shown)   │                        │
     │── rebuild StaffCard with         │                        │
     │   distance label where available │                        │
```

---

## 2. User Flows

### 2.1 Happy Path — User views staff list with distances

1. User opens the home screen on a device with iOS or Android location permission already granted.
2. The app calls `geolocator.getCurrentPosition()` via `LocationService`. The result is stored as `_currentPosition`.
3. The app calls `GET /api/v1/staff`, which returns a list of `StaffProfileResponse` objects each containing `latitude` and `longitude` (null if the staff member has not set their location).
4. `Staff.fromApiResponse()` maps `latitude` and `longitude` from the response onto each `Staff` object.
5. `_applyFilters()` iterates the staff list. For each staff with non-null coordinates, it calls `LocationService.calculateDistance()` and stores the result in `staff.distance`. Staff beyond the stored `filter_max_distance` value are hidden.
6. Staff with null coordinates pass through the distance filter unconditionally; their `distance` field remains null.
7. The home screen renders each visible `StaffCard`. Cards with a non-null `distance` show a distance label (for example, "3.2 km away"). Cards with a null `distance` show no distance label.

### 2.2 Happy Path — Staff member sets their location via profile update

1. A staff user sends `PATCH /api/v1/staff/profile` with `latitude` and `longitude` included in the request body.
2. The handler validates the range: `latitude` must be between -90 and 90, `longitude` must be between -180 and 180.
3. The service applies the update to the staff profile row.
4. The handler returns `200` with the updated `StaffProfileResponse` including the new coordinate values.
5. Subsequent calls to `GET /api/v1/staff` include those coordinates in the response, making this staff member visible on distance-aware home screens.

### 2.3 Error Cases

| Scenario | Behaviour |
|---|---|
| Location permission denied by user | `_currentPosition` stays null; distance filter is skipped entirely; all staff appear without distance labels; no error is shown to the user |
| Location permission permanently denied (iOS/Android) | Same as above — silent degradation; the filter slider in Filter Settings is still visible but has no effect until permission is granted |
| `geolocator` throws a timeout or platform exception | App catches the exception, leaves `_currentPosition` as null, and proceeds without distance data |
| `PATCH /api/v1/staff/profile` sent with `latitude` out of range | `422` response; red SnackBar: `"Latitude must be between -90 and 90."` |
| `PATCH /api/v1/staff/profile` sent with `longitude` out of range | `422` response; red SnackBar: `"Longitude must be between -180 and 180."` |
| Network error on staff list fetch | Red SnackBar: `"Unable to connect. Check your network and retry."` |
| Server error (500) on staff list fetch | Red SnackBar: `"Something went wrong. Please try again later."` |

---

## 3. UI / Screen

### 3.1 Home Screen (`home_screen.dart`)

> Current location: `lib/screens/home_screen.dart`
> Status: **Modify existing**

```
┌─────────────────────────────────────────┐
│  スタッフサーチ                          │  ← AppBar
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  [Staff Avatar]  Name             │  │
│  │  Job Title       ★ 4.8  (120)     │  │
│  │  📍 3.2 km away   ← distance label│  │  ← shown only if staff.distance != null
│  │  [Online badge if available]      │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  [Staff Avatar]  Name             │  │
│  │  Job Title       ★ 4.5  (88)      │  │
│  │  (no distance — staff has no GPS) │  │  ← no label, no blank space
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| Distance label | `Text` inside `StaffCard` | Displays `"X.X km away"` where X.X is `staff.distance` rounded to one decimal place; hidden entirely when `staff.distance` is null |
| Filter Settings slider — Max Distance | `Slider` in `FilterSettingsScreen` | Already implemented; reads and writes `filter_max_distance` in SharedPreferences; range 1–100 km; default 50 km |

Changes required:

- `Staff.fromApiResponse()` currently does not map `latitude` or `longitude` from the JSON response. Add mapping for both fields so that `staff.latitude` and `staff.longitude` are populated when the API returns them.
- No other Flutter UI changes are required; `_applyFilters()` and the distance label in the staff card are already implemented.

### 3.2 Filter Settings Screen (`filter_settings_screen.dart`)

> Current location: `lib/screens/filter_settings_screen.dart`
> Status: **No changes required**

The slider and its SharedPreferences persistence are already implemented. This screen is listed here for reference only.

---

## 4. Frontend — Flutter

### 4.1 Files to Create

Not applicable for this ticket.

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/models/staff.dart` | In `Staff.fromApiResponse()`, add mapping for `latitude` from `json['latitude']` as `double?` and `longitude` from `json['longitude']` as `double?`; both default to null when absent or null in the response |

### 4.3 Staff Model Responsibilities

`Staff.fromApiResponse()` is the only constructor used when building `Staff` objects from API data. The two new fields — `latitude` and `longitude` — are already declared as `double?` on the class and accepted in the named constructor. The factory method currently omits these fields, so the values are always null regardless of what the API returns. The change in this ticket closes that gap by reading `json['latitude']` and `json['longitude']` from the decoded response map and casting them to `double?` using the null-aware numeric cast pattern already in use for other nullable fields in the same factory.

No changes are needed to `Staff.fromJson()`, `Staff.toJson()`, or any provider or service that wraps the model.

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| `PATCH` | `/api/v1/staff/profile` | Update own staff profile — now accepts optional `latitude` and `longitude` | Yes (staff role) |
| `GET` | `/api/v1/staff` | List all staff profiles — response now includes `latitude` and `longitude` per record | Yes |

### 5.2 PATCH /api/v1/staff/profile

This endpoint is defined in STAFF-01. This ticket extends it by adding two optional fields to the request and response.

**Request body additions:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `latitude` | float64 (pointer) | No | Must be between -90.0 and 90.0 inclusive when provided; null clears the stored value |
| `longitude` | float64 (pointer) | No | Must be between -180.0 and 180.0 inclusive when provided; null clears the stored value |

All other request fields are unchanged from STAFF-01 Section 5.3.

**Response `200`:**

All fields from STAFF-01 Section 5.3, plus:

| Field | Type | Description |
|---|---|---|
| `latitude` | float64 or null | Staff member's stored latitude, null if not set |
| `longitude` | float64 or null | Staff member's stored longitude, null if not set |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed request body |
| `401` | `unauthorized` | Missing or invalid JWT |
| `403` | `forbidden` | Authenticated user does not have the staff role |
| `404` | `not_found` | Staff profile does not exist for the authenticated user |
| `422` | `validation_error` | `latitude` is outside the range -90 to 90, or `longitude` is outside the range -180 to 180 |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities** (additions to existing STAFF-01 handler steps):

1. Parse request body into `UpdateStaffProfileRequest` as before.
2. If `latitude` is non-nil, check that the value is between -90.0 and 90.0. If not, return `422` with error code `validation_error` and message `"Latitude must be between -90 and 90."`.
3. If `longitude` is non-nil, check that the value is between -180.0 and 180.0. If not, return `422` with error code `validation_error` and message `"Longitude must be between -180 and 180."`.
4. Delegate to the staff service with the validated request.
5. Return `200` with the updated `StaffProfileResponse` including the new coordinate fields.

### 5.3 GET /api/v1/staff

This endpoint was added as part of the staff list work. This ticket extends its response shape.

**Request:** No body. No query parameters added by this ticket.

**Response `200` — array of `StaffProfileResponse`:**

All existing fields, plus these two additions per item:

| Field | Type | Description |
|---|---|---|
| `latitude` | float64 or null | Staff member's stored latitude; null if not set |
| `longitude` | float64 or null | Staff member's stored longitude; null if not set |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `401` | `unauthorized` | Missing or invalid JWT |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities:**

1. Verify JWT is present via `JWTMiddleware` (applied at the group level).
2. Call the staff service to retrieve all staff profiles.
3. Map each profile to `StaffProfileResponse` via `ToStaffProfileResponse()`, which now includes `latitude` and `longitude`.
4. Return `200` with the array.

### 5.4 Service Responsibilities

`StaffService`:

- `UpdateProfile(ctx, userID string, req UpdateStaffProfileRequest) (StaffProfile, error)` — existing method; now also applies the `Latitude` and `Longitude` pointer fields from `req` to the profile before saving.
- `ListStaff(ctx context.Context) ([]StaffProfile, error)` — existing method; no change needed beyond ensuring GORM loads the `latitude` and `longitude` columns, which will happen automatically after the migration adds those columns to the table.

---

## 6. Database

### 6.1 Migration

**File:** `migrations/20260311000000_add_lat_lng_to_staff_profiles.up.sql`
**Reverse:** `migrations/20260311000000_add_lat_lng_to_staff_profiles.down.sql`

Altered table — `staff_profiles`:

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `latitude` | `NUMERIC(10,7)` | Yes | NULL | Geographic latitude of the staff member's primary work location; -90.0 to 90.0 |
| `longitude` | `NUMERIC(10,7)` | Yes | NULL | Geographic longitude of the staff member's primary work location; -180.0 to 180.0 |

New indexes: None. These columns are not indexed for MVP. Spatial indexing using PostGIS is deferred to SEARCH-05 (Map view integration).

The up migration uses `ALTER TABLE staff_profiles ADD COLUMN IF NOT EXISTS` for both columns to ensure idempotency. The down migration uses `ALTER TABLE staff_profiles DROP COLUMN IF EXISTS` to fully reverse the change.

### 6.2 GORM Model Change

The `StaffProfile` struct in `internal/model/staff_profile.go` requires two new fields:

| Go Field | GORM Tag | JSON Tag | Description |
|---|---|---|---|
| `Latitude` | `gorm:"type:numeric(10,7)"` | `json:"latitude,omitempty"` | Nullable float64 pointer |
| `Longitude` | `gorm:"type:numeric(10,7)"` | `json:"longitude,omitempty"` | Nullable float64 pointer |

Both fields are `*float64` (pointer type) so that null in the database is represented as nil in Go and omitted from JSON when nil.

---

## 7. Configuration

Not applicable for this ticket.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | `PATCH /api/v1/staff/profile` with valid `latitude: 35.6762` and `longitude: 139.6503` | `200`; response contains `latitude: 35.6762`, `longitude: 139.6503` |
| 2 | `PATCH /api/v1/staff/profile` with `latitude: 91.0` | `422 validation_error`; message `"Latitude must be between -90 and 90."` |
| 3 | `PATCH /api/v1/staff/profile` with `longitude: -181.0` | `422 validation_error`; message `"Longitude must be between -180 and 180."` |
| 4 | `PATCH /api/v1/staff/profile` with `latitude: null` (explicitly null) | `200`; `latitude` in response is null; database value is set to NULL |
| 5 | `PATCH /api/v1/staff/profile` without `latitude` or `longitude` fields | `200`; existing coordinate values are unchanged |
| 6 | `GET /api/v1/staff` when some profiles have coordinates and some do not | `200`; records with coordinates include non-null `latitude` and `longitude`; records without have null values |
| 7 | `GET /api/v1/staff` with no JWT | `401 unauthorized` |
| 8 | `PATCH /api/v1/staff/profile` with `latitude: -90.0` (boundary) | `200`; boundary value accepted and stored |
| 9 | `PATCH /api/v1/staff/profile` with `longitude: 180.0` (boundary) | `200`; boundary value accepted and stored |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | API returns staff with `latitude: 35.6762` and `longitude: 139.6503`; user is 3.2 km away | `Staff.fromApiResponse()` sets `latitude` and `longitude`; after `_applyFilters()`, `staff.distance` is approximately 3.2; staff card shows distance label `"3.2 km away"` |
| 2 | API returns staff with `latitude: null` and `longitude: null` | `Staff.fromApiResponse()` sets both to null; `_applyFilters()` skips distance check; staff appears in list without distance label |
| 3 | Device location permission denied | `_currentPosition` is null; `_applyFilters()` skips all distance calculations; all staff appear without distance labels; no SnackBar is shown |
| 4 | Staff is 60 km away; `filter_max_distance` is 50 km; staff has valid coordinates | Staff is filtered out and does not appear in the list |
| 5 | Staff is 40 km away; `filter_max_distance` is 50 km | Staff appears in the list with a distance label |
| 6 | API call fails with network error | Red SnackBar: `"Unable to connect. Check your network and retry."` |

---

## 9. Security Checklist

- [ ] Coordinate values are validated server-side in the handler before reaching the service layer — client-side validation alone is not sufficient
- [ ] No sensitive data is returned in error messages — validation errors include only the field constraint description
- [ ] Location data is optional — storing coordinates is never required and never implied as a prerequisite for app functionality
- [ ] The `latitude` and `longitude` fields are publicly visible in `GET /api/v1/staff` responses — staff members must be informed via the profile edit screen (STAFF-06 scope) that their coordinates will be shared with app users
- [ ] GORM pointer types (`*float64`) ensure null is correctly round-tripped through the ORM without defaulting to zero, which would be a valid coordinate (0, 0 is a real geographic location in the Gulf of Guinea)

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Model | `lib/models/staff.dart` | Modify — add `latitude` and `longitude` mapping in `Staff.fromApiResponse()` |
| Go — DTO | `internal/dto/staff_dto.go` | Modify — add `Latitude *float64` and `Longitude *float64` to `UpdateStaffProfileRequest`; add same fields to `StaffProfileResponse`; update `ToStaffProfileResponse()` to map both fields |
| Go — Model | `internal/model/staff_profile.go` | Modify — add `Latitude *float64` and `Longitude *float64` fields with GORM tags `type:numeric(10,7)` |
| Go — Handler | `internal/handler/staff_handler.go` | Modify — add range validation for `latitude` (-90 to 90) and `longitude` (-180 to 180) before calling the update service method |
| Go — Service | `internal/service/staff_service.go` | Modify — `UpdateProfile()`: apply `req.Latitude` and `req.Longitude` to the profile struct before saving |
| DB Migration | `migrations/20260311000000_add_lat_lng_to_staff_profiles.up.sql` | Create — `ALTER TABLE staff_profiles ADD COLUMN IF NOT EXISTS latitude NUMERIC(10,7) NULL, ADD COLUMN IF NOT EXISTS longitude NUMERIC(10,7) NULL` |
| DB Migration | `migrations/20260311000000_add_lat_lng_to_staff_profiles.down.sql` | Create — `ALTER TABLE staff_profiles DROP COLUMN IF EXISTS latitude, DROP COLUMN IF EXISTS longitude` |

---

## 11. Acceptance Criteria

- [ ] `GET /api/v1/staff` returns `latitude` and `longitude` for every staff record; values are the stored coordinates or null when not set
- [ ] `PATCH /api/v1/staff/profile` with a valid `latitude` and `longitude` stores both values and returns them in the `200` response
- [ ] `PATCH /api/v1/staff/profile` with `latitude` outside -90 to 90 returns `422` with message `"Latitude must be between -90 and 90."`
- [ ] `PATCH /api/v1/staff/profile` with `longitude` outside -180 to 180 returns `422` with message `"Longitude must be between -180 and 180."`
- [ ] `PATCH /api/v1/staff/profile` without `latitude` or `longitude` fields leaves existing coordinate values unchanged
- [ ] `Staff.fromApiResponse()` correctly maps non-null `latitude` and `longitude` from the API response onto the Flutter `Staff` object
- [ ] After `_applyFilters()`, a staff member with coordinates within the `filter_max_distance` threshold displays a distance label on their home screen card
- [ ] After `_applyFilters()`, a staff member whose distance exceeds `filter_max_distance` is excluded from the visible list
- [ ] A staff member with null coordinates (not set) always appears in the home screen list regardless of the `filter_max_distance` setting, with no distance label
- [ ] When device location permission is denied, the home screen loads normally without errors and all staff appear without distance labels
- [ ] The `staff_profiles` table `latitude` and `longitude` columns are `NUMERIC(10,7) NULL` with no default; existing rows retain NULL values after migration
- [ ] All API endpoints return the standard error envelope `{ "error": "...", "message": "..." }` for all non-2xx responses
