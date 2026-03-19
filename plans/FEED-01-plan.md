# [FEED-01] Create Post (Image / Video, Caption) — Implementation Plan

**Spec:** `docs/screens/FEED-01-create-post.md`
**Priority:** P1 MVP
**Phụ thuộc:**
- DB-01: `posts` table đã defined
- DB-05: `UploadService`, `POST /api/v1/media/upload-url`, folder `post` (100MB, jpg/png/webp/mp4)
- AUTH-01: `ApiClient`, JWT auth

---

## Backend Steps

### Step 1: Verify Post GORM model

**File:** `internal/model/post.go` (đã tạo ở DB-01 — verify và bổ sung nếu cần)

Đảm bảo struct `Post` có:
```go
type Post struct {
  ID            string    `gorm:"primaryKey;type:varchar(26)"`
  AuthorID      string    `gorm:"type:varchar(26);not null;index"`
  Author        User      `gorm:"foreignKey:AuthorID;constraint:OnDelete:CASCADE"`
  Content       *string   `gorm:"type:text"`
  MediaURL      *string   `gorm:"type:text"`
  MediaType     *string   `gorm:"type:varchar(10)"` // "image" hoặc "video"
  LikesCount    int       `gorm:"not null;default:0"`
  CommentsCount int       `gorm:"not null;default:0"`
  IsActive      bool      `gorm:"not null;default:true"`
  CreatedAt     time.Time
  UpdatedAt     time.Time
}
```

### Step 2: Thêm methods vào PostRepository

**File:** `internal/repository/post_repository.go` (scaffold đã có từ DB-01)

Method `Create(ctx, post *model.Post) error`:
```go
return r.db.WithContext(ctx).Create(post).Error
```

Method `GetByID(ctx, postID string) (*model.Post, error)`:
```go
var post model.Post
err := r.db.WithContext(ctx).
  Preload("Author").
  Where("id = ? AND is_active = true", postID).
  First(&post).Error
// map ErrRecordNotFound → model.ErrNotFound
```

Method `GetFeed(ctx, userID, cursor string, limit int, category string) ([]*model.Post, string, bool, error)`:
```go
query := r.db.WithContext(ctx).
  Preload("Author").
  Where("posts.is_active = true")

// Category filter: JOIN với staff_profiles nếu category được cung cấp
if category != "" {
  query = query.
    Joins("JOIN users ON users.id = posts.author_id").
    Joins("LEFT JOIN staff_profiles ON staff_profiles.user_id = posts.author_id").
    Where("staff_profiles.job_category = ?", category)
}

// Cursor pagination: lấy posts có ID < cursor (ULID ordering)
if cursor != "" {
  query = query.Where("posts.id < ?", cursor)
}

var posts []*model.Post
err := query.Order("posts.id DESC").Limit(limit + 1).Find(&posts).Error

// Check hasMore
hasMore := len(posts) > limit
if hasMore {
  posts = posts[:limit]
}

// nextCursor = last post's ID
var nextCursor string
if len(posts) > 0 {
  nextCursor = posts[len(posts)-1].ID
}

return posts, nextCursor, hasMore, err
```

### Step 3: Tạo PostService

**File:** `internal/service/post_service.go`

Method `CreatePost(ctx, userID string, req dto.CreatePostRequest) (*model.Post, error)`:
1. Build `&model.Post{ID: ulid.New(), AuthorID: userID, Content: req.Content, MediaURL: req.MediaURL, MediaType: req.MediaType}`
2. Gọi `postRepo.Create(ctx, post)`
3. Gọi `postRepo.GetByID(ctx, post.ID)` để preload Author
4. Return

Method `GetFeed(ctx, userID, cursor string, limit int, category string) ([]dto.PostResponse, string, bool, error)`:
1. Gọi `postRepo.GetFeed(ctx, userID, cursor, limit, category)`
2. Convert posts → `[]PostResponse` (bao gồm `is_liked` check)
3. Return

Method `GetByID(ctx, postID string) (*dto.PostResponse, error)`:
1. Gọi `postRepo.GetByID(ctx, postID)`. Return `ErrNotFound` nếu không có.
2. Convert → `PostResponse`

**is_liked check:** Query `likes` table `WHERE user_id = ? AND post_id = ?`. Trong MVP, có thể check batch cho tất cả posts trong feed cùng 1 query.

### Step 4: Tạo DTOs

**File:** `internal/dto/post_dto.go`

