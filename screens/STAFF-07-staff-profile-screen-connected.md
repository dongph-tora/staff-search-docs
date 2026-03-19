# [STAFF-07] Staff Profile Screen Connected to DB

| Field | Value |
|---|---|
| **Type** | Screen |
| **Priority** | P1 MVP |
| **Estimate** | 1 day manual / 0.5 days vibe coding |
| **Dependencies** | STAFF-01 — `StaffService.getProfile(userID)`, `StaffProvider`, `StaffProfileResponse` (Sections 4.3, 4.4, 5.5). STAFF-03 — `JobCategory` model for displaying category label (Section 4.1). STAFF-05 — `portfolioPhotos` on `StaffProfile` model (Section 4.1). |
| **Platforms** | iOS · Android · Web |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format (Section 2), `ApiClient` (Section 6), navigation (Section 9), error display (Section 10). |

---

## 1. Overview

This ticket replaces all mock/hardcoded data in `StaffProfileScreen` with live data fetched from the backend. It covers both the public view (any user viewing a staff's profile) and the owner view (the staff viewing their own profile with edit options). The screen fetches `StaffProfileResponse` via `StaffService.getProfile(userID)`, renders the staff's name, avatar, bio, job category, stats (rating, review count, followers), portfolio photo grid, and action buttons (Follow, Book, Tip). For the owner, Edit and Edit Portfolio buttons are shown instead.

### End-to-end flow

    [Flutter App]                    [Go API]                 [PostgreSQL]
         │                               │                           │
         │── on screen open              │                           │
         │── GET /api/v1/staff/:userID ─►│                           │
         │                               │── SELECT staff_profiles   │
         │                               │── JOIN users ────────────►│
         │                               │── SELECT portfolio_photos ►│
         │◄── 200 {staff_profile} ───────│                           │
         │── render all fields           │                           │
         │── if own profile: show edit   │                           │
         │   buttons                     │                           │

---

## 2. User Flows

### 2.1 Happy Path — Viewing Another Staff's Profile

1. User taps a staff card in the feed, search results, or ranking screen.
2. App navigates to `StaffProfileScreen` with `userID` as a route argument.
3. Screen shows `CircularProgressIndicator` while loading.
4. `StaffService.getProfile(userID)` is called.
5. On `200` response, all sections render: header (avatar, name, staff number, job title, category badge), stats row (rating, reviews, followers), bio, portfolio grid, action buttons (Follow + Book Now + Tip).
6. Tapping "Follow" updates follower state (SOCIAL-01).
7. Tapping "Book Now" navigates to the booking flow (BOOK-01).
8. Tapping "Tip" navigates to the tip flow (PAY-07).

### 2.2 Happy Path — Owner Viewing Their Own Profile

1. Staff user navigates to their own profile (via the Profile tab).
2. App compares screen `userID` with `AuthProvider.currentUser.id`.
3. If they match, the screen renders in owner mode: Follow/Tip/Book buttons are replaced by "Edit Profile" and "Edit Portfolio" buttons.
4. Tapping "Edit Profile" navigates to `StaffProfileEditScreen` (STAFF-01).
5. Tapping "Edit Portfolio" navigates to `PortfolioEditScreen` (STAFF-05).

### 2.3 Error Cases

| Scenario | Behaviour |
|---|---|
| Staff profile not found (404) | Full-screen empty state with icon and text: `"This staff profile could not be found."` and a back button |
| Network error on load | Full-screen error state with text: `"Unable to connect. Check your network and retry."` and a Retry button that re-fetches the profile |
| Server error (500) | Full-screen error state: `"Something went wrong. Please try again later."` with Retry button |
| Profile has no portfolio photos | Portfolio section shows placeholder text: `"No portfolio photos yet."` |
| Profile has no bio | Bio section is hidden entirely |

---

## 3. UI / Screen

### 3.1 Staff Profile Screen (`staff_profile_screen.dart`)

> Current location: `lib/screens/staff/staff_profile_screen.dart`
> Status: **Modify existing** — replace all mock data with live API calls

    ┌─────────────────────────────────────────┐
    │  ←  [Staff Name]          [⋮ menu]    │
    ├─────────────────────────────────────────┤
    │                                         │
    │  [Avatar 80px]  Staff ID: #123456       │
    │  Nail Artist  [ネイリスト badge]        │
    │                                         │
    │  ★ 4.8  ·  120 reviews  ·  1.2k fans  │
    │  📍 Tokyo, Shibuya                      │
    │                                         │
    │  ─── About ──────────────────────       │
    │  [Bio text, max 3 lines with "See more"]│
    │                                         │
    │  ─── Portfolio ──────────────────────  │
    │  ┌─────┐ ┌─────┐ ┌─────┐              │
    │  │     │ │     │ │     │              │
    │  └─────┘ └─────┘ └─────┘              │
    │  ┌─────┐ ┌─────┐ ┌─────┐              │
    │  │     │ │     │ │     │              │
    │  └─────┘ └─────┘ └─────┘              │
    │                                         │
    │  [  Follow  ]  [ Book Now ]  [ Tip ]   │
    └─────────────────────────────────────────┘

    ─── Owner mode (own profile) ───
    │  [ Edit Profile ]  [ Edit Portfolio ] │

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| Avatar | `CircleAvatar` with `CachedNetworkImage` | Falls back to initials if `avatar_url` is null |
| Staff ID label | `Text` | Displays `#` + `staff_number` |
| Job category badge | `Chip` | Displays `label_ja` from STAFF-03 category list; uses category-specific color |
| Stats row | `Row` of 3 `Column` widgets | Rating (star icon + decimal), review count, followers count |
| Bio section | `Text` with "See more" toggle | Hidden if bio is null or empty |
| Portfolio grid | `GridView.builder` with 3 columns | Each photo is a `CachedNetworkImage` tappable for fullscreen view |
| Follow button | `OutlinedButton` | Shown in visitor mode only; toggles follow state |
| Book Now button | `ElevatedButton` | Shown in visitor mode only; disabled if `accept_bookings = false` |
| Tip button | `TextButton` | Shown in visitor mode only |
| Edit Profile button | `ElevatedButton` | Shown in owner mode only |
| Edit Portfolio button | `OutlinedButton` | Shown in owner mode only |
| Loading state | `CircularProgressIndicator` centered | Shown while initial fetch is in progress |
| Error state | Full-screen `Column` with icon, message, Retry button | Shown on 404, 5xx, or network error |

Changes required:
- Remove all hardcoded mock `Staff` objects.
- Add `userID` route argument and pass it to `StaffService.getProfile(userID)` on `initState`.
- Add `isOwner` computed property: `AuthProvider.currentUser?.id == userID`.
- Render Follow/Book/Tip buttons conditionally based on `isOwner`.
- Render Edit/Edit Portfolio buttons conditionally based on `isOwner`.
- Replace static portfolio grid with `StaffProfile.portfolioPhotos` list.
- Add full-screen loading and error states.
- Use `CachedNetworkImage` for avatar and portfolio photos.

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/widgets/staff_stats_row.dart` | Reusable row widget displaying rating, review count, and follower count |
| `lib/widgets/portfolio_grid.dart` | Reusable 3-column grid widget for portfolio photos; tappable photos open fullscreen viewer |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/screens/staff/staff_profile_screen.dart` | Full rewrite of data layer: remove mock data, add API call, add loading/error states, add owner/visitor conditional rendering |
| `lib/main.dart` | Ensure `StaffProfileScreen` route accepts `userID` as a named route argument |

### 4.3 StaffProfileScreen Responsibilities

`StaffProfileScreen` receives a `userID` argument via the route. On `initState`, it calls `StaffService.getProfile(userID)` and stores the result in local state. The `isOwner` boolean is computed by comparing `userID` with `context.read<AuthProvider>().currentUser?.id`. The screen rebuilds on auth state changes via `context.watch<AuthProvider>()`. All API calls are made through `StaffService`; no `StaffProvider` state mutation happens here (the provider is for the owner's own profile data mutated by STAFF-01 and STAFF-05 flows).

### 4.4 CachedNetworkImage Dependency

Portfolio photos and the avatar should use the `cached_network_image` package for efficient image loading and caching. If not already in `pubspec.yaml`, it must be added as part of this ticket.

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

No new endpoints are introduced in this ticket. It uses `GET /api/v1/staff/:userID` defined in STAFF-01 Section 5.5 and `GET /api/v1/staff/me` defined in STAFF-01 Section 5.4.

This ticket ensures both endpoints return the complete `StaffProfileResponse` including:
- Nested `user` object with `name` and `avatar_url`
- Full `portfolio_photos` array sorted by `display_order`
- Computed `is_online` field based on `is_available`

Not applicable for this ticket beyond what is specified in STAFF-01 Section 5.5.

---

## 6. Database

Not applicable for this ticket. No schema changes are required.

---

## 7. Configuration

Not applicable for this ticket. No new environment variables are required.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | GET /staff/:userID — valid staff user | `200`, all profile fields populated, `portfolio_photos` sorted by `display_order` |
| 2 | GET /staff/:userID — user exists but has no staff profile | `404 not_found` |
| 3 | GET /staff/:userID — non-existent userID | `404 not_found` |
| 4 | GET /staff/:userID — response does not include email or password_hash | Sensitive fields absent from response body |
| 5 | GET /staff/me — authenticated staff | `200`, own profile returned |
| 6 | All endpoints without JWT | `401 unauthorized` |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Screen opened for another staff | `CircularProgressIndicator` shown; on load, all profile data rendered; Follow/Book/Tip buttons visible |
| 2 | Screen opened for own profile | `CircularProgressIndicator` shown; on load, Edit Profile and Edit Portfolio buttons visible; Follow/Book/Tip buttons hidden |
| 3 | Profile not found (404) | Full-screen empty state with message `"This staff profile could not be found."` |
| 4 | Network error on load | Full-screen error state with Retry button; red SnackBar: `"Unable to connect. Check your network and retry."` |
| 5 | Retry button tapped | API call is re-made; loading state shown again |
| 6 | Profile has no bio | Bio section is completely hidden |
| 7 | Profile has no portfolio photos | Portfolio section shows `"No portfolio photos yet."` |
| 8 | Portfolio photo tapped | Fullscreen photo viewer opens |
| 9 | No mock data visible | All profile data comes from API; no hardcoded names or images remain |

---

## 9. Security Checklist

- [ ] `email`, `password_hash`, `google_id`, `apple_id` are never included in `StaffProfileResponse` for any user's public profile.
- [ ] The `isOwner` check in Flutter uses `AuthProvider.currentUser.id` from the JWT claims, not a field from the profile response.
- [ ] No API call is made with the userID from the URL directly as an auth credential.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Screen | `lib/screens/staff/staff_profile_screen.dart` | Modify — remove mock data, add API call, loading/error states, owner/visitor modes |
| Flutter — Widget | `lib/widgets/staff_stats_row.dart` | Create — stats row widget |
| Flutter — Widget | `lib/widgets/portfolio_grid.dart` | Create — portfolio photo grid widget |
| Flutter — App | `lib/main.dart` | Modify — update StaffProfileScreen route to accept userID argument |
| Flutter — Config | `pubspec.yaml` | Modify — add cached_network_image if not already present |

---

## 11. Acceptance Criteria

- [ ] `StaffProfileScreen` fetches data from `GET /api/v1/staff/:userID` on load; no mock data is used.
- [ ] The screen shows `CircularProgressIndicator` while the API call is in progress.
- [ ] All profile fields (name, staff number, job title, category badge, rating, review count, followers, bio, portfolio photos) are populated from the API response.
- [ ] When viewing another user's profile, Follow, Book Now, and Tip buttons are visible.
- [ ] When viewing own profile (`userID == AuthProvider.currentUser.id`), Edit Profile and Edit Portfolio buttons are visible; Follow, Book Now, and Tip are hidden.
- [ ] A 404 response renders a full-screen empty state with text `"This staff profile could not be found."`.
- [ ] A network error renders a full-screen error state with a Retry button.
- [ ] Portfolio photos are displayed in a 3-column grid, sorted by `display_order`.
- [ ] Portfolio photos use `CachedNetworkImage` for efficient loading.
- [ ] `email`, `password_hash`, and other sensitive fields are never displayed or logged on the client.
- [ ] All API endpoints return `401` when called without a valid JWT.
