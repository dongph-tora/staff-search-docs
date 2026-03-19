# [FEED-02] TikTok-Style Vertical Swipe Feed

| Field | Value |
|---|---|
| **Type** | Screen |
| **Priority** | P1 MVP |
| **Estimate** | 2 days manual / 0.75 days vibe coding |
| **Dependencies** | FEED-01 — `PostService.getFeed`, `Post` model, `GET /api/v1/posts/feed` endpoint defined in Sections 4.3, 5.3, 5.4. AUTH-01 — `AuthProvider` in Sections 4.1–4.4. STAFF-01 — `StaffProfile` link from `Post.author` for profile navigation. STAFF-03 — `JobCategoryChips` for category filtering (Section 3.2). |
| **Platforms** | iOS · Android · Web |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for `ApiClient` (Section 6), navigation (Section 9), error display (Section 10). |

---

## 1. Overview

This ticket implements the main home feed: a full-screen, vertically scrollable `PageView` that snaps one post at a time to fill the screen, similar to TikTok. Each post item renders either a full-screen image or a looping video player, with an overlay showing the author name, caption, action buttons (like, comment, share, tip), and a staff profile thumbnail. The feed uses cursor-based pagination to load more posts as the user scrolls toward the end. Category filter chips (STAFF-03) allow filtering by job category.

### End-to-end flow — Load Feed

    [Flutter App]                    [Go API]                 [PostgreSQL]
         │                               │                           │
         │── on Home tab open            │                           │
         │── GET /api/v1/posts/feed ────►│                           │
         │   ?limit=10                   │── SELECT posts + users    │
         │                               │   ORDER BY id DESC ──────►│
         │◄── 200 {posts, next_cursor}───│                           │
         │── render first post full-     │                           │
         │   screen in PageView          │                           │
         │                               │                           │
         │── (user swipes near end)      │                           │
         │── GET /posts/feed?cursor=X── ►│                           │
         │◄── 200 {more posts} ──────────│                           │

---

## 2. User Flows

### 2.1 Happy Path — Browse Feed

1. User opens the app and navigates to the Home tab.
2. `FeedProvider` fetches the first page of posts via `PostService.getFeed(cursor: null, limit: 10)`.
3. A `PageView.builder` renders posts vertically. The first post fills the screen.
4. If the post has `media_type = "video"`, the video begins playing automatically with sound muted; tapping the post toggles mute.
5. If the post has `media_type = "image"`, the image fills the screen.
6. User swipes up to see the next post. The previous post's video pauses; the new post's video auto-plays.
7. When 2 posts remain before the end, the feed pre-fetches the next page using the `next_cursor`.
8. If `has_more = false`, no more fetches are made; swiping on the last post has no effect.

### 2.2 Happy Path — Filter by Category

1. A horizontal chip row is visible at the top of the feed (from STAFF-03 `JobCategoryChips`).
2. User taps a category chip (e.g., "ネイリスト").
3. `FeedProvider` resets the post list and fetches `GET /api/v1/posts/feed?category=nail_art`.
4. Feed renders posts filtered to nail artists only.
5. Tapping "すべて" (All) removes the filter and reloads the unfiltered feed.

### 2.3 Happy Path — Navigate to Staff Profile from Feed

1. User taps the author avatar or name overlay on a feed post.
2. App navigates to `StaffProfileScreen` with the post's `author_id`.

### 2.4 Error Cases

| Scenario | Behaviour |
|---|---|
| Network error on initial load | Full-screen error state with icon, text `"Unable to load feed. Check your network."`, and Retry button |
| Network error on next-page load | Red SnackBar: `"Unable to load more posts."` Feed remains on current post |
| Server error (500) on initial load | Full-screen error state: `"Something went wrong. Please try again later."` with Retry button |
| Feed is empty (no posts exist) | Full-screen empty state: `"No posts yet. Check back soon!"` |
| Video fails to load | Video area shows a grey placeholder with a play icon; no crash |
| Image fails to load | `CachedNetworkImage` error widget shown |

---

## 3. UI / Screen

### 3.1 Feed Screen (`feed_screen.dart`)