```go
type CreatePostRequest struct {
  Content   *string `json:"content"`
  MediaURL  *string `json:"media_url"`
  MediaType *string `json:"media_type"`
}

type AuthorInfo struct {
  ID        string  `json:"id"`
  Name      string  `json:"name"`
  AvatarURL *string `json:"avatar_url"`
}

type PostResponse struct {
  ID            string     `json:"id"`
  AuthorID      string     `json:"author_id"`
  Author        AuthorInfo `json:"author"`
  Content       *string    `json:"content"`
  MediaURL      *string    `json:"media_url"`
  MediaType     *string    `json:"media_type"`
  LikesCount    int        `json:"likes_count"`
  CommentsCount int        `json:"comments_count"`
  IsLiked       bool       `json:"is_liked"`
  StaffCategory *string    `json:"staff_category"` // từ staff_profiles.job_category nếu author là staff
  CreatedAt     time.Time  `json:"created_at"`
}

type FeedResponse struct {
  Posts      []PostResponse `json:"posts"`
  NextCursor *string        `json:"next_cursor"`
  HasMore    bool           `json:"has_more"`
}
```

### Step 5: Tạo PostHandler

**File:** `internal/handler/post_handler.go`

Method `CreatePost(c fiber.Ctx) error`:
1. Read `userID` từ JWT
2. Parse body vào `CreatePostRequest`
3. Validate: nếu cả `Content` và `MediaURL` đều nil/empty → `400 bad_request` "At least a caption or media is required."
4. Nếu `MediaURL != nil` và `MediaType == nil` → `400 bad_request`
5. Nếu `MediaType != nil` && không phải "image"/"video" → `422 validation_error`
6. Nếu `Content != nil && len(*Content) > 500` → `422 validation_error`
7. Nếu `MediaURL != nil` && không bắt đầu bằng `STORAGE_PUBLIC_URL` → `422 validation_error`
8. Gọi `postService.CreatePost(ctx, userID, req)`. Return `500` nếu lỗi.
9. Return `201` với `PostResponse`.

Method `GetFeed(c fiber.Ctx) error`:
1. Read `userID` từ JWT
2. Parse query params: `cursor` (string), `limit` (int, default 20, max 50), `category` (string)
3. Nếu `limit > 50` → `422 validation_error`
4. Gọi `postService.GetFeed(ctx, userID, cursor, limit, category)`
5. Return `200` với `FeedResponse`

Method `GetPostByID(c fiber.Ctx) error`:
1. Read `postID` từ `c.Params("postID")`
2. Gọi `postService.GetByID(ctx, postID)`. `ErrNotFound` → `404`.
3. Return `200` với `PostResponse`

### Step 6: Đăng ký routes

**File:** `router/router.go`

```go
posts := protected.Group("/posts")
posts.Post("", postHandler.CreatePost)
posts.Get("/feed", postHandler.GetFeed)
posts.Get("/:postID", postHandler.GetPostByID)
```

### Step 7: Update main.go

Wire `postRepo`, `postService`, `postHandler`. Pass vào `router.Setup()`.

---

## Flutter Steps

### Step 1: Thêm packages

**File:** `pubspec.yaml`

```yaml
video_player: ^2.9.2
image_picker: ^1.1.2
```

### Step 2: Tạo Post model

**File:** `lib/models/post.dart`

```dart
class AuthorInfo {
  final String id;
  final String name;
  final String? avatarUrl;
  factory AuthorInfo.fromJson(Map<String, dynamic> json) { ... }
}

class Post {
  final String id;
  final String authorId;
  final AuthorInfo author;
  final String? content;
  final String? mediaUrl;
  final String? mediaType; // "image" | "video" | null
  final int likesCount;
  final int commentsCount;
  final bool isLiked;
  final String? staffCategory;
  final DateTime createdAt;

  factory Post.fromJson(Map<String, dynamic> json) { ... }
}

class FeedResponse {
  final List<Post> posts;
  final String? nextCursor;
  final bool hasMore;
  factory FeedResponse.fromJson(Map<String, dynamic> json) { ... }
}
```

### Step 3: Tạo PostService

**File:** `lib/services/post_service.dart`

```dart
class PostService {
  static final PostService instance = PostService._();
  PostService._();

  Future<Post> createPost({
    String? content,
    String? mediaUrl,
    String? mediaType,
  }) async {
    final response = await ApiClient.instance.post('/api/v1/posts', {
      if (content != null) 'content': content,
      if (mediaUrl != null) 'media_url': mediaUrl,
      if (mediaType != null) 'media_type': mediaType,
    });
    // parse + return Post
  }

  Future<FeedResponse> getFeed({
    String? cursor,
    int limit = 20,
    String? category,
  }) async {
    final queryParams = {
      if (cursor != null) 'cursor': cursor,
      'limit': limit.toString(),
      if (category != null) 'category': category,
    };
    final response = await ApiClient.instance.get(
      '/api/v1/posts/feed',
      queryParams: queryParams,
    );
    return FeedResponse.fromJson(response.data['data']);
  }

  Future<Post?> getPost(String postID) async { ... }
}
```

