# [DB-01] Database Schema Design — Implementation Plan

**Spec:** `docs/screens/DB-01-database-schema.md`
**Priority:** P1 MVP
**Phụ thuộc:** AUTH-01 (users + refresh_tokens tables đã có)

---

## Trạng thái hiện tại

Đã có:
- `users` table (migration 20260310000001)
- `refresh_tokens` table (migration 20260310000002)
- GORM models: `model.User`, `model.RefreshToken`

Cần thêm tất cả tables còn lại cho Phase 1.

---

## Backend Steps

### Step 1: Tạo GORM models file cho từng domain

Mỗi domain có 1 file model riêng trong `internal/model/`.

**File:** `internal/model/staff.go`

Structs cần tạo:
- `StaffProfile` — fields: ID (ULID), UserID, User (FK), StaffNumber (VARCHAR(6) unique), JobTitle, JobCategory, Location, Bio, IntroVideoURL (*string), IsAvailable (bool, default false), AcceptBookings (bool, default true), Rating (float32, default 0), ReviewCount (int), FollowersCount (int), TotalTipsReceived (int), CreatedAt, UpdatedAt
- `StaffPortfolioPhoto` — fields: ID, StaffProfileID, StaffProfile (FK), PhotoURL, DisplayOrder (int), CreatedAt
- `StaffSocialLink` — fields: ID, StaffProfileID, Platform (varchar 50), URL (text), CreatedAt

**File:** `internal/model/post.go`

Structs:
- `Post` — ID, AuthorID, Author (FK→User), Content (*string), MediaURL (*string), MediaType (*string, "image"|"video"), LikesCount, CommentsCount, IsActive (bool default true), CreatedAt, UpdatedAt
- `Comment` — ID, PostID, Post (FK), AuthorID, Author (FK→User), Content (text, not null), IsActive, CreatedAt, UpdatedAt
- `Like` — ID, UserID, PostID, CreatedAt (unique index: user_id + post_id)
- `Follow` — ID, FollowerID, FollowedID, CreatedAt (unique index: follower_id + followed_id)
- `Story` — ID, AuthorID, MediaURL, MediaType, ViewsCount, ExpiresAt (TIMESTAMPTZ), CreatedAt

**File:** `internal/model/booking.go`

Structs:
- `Service` — ID, StaffProfileID (FK), Name, Description (*string), Price (DECIMAL(10,2)), DurationMinutes (int), IsActive, CreatedAt, UpdatedAt
- `Booking` — ID, UserID (FK→User), StaffProfileID (FK→StaffProfile), ServiceID (*string FK→Service), Status (varchar 20: pending|confirmed|completed|cancelled), Note (*string), ScheduledAt (TIMESTAMPTZ), CreatedAt, UpdatedAt

**File:** `internal/model/payment.go`

Structs:
- `Tip` — ID, SenderID (FK→User), RecipientID (FK→User), StaffProfileID (FK), Amount (int, in coins), Message (*string), CreatedAt
- `PointTransaction` — ID, UserID (FK→User), Type (varchar 50: purchase|tip_sent|tip_received|withdrawal), Amount (int), BalanceAfter (int), ReferenceID (*string), CreatedAt

**File:** `internal/model/live_stream.go`

Structs:
- `LiveStream` — ID, StaffProfileID (FK), AgoraChannelName, Status (varchar 20: live|ended), ViewerCount (int), TipTotal (int), StartedAt, EndedAt (*TIMESTAMPTZ), CreatedAt, UpdatedAt

**File:** `internal/model/review.go`

Structs:
- `Review` — ID, BookingID (FK unique), ReviewerID (FK→User), RevieweeID (FK→User), Rating (smallint 1-5), Comment (*text), CreatedAt

**File:** `internal/model/notification.go`

Structs:
- `Notification` — ID, UserID (FK→User), Type (varchar 50), Title, Body, Data (JSONB *string), IsRead (bool default false), CreatedAt

**File:** `internal/model/headhunt.go`

Structs:
- `HeadhuntOffer` — ID, CompanyID (FK→User), StaffProfileID (FK), Title, Message (text), SalaryMin (*DECIMAL), SalaryMax (*DECIMAL), Status (varchar 20: pending|accepted|declined|expired), ExpiresAt (*TIMESTAMPTZ), CreatedAt, UpdatedAt

### Step 2: Tạo SQL migrations cho Phase 1 tables

Tạo migration files theo thứ tự (đặt tên theo convention YYYYMMDDHHMMSS):

