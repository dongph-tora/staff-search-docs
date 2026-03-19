# [FEED-02] TikTok-Style Vertical Swipe Feed — Implementation Plan

**Spec:** `docs/screens/FEED-02-tiktok-feed.md`
**Priority:** P1 MVP
**Phụ thuộc:**
- FEED-01: `PostService.getFeed()`, `Post` model, `GET /api/v1/posts/feed` endpoint
- AUTH-01: `AuthProvider`
- STAFF-03: `JobCategoryChips` widget
- Packages: `video_player` (từ FEED-01), `cached_network_image` (từ STAFF-07)

> Backend-only: không có endpoints mới. Flutter cần implement toàn bộ feed UI.

---

## Backend Steps

Backend đã đủ từ FEED-01. Chỉ cần verify:

### Verify: GET /api/v1/posts/feed response đầy đủ cho feed

Đảm bảo mỗi `PostResponse` trong feed bao gồm:
- `author.id`, `author.name`, `author.avatar_url` ← để hiển thị overlay
- `staff_category` ← String|null (job_category của staff nếu author là staff)
- `is_liked` ← bool để render filled/empty heart icon
- `likes_count`, `comments_count`

Nếu `staff_category` chưa được populate trong FEED-01: update `PostRepository.GetFeed()` để LEFT JOIN với `staff_profiles` và lấy `job_category`.

---

## Flutter Steps

### Step 1: Tạo FeedProvider

**File:** `lib/providers/feed_provider.dart`

```dart
class FeedProvider extends ChangeNotifier {
  List<Post> posts = [];
  bool isLoading = false;
  bool isLoadingMore = false;
  bool hasMore = true;
  String? nextCursor;
  String? selectedCategory; // null = All
  String? error;

  Future<void> loadFeed() async { ... }
  Future<void> loadMore() async { ... }
  void setCategory(String? category) {
    selectedCategory = category;
    loadFeed(); // reset + reload
  }
}
```

**`loadFeed()` logic:**
1. Set `isLoading = true`, `error = null`, clear `posts`, reset `nextCursor = null`, `hasMore = true`, notifyListeners()
2. Gọi `PostService.instance.getFeed(cursor: null, limit: 10, category: selectedCategory)`
3. Nếu thành công: set `posts`, `nextCursor`, `hasMore`
4. Nếu network error: `error = "Unable to load feed. Check your network."`
5. Nếu server error: `error = "Something went wrong. Please try again later."`
6. `finally: isLoading = false, notifyListeners()`

**`loadMore()` logic:**
1. Nếu `!hasMore || isLoadingMore || isLoading` → return
2. Set `isLoadingMore = true`, notifyListeners()
3. Gọi `PostService.instance.getFeed(cursor: nextCursor, limit: 10, category: selectedCategory)`
4. Nếu thành công: append to `posts`, update `nextCursor`, `hasMore`
5. Nếu lỗi: show SnackBar "Unable to load more posts." (pass SnackBar context via callback hoặc GlobalKey)
6. `finally: isLoadingMore = false, notifyListeners()`

### Step 2: Tạo VideoFeedPlayer widget

**File:** `lib/widgets/video_feed_player.dart`

StatefulWidget nhận: `String videoUrl`, `bool isActive` (true khi page này đang visible).

**initState:**
- Tạo `VideoPlayerController.networkUrl(Uri.parse(videoUrl))`
- Gọi `controller.initialize().then((_) { controller.setLooping(true); controller.setVolume(0); setState(() {}); })`

**didUpdateWidget:**
- Nếu `widget.isActive != oldWidget.isActive`:
  - `isActive == true` → `controller.play()`
  - `isActive == false` → `controller.pause()`

**dispose:**
- `controller.dispose()`

**Mute toggle:**
- State: `bool _isMuted = true`
- Tap video area: `setState(() { _isMuted = !_isMuted; controller.setVolume(_isMuted ? 0 : 1); })`
- Show mute/unmute icon 500ms rồi ẩn (dùng Timer)