### Step 4: Tạo CreatePostScreen

**File:** `lib/screens/feed/create_post_screen.dart` (tạo mới)

**State:**
```dart
XFile? _selectedFile;
String? _mimeType;
String? _videoThumbnail; // path đến generated thumbnail
TextEditingController _captionController = TextEditingController();
bool _isUploading = false;
double _uploadProgress = 0.0;
```

**Layout:**
```
Scaffold
└── AppBar:
    ├── Leading: IconButton(Icons.close) → Navigator.pop()
    └── Actions: TextButton("Share", loading: _isUploading)
└── body: Column
    ├── Media picker area (tappable GestureDetector):
    │   ├── Nếu _selectedFile == null: center icon + "Tap to add photo or video"
    │   ├── Nếu image: Image.file
    │   └── Nếu video: thumbnail preview + play icon overlay
    ├── TextFormField (caption, multiline, max 500 chars, counter)
    └── Nếu _isUploading: LinearProgressIndicator
```

**Tap media area:** show BottomSheet với 2 options:
- "Photo Library" → `ImagePicker().pickImage()`
- "Video" → `ImagePicker().pickVideo()`

**Share handler `_handleShare()`:**
1. Validate: nếu `_selectedFile == null && _captionController.text.isEmpty` → show inline message "Add a photo, video, or caption to share."
2. Show loading (Share button → CircularProgressIndicator 20×20, disabled)
3. Nếu có file:
   ```dart
   try {
     final result = await UploadService.instance.uploadFile(
       _selectedFile!, UploadFolder.post
     );
     mediaUrl = result.publicUrl;
   } on UploadValidationException catch (e) {
     showRedSnackBar(e.message); return;
   } on UploadNetworkException {
     showRedSnackBar("Upload failed. Check your connection and try again."); return;
   }
   ```
4. Gọi `PostService.instance.createPost(content: caption, mediaUrl: mediaUrl, mediaType: mimeType?.startsWith('video') == true ? 'video' : 'image')`
5. Nếu 201: green SnackBar "Post shared successfully." → `Navigator.pushNamedAndRemoveUntil('/home', (_) => false)`
6. Nếu lỗi: red SnackBar tương ứng
7. Finally: clear loading state

### Step 5: Thêm route vào main.dart

**File:** `lib/main.dart`

```dart
'/create-post': (context) => const CreatePostScreen(),
```

Thêm Create Post button (+) trong navigation bar (nếu chưa có) navigates to `/create-post`.

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/model/post.go` | VERIFY/MODIFY — Post struct đủ fields |
| `internal/repository/post_repository.go` | MODIFY — thêm Create, GetByID, GetFeed |
| `internal/service/post_service.go` | CREATE |
| `internal/dto/post_dto.go` | CREATE |
| `internal/handler/post_handler.go` | CREATE |
| `router/router.go` | MODIFY — đăng ký /api/v1/posts routes |
| `main.go` | MODIFY — wire PostHandler |

### Flutter
| File | Action |
|---|---|
| `pubspec.yaml` | MODIFY — thêm video_player, image_picker |
| `lib/models/post.dart` | CREATE |
| `lib/services/post_service.dart` | CREATE |
| `lib/screens/feed/create_post_screen.dart` | CREATE |
| `lib/main.dart` | MODIFY — thêm /create-post route |

---

## Verification Checklist

- [ ] `POST /api/v1/posts` với image media_url + caption → 201, `media_type = "image"`
- [ ] `POST /api/v1/posts` với caption only → 201, `media_url = null`
- [ ] `POST /api/v1/posts` không có content và media_url → 400
- [ ] `POST /api/v1/posts` với media_url nhưng không có media_type → 400
- [ ] `POST /api/v1/posts` với `media_type = "gif"` → 422
- [ ] `POST /api/v1/posts` với caption > 500 chars → 422
- [ ] `GET /api/v1/posts/feed` → 200, posts array + next_cursor + has_more
- [ ] `GET /api/v1/posts/feed?category=nail_art` → chỉ posts của nail art staff
- [ ] `GET /api/v1/posts/feed?limit=51` → 422
- [ ] `GET /api/v1/posts/:id` → 200
- [ ] `GET /api/v1/posts/:id` với `is_active=false` → 404
- [ ] Flutter: Tap Share không có media/caption → inline message "Add a photo, video, or caption to share."
- [ ] Flutter: File > 100MB → SnackBar "File is too large. Maximum size is 100 MB."
- [ ] Flutter: Upload thành công → green SnackBar "Post shared successfully." + navigate to feed
- [ ] Flutter: Share button disabled + loading khi đang upload
- [ ] Flutter: Caption counter đổi màu đỏ khi > 500 chars