> Current location: `lib/screens/home/home_screen.dart` or `lib/screens/feed/feed_screen.dart`
> Status: **Modify existing** — replace mock post list with live `FeedProvider` data and `PageView`

    ┌─────────────────────────────────────────┐
    │  [Category chip row, horizontal scroll] │  ← overlaid at top
    │                                         │
    │                                         │
    │  [Full-screen image OR video player]    │
    │                                         │
    │                                         │
    │  ─── bottom overlay ──────────────────  │
    │  [Avatar 40px]  @authorName             │  ← left side
    │  [Caption text, 2 lines + "more"]       │
    │                                  [❤️ ]  │  ← right action column
    │                                  [💬 ]  │
    │                                  [🔗 ]  │
    │                                  [💰 ]  │
    │  [Staff category badge]                 │
    └─────────────────────────────────────────┘
    (swipe up = next post, swipe down = prev)

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| Category chip row | `JobCategoryChips` from STAFF-03 | Sticky at top with slight transparency; single selection; resets feed on selection change |
| `PageView.builder` | `PageView` with `scrollDirection: Axis.vertical` | Full viewport height; `PageSnapping = true`; one post per page |
| Video player | `VideoPlayer` widget with `video_player` package | Auto-plays on page entry (muted); pauses on page exit; looping |
| Image display | `CachedNetworkImage` with `BoxFit.cover` | Fills the viewport |
| Author overlay | `Positioned` at bottom-left | Avatar (40px circle), display name, caption (2 lines, expandable) |
| Action column | `Positioned` at right | Like button with count, comment button with count, share button, tip button |
| Like button | `IconButton` | Filled heart when liked; animates on tap; calls FEED-03 like endpoint (stubbed for now — increments count locally) |
| Comment button | `IconButton` | Navigates to comment sheet (FEED-03) |
| Tip button | `IconButton` with coin icon | Navigates to tip flow (PAY-07) |
| Mute toggle | `GestureDetector` on video area | Tapping the video area toggles mute; shows mute icon briefly |
| Loading indicator | `CircularProgressIndicator` centered | Shown while initial feed loads |
| Pre-fetch trigger | Detected via `PageController.addListener` | When current page index = `posts.length - 2`, trigger next page fetch |

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/screens/feed/feed_screen.dart` | Main vertical swipe feed using `PageView.builder` |
| `lib/providers/feed_provider.dart` | `ChangeNotifier` managing posts list, cursor, loading state, category filter |
| `lib/widgets/feed_post_item.dart` | Single full-screen post widget — image or video, overlays, action column |
| `lib/widgets/video_feed_player.dart` | Video player widget for feed posts — auto-play, loop, mute toggle, lifecycle management |

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/screens/home/home_screen.dart` | Replace mock feed with `FeedScreen` widget; register `FeedProvider` |
| `lib/main.dart` | Register `FeedProvider` in `MultiProvider` |
| `pubspec.yaml` | Confirm `video_player: ^2.9.2` and `cached_network_image` are present (added in FEED-01 / STAFF-07) |

### 4.3 FeedProvider Responsibilities

`FeedProvider` (ChangeNotifier) owns:
- `posts` (`List<Post>`) — the currently loaded list of posts.
- `isLoading` (`bool`) — true during initial load; false otherwise.
- `isLoadingMore` (`bool`) — true during next-page fetch; shown as a spinner at the bottom of the list.
- `hasMore` (`bool`) — false when the API returns `has_more = false`.
- `nextCursor` (`String?`) — cursor for the next API call.
- `selectedCategory` (`String?`) — active job category filter; null = unfiltered.
- `error` (`String?`) — error message for initial load failure.

Methods:
- `loadFeed()` — Resets all state, fetches first page via `PostService.getFeed`.
- `loadMore()` — Appends next page if `hasMore = true` and not already loading.
- `setCategory(String? category)` — Sets `selectedCategory`, calls `loadFeed()` to reload.

`FeedProvider.loadFeed()` is called once on app start and whenever the Home tab becomes active (using `RouteObserver` or `WidgetsBindingObserver`).

### 4.4 Video Lifecycle Management