**build:**
```dart
if (!controller.value.isInitialized)
  return const SizedBox.expand(child: ColoredBox(color: Colors.black));

return GestureDetector(
  onTap: _toggleMute,
  child: Stack(
    children: [
      SizedBox.expand(
        child: FittedBox(
          fit: BoxFit.cover,
          child: SizedBox(
            width: controller.value.size.width,
            height: controller.value.size.height,
            child: VideoPlayer(controller),
          ),
        ),
      ),
      if (_showMuteIcon) Center(child: Icon(Icons.volume_off, color: Colors.white, size: 48)),
    ],
  ),
);
```

### Step 3: Tạo FeedPostItem widget

**File:** `lib/widgets/feed_post_item.dart`

StatelessWidget nhận: `Post post`, `bool isActive` (cho video player).

**Layout:** full-screen `Stack`:

```
Stack
├── Background (black)
├── Media layer:
│   ├── if post.mediaType == 'video': VideoFeedPlayer(videoUrl: post.mediaUrl!, isActive: isActive)
│   ├── if post.mediaType == 'image': SizedBox.expand → CachedNetworkImage(BoxFit.cover)
│   └── if no media: ColoredBox(black) với centered caption text
│
└── Overlay (Positioned.fill):
    ├── Left bottom: Author info
    │   ├── Row: GestureDetector (tap → navigate to StaffProfileScreen)
    │   │   ├── CircleAvatar 40px: CachedNetworkImage (post.author.avatarUrl) fallback initials
    │   │   └── Column: Text(post.author.name), Text(caption 2 lines, expandable)
    │   └── if post.staffCategory != null: Category badge chip
    │
    └── Right column: Action buttons (icon + count text)
        ├── Like: IconButton (filled heart if post.isLiked, empty heart if not)
        │   - Tap: optimistic toggle + stub API call (FEED-03)
        │   - Count: post.likesCount
        ├── Comment: IconButton(Icons.comment_outlined)
        │   - Count: post.commentsCount
        │   - Tap: stub (FEED-03)
        ├── Share: IconButton(Icons.share)
        │   - Tap: stub
        └── Tip: IconButton(coin icon)
            - Tap: stub (PAY-07)
```

**Gradient overlay** ở bottom để text readable trên media:
```dart
Positioned(
  bottom: 0, left: 0, right: 0,
  child: Container(
    height: 200,
    decoration: BoxDecoration(
      gradient: LinearGradient(
        begin: Alignment.topCenter,
        end: Alignment.bottomCenter,
        colors: [Colors.transparent, Colors.black54],
      ),
    ),
  ),
)
```

### Step 4: Tạo FeedScreen

**File:** `lib/screens/feed/feed_screen.dart`

```dart
class FeedScreen extends StatefulWidget { ... }

class _FeedScreenState extends State<FeedScreen> {
  late PageController _pageController;
  int _currentPage = 0;

  @override
  void initState() {
    super.initState();
    _pageController = PageController();
    _pageController.addListener(_onPageChanged);
    // Load feed nếu chưa có
    WidgetsBinding.instance.addPostFrameCallback((_) {
      context.read<FeedProvider>().loadFeed();
    });
  }

  void _onPageChanged() {
    final page = _pageController.page?.round() ?? 0;
    if (page != _currentPage) {
      setState(() { _currentPage = page; });
      // Pre-fetch: nếu còn 2 posts đến cuối
      final provider = context.read<FeedProvider>();
      if (page >= provider.posts.length - 2) {
        provider.loadMore();
      }
    }
  }

  @override
  void dispose() {
    _pageController.removeListener(_onPageChanged);
    _pageController.dispose();
    super.dispose();
  }
}
```