**File:** `migrations/20260310000005_create_staff_profiles.up.sql`
```sql
CREATE TABLE IF NOT EXISTS staff_profiles (
  id VARCHAR(26) PRIMARY KEY,
  user_id VARCHAR(26) NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  staff_number VARCHAR(6) NOT NULL UNIQUE,
  job_title VARCHAR(100) NOT NULL,
  job_category VARCHAR(50) NOT NULL,
  location VARCHAR(255),
  bio TEXT,
  intro_video_url TEXT,
  is_available BOOLEAN NOT NULL DEFAULT FALSE,
  accept_bookings BOOLEAN NOT NULL DEFAULT TRUE,
  rating DECIMAL(3,2) NOT NULL DEFAULT 0.00,
  review_count INTEGER NOT NULL DEFAULT 0,
  followers_count INTEGER NOT NULL DEFAULT 0,
  total_tips_received INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_staff_profiles_user_id ON staff_profiles(user_id);
CREATE INDEX IF NOT EXISTS idx_staff_profiles_job_category ON staff_profiles(job_category);
CREATE INDEX IF NOT EXISTS idx_staff_profiles_is_available ON staff_profiles(is_available);
```

**File:** `migrations/20260310000006_create_staff_portfolio_photos.up.sql`
```sql
CREATE TABLE IF NOT EXISTS staff_portfolio_photos (
  id VARCHAR(26) PRIMARY KEY,
  staff_profile_id VARCHAR(26) NOT NULL REFERENCES staff_profiles(id) ON DELETE CASCADE,
  photo_url TEXT NOT NULL,
  display_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_portfolio_staff_profile_id ON staff_portfolio_photos(staff_profile_id);
```

**File:** `migrations/20260310000007_create_posts.up.sql`
```sql
CREATE TABLE IF NOT EXISTS posts (
  id VARCHAR(26) PRIMARY KEY,
  author_id VARCHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content TEXT,
  media_url TEXT,
  media_type VARCHAR(10), -- 'image' or 'video'
  likes_count INTEGER NOT NULL DEFAULT 0,
  comments_count INTEGER NOT NULL DEFAULT 0,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_posts_author_id ON posts(author_id);
CREATE INDEX IF NOT EXISTS idx_posts_created_at ON posts(created_at DESC);
CREATE INDEX IF NOT EXISTS idx_posts_is_active ON posts(is_active);
```

**File:** `migrations/20260310000008_create_likes_comments_follows.up.sql`
```sql
CREATE TABLE IF NOT EXISTS likes (
  id VARCHAR(26) PRIMARY KEY,
  user_id VARCHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id VARCHAR(26) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(user_id, post_id)
);

CREATE TABLE IF NOT EXISTS comments (
  id VARCHAR(26) PRIMARY KEY,
  post_id VARCHAR(26) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  author_id VARCHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_comments_post_id ON comments(post_id);

CREATE TABLE IF NOT EXISTS follows (
  id VARCHAR(26) PRIMARY KEY,
  follower_id VARCHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  followed_id VARCHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(follower_id, followed_id)
);
CREATE INDEX IF NOT EXISTS idx_follows_follower_id ON follows(follower_id);
CREATE INDEX IF NOT EXISTS idx_follows_followed_id ON follows(followed_id);
```

**File:** `migrations/20260310000009_create_bookings.up.sql`
```sql
CREATE TABLE IF NOT EXISTS services (
  id VARCHAR(26) PRIMARY KEY,
  staff_profile_id VARCHAR(26) NOT NULL REFERENCES staff_profiles(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  duration_minutes INTEGER NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS bookings (
  id VARCHAR(26) PRIMARY KEY,
  user_id VARCHAR(26) NOT NULL REFERENCES users(id),
  staff_profile_id VARCHAR(26) NOT NULL REFERENCES staff_profiles(id),
  service_id VARCHAR(26) REFERENCES services(id),
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  note TEXT,
  scheduled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_bookings_user_id ON bookings(user_id);
CREATE INDEX IF NOT EXISTS idx_bookings_staff_profile_id ON bookings(staff_profile_id);
CREATE INDEX IF NOT EXISTS idx_bookings_status ON bookings(status);
```

**File:** `migrations/20260310000010_create_tips_points.up.sql`
```sql
CREATE TABLE IF NOT EXISTS tips (
  id VARCHAR(26) PRIMARY KEY,
  sender_id VARCHAR(26) NOT NULL REFERENCES users(id),
  recipient_id VARCHAR(26) NOT NULL REFERENCES users(id),
  staff_profile_id VARCHAR(26) NOT NULL REFERENCES staff_profiles(id),
  amount INTEGER NOT NULL,
  message TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_tips_sender_id ON tips(sender_id);
CREATE INDEX IF NOT EXISTS idx_tips_recipient_id ON tips(recipient_id);

CREATE TABLE IF NOT EXISTS point_transactions (
  id VARCHAR(26) PRIMARY KEY,
  user_id VARCHAR(26) NOT NULL REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  amount INTEGER NOT NULL,
  balance_after INTEGER NOT NULL,
  reference_id VARCHAR(26),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_point_transactions_user_id ON point_transactions(user_id);
```

**File:** `migrations/20260310000011_create_reviews.up.sql`
```sql
CREATE TABLE IF NOT EXISTS reviews (
  id VARCHAR(26) PRIMARY KEY,
  booking_id VARCHAR(26) NOT NULL UNIQUE REFERENCES bookings(id),
  reviewer_id VARCHAR(26) NOT NULL REFERENCES users(id),
  reviewee_id VARCHAR(26) NOT NULL REFERENCES users(id),
  rating SMALLINT NOT NULL CHECK (rating >= 1 AND rating <= 5),
  comment TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_reviews_reviewee_id ON reviews(reviewee_id);
```