`VideoFeedPlayer` creates and disposes a `VideoPlayerController` per post item. Video auto-plays when the post enters the viewport (detected by `PageController.page` matching the item's index). Video pauses when the post leaves the viewport. All controllers are disposed when the widget is removed from the tree. Only one video plays at a time.

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

No new endpoints are introduced in this ticket. The feed is powered by `GET /api/v1/posts/feed` defined in FEED-01 Section 5.3. This ticket consumes that endpoint from the Flutter side.

The feed endpoint must return posts in reverse-chronological order (newest first) with cursor pagination. For MVP, no personalization algorithm is applied — posts are ordered by `created_at DESC`. Each `PostResponse` includes:
- `author` object with `id`, `name`, `avatar_url`
- `staff_category` string (the author's `job_category` from `staff_profiles` if they are a staff user, otherwise null)
- `is_liked` bool (whether the requesting user has liked this post)
- `likes_count` and `comments_count` integers

If the author is a staff user, the `staff_profile_id` is included so the client can navigate to their profile.

Not applicable for this ticket beyond what is specified in FEED-01 Section 5.3.

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
| 1 | GET /posts/feed — 25 posts exist, limit=10 | `200`, 10 posts, `has_more = true`, `next_cursor` set |
| 2 | GET /posts/feed with cursor | Posts older than cursor returned, correct ordering |
| 3 | GET /posts/feed — category=nail_art | Only posts by staff with `job_category = "nail_art"` returned |
| 4 | GET /posts/feed — 5 posts exist, limit=10 | `200`, 5 posts, `has_more = false`, `next_cursor = null` |
| 5 | GET /posts/feed — `is_liked` field | `true` for posts the requesting user has liked |
| 6 | GET /posts/feed — `staff_category` field | Populated for staff authors, `null` for regular users |
| 7 | GET /posts/feed without JWT | `401 unauthorized` |
| 8 | GET /posts/feed — limit=51 | `422 validation_error` |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Home tab opened | `CircularProgressIndicator` then first post fills screen |
| 2 | User swipes up | Next post snaps to full screen; previous post's video pauses |
| 3 | Video post enters view | Video auto-plays muted |
| 4 | User taps video | Mute icon shown briefly; audio toggles on/off |
| 5 | User swipes near end (2 from last) | `loadMore()` triggered; new posts appended without visible interruption |
| 6 | Category chip selected | Feed reloads with filtered posts; chip highlighted |
| 7 | Network error on initial load | Full-screen error state with Retry button |
| 8 | Network error on next-page load | Red SnackBar: `"Unable to load more posts."` Current post unaffected |
| 9 | Empty feed | Full-screen empty state: `"No posts yet. Check back soon!"` |
| 10 | Author name tapped | Navigate to `StaffProfileScreen` for that author |
| 11 | No mock data | All posts from API; no hardcoded content |

---

## 9. Security Checklist

- [ ] The feed endpoint requires authentication; anonymous browsing is not permitted for MVP.
- [ ] `is_liked` is computed per the requesting user's JWT identity; users cannot query liked status for other users.
- [ ] Author sensitive fields (email, password_hash) are never included in `PostResponse`.

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — Screen | `lib/screens/feed/feed_screen.dart` | Create — vertical PageView feed |
| Flutter — Provider | `lib/providers/feed_provider.dart` | Create — FeedProvider ChangeNotifier |
| Flutter — Widget | `lib/widgets/feed_post_item.dart` | Create — single post item with overlays and action column |
| Flutter — Widget | `lib/widgets/video_feed_player.dart` | Create — video player with lifecycle management |
| Flutter — Screen | `lib/screens/home/home_screen.dart` | Modify — replace mock feed with FeedScreen |
| Flutter — App | `lib/main.dart` | Modify — register FeedProvider in MultiProvider |

---

## 11. Acceptance Criteria

- [ ] The home feed renders as a full-screen vertical `PageView` with one post per page.
- [ ] Swiping up advances to the next post; swiping down returns to the previous post; scrolling snaps to page boundaries.
- [ ] Video posts auto-play (muted) when they enter the viewport and pause when they leave.
- [ ] Only one video plays at a time; all other videos are paused.
- [ ] Image posts fill the screen with `BoxFit.cover`.
- [ ] The feed loads posts from `GET /api/v1/posts/feed`; no hardcoded or mock posts are visible.
- [ ] When 2 posts from the end are reached, the next page is fetched automatically.
- [ ] Category filter chips correctly reload the feed with category-filtered posts.
- [ ] Tapping the author name or avatar navigates to `StaffProfileScreen` for that user.
- [ ] An initial load network error shows a full-screen error state with a Retry button.
- [ ] `has_more = false` stops further pagination; no unnecessary API calls are made.
- [ ] All API calls use the standard `ApiClient`; no direct HTTP calls from the screen.