**build():**
```dart
Consumer<FeedProvider>(
  builder: (context, feedProvider, child) {
    // Loading state
    if (feedProvider.isLoading) {
      return const Center(child: CircularProgressIndicator());
    }

    // Error state
    if (feedProvider.error != null) {
      return _buildErrorState(feedProvider.error!, feedProvider.loadFeed);
    }

    // Empty state
    if (feedProvider.posts.isEmpty) {
      return const Center(child: Text("No posts yet. Check back soon!"));
    }

    return Stack(
      children: [
        // Feed PageView
        PageView.builder(
          controller: _pageController,
          scrollDirection: Axis.vertical,
          itemCount: feedProvider.posts.length + (feedProvider.isLoadingMore ? 1 : 0),
          itemBuilder: (context, index) {
            if (index >= feedProvider.posts.length) {
              return const Center(child: CircularProgressIndicator());
            }
            return FeedPostItem(
              post: feedProvider.posts[index],
              isActive: index == _currentPage,
            );
          },
        ),

        // Category chips (sticky at top)
        Positioned(
          top: MediaQuery.of(context).padding.top + 8,
          left: 0, right: 0,
          child: JobCategoryChips(
            selectedKey: feedProvider.selectedCategory,
            onSelected: feedProvider.setCategory,
          ),
        ),
      ],
    );
  }
)
```

**Error state widget:**
```dart
Widget _buildErrorState(String message, VoidCallback onRetry) {
  return Center(
    child: Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Icon(Icons.error_outline, size: 48, color: Colors.grey),
        const SizedBox(height: 16),
        Text(message, textAlign: TextAlign.center),
        const SizedBox(height: 16),
        ElevatedButton(onPressed: onRetry, child: const Text("Retry")),
      ],
    ),
  );
}
```

### Step 5: Modify HomeScreen để dùng FeedScreen

**File:** `lib/screens/home_screen.dart`

Thay nội dung của Home tab bằng `FeedScreen()` widget.
Xóa tất cả mock data imports và mock post rendering.

### Step 6: Đăng ký FeedProvider trong main.dart

**File:** `lib/main.dart`

Thêm vào `MultiProvider`:
```dart
ChangeNotifierProvider(create: (_) => FeedProvider()),
```

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/repository/post_repository.go` | VERIFY/MODIFY — GetFeed JOIN với staff_profiles để lấy staff_category |
| `internal/dto/post_dto.go` | VERIFY/MODIFY — PostResponse có staff_category field |

### Flutter
| File | Action |
|---|---|
| `lib/providers/feed_provider.dart` | CREATE |
| `lib/widgets/video_feed_player.dart` | CREATE |
| `lib/widgets/feed_post_item.dart` | CREATE |
| `lib/screens/feed/feed_screen.dart` | CREATE |
| `lib/screens/home_screen.dart` | MODIFY — thay mock feed bằng FeedScreen |
| `lib/main.dart` | MODIFY — đăng ký FeedProvider |

---

## Verification Checklist

- [ ] Home tab load → CircularProgressIndicator → first post fills screen
- [ ] Swipe up → next post snaps to full screen
- [ ] Video post auto-plays muted khi enter viewport
- [ ] Video pauses khi swipe away (leave viewport)
- [ ] Chỉ 1 video play tại 1 thời điểm
- [ ] Tap video → mute toggle icon flash, audio bật/tắt
- [ ] Image post fill screen với BoxFit.cover
- [ ] Author overlay: avatar + name + caption ở bottom-left
- [ ] Action column: like/comment/share/tip ở right side
- [ ] Khi còn 2 posts đến cuối → auto fetch next page, append không blink
- [ ] Category chip select → feed reload với filtered posts, chip highlighted
- [ ] Tap "すべて" → load unfiltered feed
- [ ] Tap author name/avatar → navigate to StaffProfileScreen
- [ ] Initial load error → full-screen error + Retry button
- [ ] `has_more = false` → không fetch thêm khi swipe đến cuối
- [ ] Không có mock/hardcoded posts nào
- [ ] Empty feed → "No posts yet. Check back soon!"
- [ ] Video controller disposed khi widget unmount (no memory leak)