**File:** `migrations/20260310000012_create_notifications.up.sql`
```sql
CREATE TABLE IF NOT EXISTS notifications (
  id VARCHAR(26) PRIMARY KEY,
  user_id VARCHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type VARCHAR(50) NOT NULL,
  title VARCHAR(255) NOT NULL,
  body TEXT NOT NULL,
  data JSONB,
  is_read BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_notifications_user_id ON notifications(user_id);
CREATE INDEX IF NOT EXISTS idx_notifications_is_read ON notifications(user_id, is_read);
```

Tạo down migrations tương ứng cho từng file (DROP TABLE IF EXISTS theo thứ tự ngược).

### Step 3: Register tất cả models vào AutoMigrate

**File:** `main.go`

Cập nhật `db.AutoMigrate()` call để include tất cả models mới:
```go
db.AutoMigrate(
  &model.User{},
  &model.RefreshToken{},
  &model.PasswordResetToken{},
  &model.StaffProfile{},
  &model.StaffPortfolioPhoto{},
  &model.StaffSocialLink{},
  &model.Post{},
  &model.Like{},
  &model.Comment{},
  &model.Follow{},
  &model.Service{},
  &model.Booking{},
  &model.Tip{},
  &model.PointTransaction{},
  &model.Review{},
  &model.Notification{},
)
```

### Step 4: Tạo repository scaffolding

Tạo empty repository files cho mỗi domain (sẽ được populate bởi feature tickets):

- `internal/repository/staff_repository.go` — struct `StaffRepository` với db field
- `internal/repository/post_repository.go`
- `internal/repository/follow_repository.go`
- `internal/repository/booking_repository.go`
- `internal/repository/review_repository.go`
- `internal/repository/notification_repository.go`

Mỗi file có `NewXxxRepository(db *gorm.DB) *XxxRepository` constructor.

---

## Flutter Steps

DB-01 là backend-only ticket. Flutter không cần thay đổi gì ở bước này.

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/model/staff.go` | CREATE |
| `internal/model/post.go` | CREATE |
| `internal/model/booking.go` | CREATE |
| `internal/model/payment.go` | CREATE |
| `internal/model/review.go` | CREATE |
| `internal/model/notification.go` | CREATE |
| `migrations/20260310000005_create_staff_profiles.up.sql` | CREATE |
| `migrations/20260310000005_create_staff_profiles.down.sql` | CREATE |
| `migrations/20260310000006_create_staff_portfolio_photos.up.sql` | CREATE |
| `migrations/20260310000006_create_staff_portfolio_photos.down.sql` | CREATE |
| `migrations/20260310000007_create_posts.up.sql` | CREATE |
| `migrations/20260310000007_create_posts.down.sql` | CREATE |
| `migrations/20260310000008_create_likes_comments_follows.up.sql` | CREATE |
| `migrations/20260310000008_create_likes_comments_follows.down.sql` | CREATE |
| `migrations/20260310000009_create_bookings.up.sql` | CREATE |
| `migrations/20260310000009_create_bookings.down.sql` | CREATE |
| `migrations/20260310000010_create_tips_points.up.sql` | CREATE |
| `migrations/20260310000010_create_tips_points.down.sql` | CREATE |
| `migrations/20260310000011_create_reviews.up.sql` | CREATE |
| `migrations/20260310000011_create_reviews.down.sql` | CREATE |
| `migrations/20260310000012_create_notifications.up.sql` | CREATE |
| `migrations/20260310000012_create_notifications.down.sql` | CREATE |
| `internal/repository/staff_repository.go` | CREATE (scaffold) |
| `internal/repository/post_repository.go` | CREATE (scaffold) |
| `internal/repository/booking_repository.go` | CREATE (scaffold) |
| `internal/repository/review_repository.go` | CREATE (scaffold) |
| `internal/repository/notification_repository.go` | CREATE (scaffold) |
| `main.go` | MODIFY — thêm models vào AutoMigrate |

---

## Verification Checklist

- [ ] App khởi động không bị lỗi AutoMigrate
- [ ] `staff_profiles` table tồn tại trong DB với đúng columns
- [ ] `staff_portfolio_photos` table có FK constraint với staff_profiles
- [ ] `posts` table tồn tại với columns đúng
- [ ] `likes` table có UNIQUE constraint (user_id, post_id)
- [ ] `follows` table có UNIQUE constraint (follower_id, followed_id)
- [ ] `bookings` table có đúng status values
- [ ] `reviews` table có `rating` CHECK constraint (1-5)
- [ ] `notifications` table có JSONB `data` column
- [ ] Tất cả ULID primary keys là VARCHAR(26)
- [ ] Tất cả FK constraints đều có index
- [ ] Down migrations có thể rollback thành công
