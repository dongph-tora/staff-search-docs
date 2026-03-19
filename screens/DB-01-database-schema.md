# [DB-01] Database Schema Design (All Tables)

| Field | Value |
|---|---|
| **Type** | Infrastructure |
| **Priority** | P1 MVP |
| **Estimate** | 3 days manual / 1 day vibe coding |
| **Dependencies** | AUTH-01 — `User` and `RefreshToken` models created in `internal/model/user.go` (Section 6). AUTH-02 — Google OAuth fields on User model (Section 6.1). |
| **Platforms** | Backend only (Go + PostgreSQL) |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for GORM model rules (Section 4), migration naming (Section 4.4), ULID primary keys, and clean architecture layers. |

---

## 1. Overview

This ticket defines the complete PostgreSQL database schema for the staffsearch platform. It covers every table required from Phase 1 (MVP) through Phase 3 (Scale), organized by domain. Each table is specified with column definitions, types, constraints, indexes, and foreign key relationships.

Two tables already exist from AUTH-01: `users` and `refresh_tokens`. This ticket adds all remaining tables and documents the full schema in one place so that every downstream ticket (STAFF, FEED, PAY, LIVE, BOOK, CHAT, SEARCH, REV, RANK, NOTIF, SOCIAL) has a single source of truth for database structure.

All tables follow the project conventions: ULID primary keys as VARCHAR(26) generated in Go, GORM struct tags, `created_at` and `updated_at` timestamps on every table, and `db.WithContext(ctx)` in all repository methods.

### End-to-end flow

    [Developer]
         │
         ├── Adds GORM model structs to internal/model/
         ├── Runs app → main.go calls db.AutoMigrate() with all models
         │     └── GORM creates / updates tables in dev PostgreSQL
         │
         ├── Writes explicit SQL migration files in migrations/
         │     ├── YYYYMMDDHHMMSS_<description>.up.sql
         │     └── YYYYMMDDHHMMSS_<description>.down.sql
         │
         └── Runs: migrate -path migrations -database $DATABASE_URL up
               └── Applies schema to staging / production PostgreSQL

---

## 2. User Flows

### 2.1 Happy Path — Schema Creation (Development)

1. Developer adds new model structs to the appropriate file in `internal/model/`.
2. Developer registers the new structs in `main.go` inside the `db.AutoMigrate()` call.
3. Developer starts the app. GORM auto-migrates: creates missing tables, adds missing columns.
4. Developer verifies tables exist using `psql` or a DB client.

### 2.2 Happy Path — Schema Creation (Production)

1. Developer writes `.up.sql` and `.down.sql` migration files following the naming convention in `_CONVENTIONS.md` Section 4.4.
2. Developer runs `migrate -path migrations -database $DATABASE_URL up`.
3. Migration runner applies all pending migrations in order.
4. Developer verifies the migration was applied by checking the `schema_migrations` table.

### 2.3 Error Cases

| Scenario | Behaviour |
|---|---|
| AutoMigrate fails (dev) | App exits with log: "Failed to auto-migrate: ..." Developer checks model struct tags for errors. |
| Migration fails (prod) | `migrate` CLI reports the error and marks the migration as dirty. Developer fixes the SQL, runs `migrate force <version>`, then re-runs `up`. |
| Duplicate migration timestamp | `migrate` CLI refuses to run. Developer renames the file with a unique timestamp. |
| Foreign key violation in seed data | `psql` reports constraint error. Developer fixes the seed SQL insert order. |

---

## 3. UI / Screen

Not applicable for this ticket. DB-01 is a backend infrastructure ticket with no user-facing screens.

---

## 4. Frontend — Flutter

Not applicable for this ticket. No Flutter changes are required. Flutter models and API integration are handled by downstream tickets (STAFF-01, FEED-01, PAY-01, etc.).

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

No new API endpoints are introduced in this ticket. DB-01 creates the data layer (models, migrations, seed data) that downstream tickets build endpoints on top of.

### 5.2 Model Registration in main.go

The `db.AutoMigrate()` call in `main.go` must be updated to include all new model structs. The current call registers `model.User` and `model.RefreshToken`. After this ticket, it must register every model defined in Section 6.

### 5.3 Repository Scaffolding

This ticket creates empty repository files with struct definitions and constructor functions for each new domain. The actual query methods are added by downstream tickets. Each repository receives `*gorm.DB` via constructor injection and uses `db.WithContext(ctx)` on every query.

New repository files:

| File | Struct | Purpose |
|---|---|---|
| `internal/repository/staff_repository.go` | `StaffRepository` | Staff profile queries |
| `internal/repository/post_repository.go` | `PostRepository` | Post, Comment, Like queries |
| `internal/repository/story_repository.go` | `StoryRepository` | Story queries |
| `internal/repository/booking_repository.go` | `BookingRepository` | Booking and ServiceMenu queries |
| `internal/repository/tip_repository.go` | `TipRepository` | TipHistory, Gift queries |
| `internal/repository/live_stream_repository.go` | `LiveStreamRepository` | LiveStream queries |
| `internal/repository/chat_repository.go` | `ChatRepository` | Conversation, Message queries |
| `internal/repository/review_repository.go` | `ReviewRepository` | Review queries |
| `internal/repository/follow_repository.go` | `FollowRepository` | Follow queries |
| `internal/repository/notification_repository.go` | `NotificationRepository` | Notification, DeviceToken queries |
| `internal/repository/ranking_repository.go` | `RankingRepository` | League ranking queries |
| `internal/repository/point_repository.go` | `PointRepository` | PointBalance, PointTransaction, PointPackage queries |
| `internal/repository/headhunt_repository.go` | `HeadhuntRepository` | HeadhuntOffer, Company, Store queries |
| `internal/repository/subscription_repository.go` | `SubscriptionRepository` | Subscription queries |
| `internal/repository/report_repository.go` | `ReportRepository` | Report, ContentModeration queries |

---

## 6. Database

This is the primary section of this ticket. All tables are organized by domain. Existing tables (from AUTH-01) are documented here for completeness but are not re-created.

### Conventions applied to every table

| Convention | Detail |
|---|---|
| Primary key | `id VARCHAR(26)` — ULID generated in Go via `pkg/ulid.New()` |
| Timestamps | `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()` and `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()` on every table |
| Foreign keys | Named explicitly, with `ON DELETE CASCADE` or `ON DELETE SET NULL` as documented per column |
| Indexes | Declared on columns used in WHERE, JOIN, ORDER BY, or unique constraints |
| Soft deletes | Not used globally. Tables that need logical deletion use a `status` or `is_active` column |
| Naming | Snake_case for all table and column names. Plural table names (e.g., `users`, `posts`) |

---

### 6.1 AUTH Domain — Existing Tables

These tables already exist from AUTH-01. Documented here as reference for foreign keys.

#### Table: `users`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `email` | VARCHAR(255) UNIQUE | No | — | Login email |
| `password_hash` | VARCHAR(255) | Yes | NULL | Bcrypt hash; NULL for social-only accounts |
| `name` | VARCHAR(100) | No | — | Display name |
| `phone_number` | VARCHAR(20) | Yes | NULL | Optional phone |
| `avatar_url` | TEXT | Yes | NULL | Profile photo URL |
| `bio` | TEXT | Yes | NULL | Short bio |
| `role` | VARCHAR(20) | No | 'user' | One of: user, staff, admin |
| `is_staff` | BOOLEAN | No | false | Quick check for staff status |
| `is_staff_registered` | BOOLEAN | No | false | Staff profile completed |
| `is_verified` | BOOLEAN | No | false | Email verified |
| `google_id` | VARCHAR(255) UNIQUE | Yes | NULL | Google OAuth subject ID |
| `apple_id` | VARCHAR(255) UNIQUE | Yes | NULL | Apple Sign-In subject ID |
| `auth_provider` | VARCHAR(50) | No | 'email' | Primary auth method |
| `status` | VARCHAR(20) | No | 'active' | One of: active, disabled, banned |
| `points` | INTEGER | No | 0 | Legacy points field (superseded by user_point_balances) |
| `privacy_policy_accepted` | BOOLEAN | No | false | PP consent |
| `privacy_policy_version` | VARCHAR(20) | Yes | NULL | PP version accepted |
| `last_login_at` | TIMESTAMPTZ | Yes | NULL | Last successful login |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `email` (unique), `role`, `status`, `google_id` (unique, partial), `apple_id` (unique, partial).

#### Table: `refresh_tokens`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id | No | — | Token owner |
| `token_hash` | VARCHAR(64) UNIQUE | No | — | SHA-256 hash of the token |
| `expires_at` | TIMESTAMPTZ | No | — | Token expiry |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `user_id`, `token_hash` (unique), `expires_at`.

#### Table: `password_reset_tokens`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Token owner |
| `token_hash` | VARCHAR(64) UNIQUE | No | — | SHA-256 of the plaintext token |
| `expires_at` | TIMESTAMPTZ | No | — | 1 hour after creation |
| `used_at` | TIMESTAMPTZ | Yes | NULL | Set when token is consumed |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `user_id`, `token_hash` (unique), `expires_at`.

---

### 6.2 STAFF Domain

#### Table: `staff_profiles`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE, UNIQUE | No | — | One-to-one with users |
| `staff_number` | VARCHAR(6) UNIQUE | No | — | Unique 6-digit display ID |
| `job_title` | VARCHAR(100) | No | — | e.g., "Beautician", "Nail Artist" |
| `job_category` | VARCHAR(50) | No | — | e.g., "beauty", "food_beverage", "massage", "other" |
| `location` | VARCHAR(255) | Yes | NULL | e.g., "Tokyo, Shibuya" |
| `latitude` | DECIMAL(10,7) | Yes | NULL | GPS latitude for location search |
| `longitude` | DECIMAL(10,7) | Yes | NULL | GPS longitude for location search |
| `bio` | TEXT | Yes | NULL | Extended staff bio |
| `intro_video_url` | TEXT | Yes | NULL | Max 60-second intro video |
| `is_available` | BOOLEAN | No | false | Online/offline presence |
| `accept_bookings` | BOOLEAN | No | true | Whether staff accepts bookings |
| `rating` | DECIMAL(3,2) | No | 0.00 | Average rating (1.00–5.00) |
| `review_count` | INTEGER | No | 0 | Total reviews received |
| `followers_count` | INTEGER | No | 0 | Denormalized follower count |
| `total_tips_received` | INTEGER | No | 0 | Lifetime tips in coins |
| `weekly_tips_received` | INTEGER | No | 0 | Resets every Monday 00:00 UTC |
| `monthly_tips_received` | INTEGER | No | 0 | Resets 1st of month 00:00 UTC |
| `store_id` | VARCHAR(26) FK → stores.id ON DELETE SET NULL | Yes | NULL | Affiliated store (if any) |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `user_id` (unique), `staff_number` (unique), `job_category`, `is_available`, `rating`, `total_tips_received`, `store_id`, (`latitude`, `longitude`) composite for geo queries.

#### Table: `staff_portfolio_photos`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `staff_profile_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE CASCADE | No | — | Owner staff |
| `photo_url` | TEXT | No | — | S3/R2 URL |
| `display_order` | INTEGER | No | 0 | Sort position |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `staff_profile_id`.

#### Table: `staff_social_links`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `staff_profile_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE CASCADE | No | — | Owner staff |
| `platform` | VARCHAR(50) | No | — | e.g., "instagram", "twitter", "tiktok" |
| `url` | TEXT | No | — | Profile URL |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: (`staff_profile_id`, `platform`) unique composite.

---

### 6.3 FEED Domain

#### Table: `posts`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `author_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Post author |
| `content` | TEXT | Yes | NULL | Caption text |
| `media_url` | TEXT | Yes | NULL | Image or video URL |
| `media_type` | VARCHAR(10) | Yes | NULL | "image" or "video" |
| `likes_count` | INTEGER | No | 0 | Denormalized like count |
| `comments_count` | INTEGER | No | 0 | Denormalized comment count |
| `is_active` | BOOLEAN | No | true | Soft delete flag |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `author_id`, `created_at DESC` (feed ordering), `is_active`.

#### Table: `comments`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `post_id` | VARCHAR(26) FK → posts.id ON DELETE CASCADE | No | — | Parent post |
| `author_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Comment author |
| `content` | TEXT | No | — | Comment text |
| `is_active` | BOOLEAN | No | true | Soft delete flag |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `post_id`, `author_id`, `created_at`.

#### Table: `likes`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who liked |
| `post_id` | VARCHAR(26) FK → posts.id ON DELETE CASCADE | Yes | NULL | Liked post (mutually exclusive with comment_id) |
| `comment_id` | VARCHAR(26) FK → comments.id ON DELETE CASCADE | Yes | NULL | Liked comment (mutually exclusive with post_id) |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: (`user_id`, `post_id`) unique composite (partial, where post_id IS NOT NULL), (`user_id`, `comment_id`) unique composite (partial, where comment_id IS NOT NULL).

Constraint: CHECK that exactly one of `post_id` or `comment_id` is NOT NULL.

#### Table: `saved_posts`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who saved |
| `post_id` | VARCHAR(26) FK → posts.id ON DELETE CASCADE | No | — | Saved post |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: (`user_id`, `post_id`) unique composite.

#### Table: `stories`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `author_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Story author |
| `media_url` | TEXT | No | — | Image or video URL |
| `media_type` | VARCHAR(10) | No | — | "image" or "video" |
| `view_count` | INTEGER | No | 0 | Total views |
| `expires_at` | TIMESTAMPTZ | No | — | created_at + 24 hours |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `author_id`, `expires_at`, `created_at DESC`.

#### Table: `story_views`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `story_id` | VARCHAR(26) FK → stories.id ON DELETE CASCADE | No | — | Viewed story |
| `viewer_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who viewed |
| `viewed_at` | TIMESTAMPTZ | No | NOW() | When viewed |

Indexes: (`story_id`, `viewer_id`) unique composite.

---

### 6.4 SOCIAL Domain

#### Table: `follows`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `follower_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who follows |
| `followee_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who is followed |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: (`follower_id`, `followee_id`) unique composite, `followee_id` (for follower list queries).

Constraint: CHECK that `follower_id != followee_id` (cannot follow self).

---

### 6.5 PAY Domain — Coins & Tipping

#### Table: `point_packages`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `name` | VARCHAR(100) | No | — | e.g., "70 coins" |
| `coins` | INTEGER | No | — | Number of coins in package |
| `web_price` | INTEGER | No | — | Price in JPY (web purchase) |
| `app_price` | INTEGER | No | — | Price in JPY (in-app purchase) |
| `is_popular` | BOOLEAN | No | false | Highlighted in UI |
| `display_order` | INTEGER | No | 0 | Sort position |
| `is_active` | BOOLEAN | No | true | Available for purchase |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `is_active`, `display_order`.

#### Table: `user_point_balances`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `user_id` | VARCHAR(26) PK, FK → users.id ON DELETE CASCADE | No | — | One-to-one with users |
| `total_points` | INTEGER | No | 0 | All coins earned or purchased |
| `purchased_points` | INTEGER | No | 0 | Coins bought with real money |
| `bonus_points` | INTEGER | No | 0 | Promotional / free coins |
| `used_points` | INTEGER | No | 0 | Coins spent |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last balance change |

Available balance is computed as: `total_points - used_points`.

#### Table: `point_transactions`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Balance owner |
| `type` | VARCHAR(30) | No | — | "purchase", "earn_checkin", "earn_ad", "spend_tip", "spend_booking" |
| `amount` | INTEGER | No | — | Signed: positive = gain, negative = spend |
| `balance_before` | INTEGER | No | — | Balance snapshot before this transaction |
| `balance_after` | INTEGER | No | — | Balance snapshot after this transaction |
| `related_entity_id` | VARCHAR(26) | Yes | NULL | FK to related record (e.g., tip_histories.id) |
| `related_entity_type` | VARCHAR(30) | Yes | NULL | "tip", "package", "checkin", "ad" |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `user_id`, `type`, `created_at DESC`.

#### Table: `gifts`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `name` | VARCHAR(100) | No | — | e.g., "Heart", "Rose", "Diamond" |
| `emoji` | VARCHAR(10) | Yes | NULL | Emoji shorthand |
| `cost_in_coins` | INTEGER | No | — | Price in coins |
| `description` | TEXT | Yes | NULL | Gift description |
| `icon_url` | TEXT | Yes | NULL | Gift icon image URL |
| `is_active` | BOOLEAN | No | true | Available in catalog |
| `display_order` | INTEGER | No | 0 | Sort position |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `is_active`, `display_order`.

#### Table: `tip_histories`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `sender_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Tipper |
| `receiver_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Staff receiving tip |
| `gift_id` | VARCHAR(26) FK → gifts.id ON DELETE SET NULL | Yes | NULL | NULL for custom amount tips |
| `amount_in_coins` | INTEGER | No | — | Coin value of the tip |
| `context` | VARCHAR(20) | No | — | "post", "profile", "livestream", "booking" |
| `context_id` | VARCHAR(26) | Yes | NULL | FK to related entity (post, livestream, etc.) |
| `message` | TEXT | Yes | NULL | Optional tip message |
| `staff_earnings` | INTEGER | No | — | 70% or 65% of tip (depends on store affiliation) |
| `platform_earnings` | INTEGER | No | — | 30% of tip |
| `store_share_amount` | INTEGER | No | 0 | 0–10% if staff is store-affiliated |
| `shard_count` | INTEGER | No | 0 | Shards earned by sender |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `sender_id`, `receiver_id`, `context`, `created_at DESC`.

#### Table: `shards`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Shard earner |
| `staff_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE SET NULL | Yes | NULL | Staff who received the tip |
| `shard_count` | INTEGER | No | — | Number of shards earned |
| `source` | VARCHAR(20) | No | — | "gift", "daily_bonus", "event" |
| `gift_coins_value` | INTEGER | No | 0 | Coin value that generated these shards |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `user_id`, `staff_id`, `created_at DESC`.

#### Table: `shard_redemptions`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who redeemed |
| `reward_type` | VARCHAR(30) | No | — | "bronze_badge", "silver_badge", "gold_badge", "limited_stamp_set", "platinum_frame", "diamond_icon" |
| `shards_cost` | INTEGER | No | — | Shards spent (50, 100, 500, 1000, 5000, 10000) |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `user_id`, `created_at DESC`.

---

### 6.6 LIVE Domain

#### Table: `live_streams`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID, also used as Agora channel name |
| `host_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Staff hosting the stream |
| `title` | VARCHAR(255) | Yes | NULL | Stream title |
| `description` | TEXT | Yes | NULL | Stream description |
| `status` | VARCHAR(20) | No | 'active' | "active" or "ended" |
| `stream_type` | VARCHAR(20) | No | 'solo' | "solo", "duet", "trio", "squad" |
| `current_viewers` | INTEGER | No | 0 | Concurrent viewer count (updated via Redis, persisted periodically) |
| `total_viewers` | INTEGER | No | 0 | Cumulative unique viewers |
| `total_tips_received` | INTEGER | No | 0 | Denormalized tip sum in coins |
| `started_at` | TIMESTAMPTZ | No | NOW() | Stream start |
| `ended_at` | TIMESTAMPTZ | Yes | NULL | Stream end (NULL while active) |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `host_id`, `status`, `started_at DESC`.

#### Table: `live_stream_comments`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `live_stream_id` | VARCHAR(26) FK → live_streams.id ON DELETE CASCADE | No | — | Parent stream |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Comment author |
| `content` | TEXT | No | — | Comment text |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `live_stream_id`, `created_at`.

#### Table: `live_collab_sessions`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `live_stream_id` | VARCHAR(26) FK → live_streams.id ON DELETE CASCADE | No | — | Parent live stream |
| `host_user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Collab creator |
| `mode` | VARCHAR(20) | No | 'solo' | "solo", "duet", "trio", "squad" |
| `battle_type` | VARCHAR(20) | No | 'none' | "none", "one_vs_one", "two_vs_two", "free_for_all" |
| `battle_status` | VARCHAR(20) | No | 'waiting' | "waiting", "ready", "active", "finished" |
| `current_viewers` | INTEGER | No | 0 | Concurrent viewers |
| `total_viewers` | INTEGER | No | 0 | Cumulative viewers |
| `started_at` | TIMESTAMPTZ | No | NOW() | Session start |
| `ended_at` | TIMESTAMPTZ | Yes | NULL | Session end |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `live_stream_id`, `battle_status`.

#### Table: `collab_participants`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `collab_session_id` | VARCHAR(26) FK → live_collab_sessions.id ON DELETE CASCADE | No | — | Parent session |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Participant |
| `is_host` | BOOLEAN | No | false | Whether this participant is the host |
| `is_muted` | BOOLEAN | No | false | Audio muted |
| `is_camera_off` | BOOLEAN | No | false | Camera off |
| `gift_coins_received` | INTEGER | No | 0 | Tips received during battle |
| `joined_at` | TIMESTAMPTZ | No | NOW() | When participant joined |

Indexes: (`collab_session_id`, `user_id`) unique composite.

---

### 6.7 BOOK Domain

#### Table: `service_menus`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `staff_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE CASCADE | No | — | Owner staff |
| `service_name` | VARCHAR(200) | No | — | e.g., "Hair Cut", "Manicure" |
| `description` | TEXT | Yes | NULL | Service description |
| `price` | INTEGER | No | — | Price in JPY |
| `duration_minutes` | INTEGER | No | — | Service duration |
| `image_url` | TEXT | Yes | NULL | Service image |
| `is_active` | BOOLEAN | No | true | Available for booking |
| `display_order` | INTEGER | No | 0 | Sort position |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `staff_id`, `is_active`.

#### Table: `bookings`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Customer |
| `staff_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE CASCADE | No | — | Service provider |
| `service_id` | VARCHAR(26) FK → service_menus.id ON DELETE SET NULL | Yes | NULL | Booked service |
| `booking_date` | DATE | No | — | Appointment date |
| `start_time` | TIME | No | — | Appointment start time |
| `duration_minutes` | INTEGER | No | — | Duration |
| `number_of_people` | INTEGER | No | 1 | Party size |
| `notes` | TEXT | Yes | NULL | Customer notes |
| `total_price` | INTEGER | No | — | Total in JPY |
| `status` | VARCHAR(20) | No | 'pending' | "pending", "confirmed", "completed", "cancelled" |
| `cancellation_reason` | TEXT | Yes | NULL | Set on cancellation |
| `payment_status` | VARCHAR(20) | No | 'pending' | "pending", "paid", "refunded" |
| `confirmed_at` | TIMESTAMPTZ | Yes | NULL | When staff confirmed |
| `completed_at` | TIMESTAMPTZ | Yes | NULL | When service completed |
| `cancelled_at` | TIMESTAMPTZ | Yes | NULL | When cancelled |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `user_id`, `staff_id`, `status`, `booking_date`, `created_at DESC`.

#### Table: `coupons`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `staff_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE CASCADE | No | — | Coupon creator |
| `code` | VARCHAR(20) UNIQUE | No | — | Auto-generated unique code |
| `name` | VARCHAR(100) | No | — | Coupon name |
| `description` | TEXT | Yes | NULL | Description |
| `discount_type` | VARCHAR(20) | No | — | "fixed" or "percentage" |
| `discount_value` | INTEGER | No | — | Amount (JPY for fixed, percentage for %) |
| `max_usages` | INTEGER | No | — | Total uses allowed |
| `usage_count` | INTEGER | No | 0 | Times used |
| `expires_at` | TIMESTAMPTZ | No | — | Expiry timestamp |
| `is_active` | BOOLEAN | No | true | Currently available |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `staff_id`, `code` (unique), `is_active`, `expires_at`.

---

### 6.8 REV Domain

#### Table: `reviews`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `reviewer_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who wrote the review |
| `reviewee_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who is being reviewed |
| `booking_id` | VARCHAR(26) FK → bookings.id ON DELETE SET NULL | Yes | NULL | Related booking (if post-booking review) |
| `rating` | SMALLINT | No | — | 1 to 5 stars |
| `comment` | TEXT | Yes | NULL | Review text |
| `is_anonymous` | BOOLEAN | No | false | Anonymous review |
| `is_active` | BOOLEAN | No | true | Soft delete flag |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `reviewer_id`, `reviewee_id`, `booking_id`, `rating`, `created_at DESC`.

Constraint: CHECK that `rating >= 1 AND rating <= 5`.

#### Table: `review_images`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `review_id` | VARCHAR(26) FK → reviews.id ON DELETE CASCADE | No | — | Parent review |
| `image_url` | TEXT | No | — | Photo URL |
| `display_order` | INTEGER | No | 0 | Sort position |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `review_id`.

---

### 6.9 CHAT Domain

#### Table: `conversations`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user1_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | First participant (lower ULID) |
| `user2_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Second participant (higher ULID) |
| `last_message_at` | TIMESTAMPTZ | Yes | NULL | Timestamp of most recent message |
| `last_message_content` | TEXT | Yes | NULL | Preview of last message |
| `user1_unread_count` | INTEGER | No | 0 | Unread messages for user1 |
| `user2_unread_count` | INTEGER | No | 0 | Unread messages for user2 |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: (`user1_id`, `user2_id`) unique composite, `user1_id`, `user2_id`, `last_message_at DESC`.

Convention: `user1_id` always holds the lexicographically smaller ULID to prevent duplicate conversations.

#### Table: `messages`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `conversation_id` | VARCHAR(26) FK → conversations.id ON DELETE CASCADE | No | — | Parent conversation |
| `sender_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Message author |
| `content` | TEXT | Yes | NULL | Message text |
| `media_url` | TEXT | Yes | NULL | Attached image URL |
| `media_type` | VARCHAR(10) | Yes | NULL | "image" (extensible to "video", "audio" later) |
| `is_read` | BOOLEAN | No | false | Read by recipient |
| `read_at` | TIMESTAMPTZ | Yes | NULL | When read |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `conversation_id`, `sender_id`, `created_at DESC`, (`conversation_id`, `created_at DESC`) composite for message history pagination.

---

### 6.10 NOTIF Domain

#### Table: `device_tokens`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Token owner |
| `token` | TEXT UNIQUE | No | — | FCM device token |
| `platform` | VARCHAR(10) | No | — | "ios", "android", "web" |
| `is_active` | BOOLEAN | No | true | Token still valid |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `user_id`, `token` (unique), `is_active`.

#### Table: `notifications`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Notification recipient |
| `type` | VARCHAR(30) | No | — | "follow", "like", "comment", "tip", "message", "booking", "review", "system" |
| `title` | VARCHAR(255) | No | — | Notification title |
| `body` | TEXT | No | — | Notification body text |
| `related_user_id` | VARCHAR(26) FK → users.id ON DELETE SET NULL | Yes | NULL | Who triggered the notification |
| `related_entity_id` | VARCHAR(26) | Yes | NULL | Related entity ID |
| `related_entity_type` | VARCHAR(30) | Yes | NULL | "post", "booking", "livestream", "review", etc. |
| `is_read` | BOOLEAN | No | false | Read status |
| `read_at` | TIMESTAMPTZ | Yes | NULL | When read |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `user_id`, (`user_id`, `is_read`) composite for unread count, `type`, `created_at DESC`.

---

### 6.11 RANK Domain

#### Table: `liver_league_infos`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `staff_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE CASCADE, UNIQUE | No | — | One-to-one with staff |
| `current_league` | VARCHAR(20) | No | 'bronze' | "bronze", "silver", "gold", "platinum", "diamond", "master", "grandmaster", "legend" |
| `total_coins_received` | INTEGER | No | 0 | Lifetime tip coins |
| `weekly_coins` | INTEGER | No | 0 | Resets Monday 00:00 UTC |
| `monthly_coins` | INTEGER | No | 0 | Resets 1st of month 00:00 UTC |
| `rank` | INTEGER | No | 0 | Global ranking position |
| `total_shards` | INTEGER | No | 0 | Lifetime shards |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `staff_id` (unique), `current_league`, `total_coins_received DESC`, `weekly_coins DESC`, `monthly_coins DESC`, `rank`.

#### Table: `gifter_league_infos`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE, UNIQUE | No | — | One-to-one with users |
| `current_league` | VARCHAR(20) | No | 'bronze' | Same league tiers as liver |
| `total_coins_sent` | INTEGER | No | 0 | Lifetime coins tipped |
| `weekly_coins_sent` | INTEGER | No | 0 | Resets Monday 00:00 UTC |
| `monthly_coins_sent` | INTEGER | No | 0 | Resets 1st of month 00:00 UTC |
| `rank` | INTEGER | No | 0 | Global ranking position |
| `total_shards` | INTEGER | No | 0 | Lifetime shards earned |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `user_id` (unique), `current_league`, `total_coins_sent DESC`, `weekly_coins_sent DESC`, `monthly_coins_sent DESC`, `rank`.

---

### 6.12 HEADHUNT Domain

#### Table: `companies`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `owner_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Company owner |
| `name` | VARCHAR(200) | No | — | Company name |
| `company_type` | VARCHAR(50) | No | — | "restaurant", "beauty_salon", "other" |
| `address` | TEXT | Yes | NULL | Address |
| `phone_number` | VARCHAR(20) | Yes | NULL | Contact phone |
| `email` | VARCHAR(255) | Yes | NULL | Contact email |
| `website_url` | TEXT | Yes | NULL | Website |
| `description` | TEXT | Yes | NULL | Company description |
| `is_store` | BOOLEAN | No | false | Also functions as a store |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `owner_id`, `company_type`.

#### Table: `stores`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `company_id` | VARCHAR(26) FK → companies.id ON DELETE CASCADE, UNIQUE | No | — | One-to-one with company |
| `store_name` | VARCHAR(200) | No | — | Store display name |
| `store_type` | VARCHAR(50) | No | — | "restaurant", "beauty_salon", "other" |
| `address` | TEXT | Yes | NULL | Store address |
| `phone_number` | VARCHAR(20) | Yes | NULL | Store phone |
| `business_hours` | VARCHAR(100) | Yes | NULL | e.g., "10:00-22:00" |
| `closed_days` | TEXT | Yes | NULL | JSON array of day names |
| `description` | TEXT | Yes | NULL | Store description |
| `tip_share_rate` | SMALLINT | No | 0 | 0–10 percent of affiliated staff tips |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `company_id` (unique), `store_type`.

Constraint: CHECK that `tip_share_rate >= 0 AND tip_share_rate <= 10`.

#### Table: `store_images`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `store_id` | VARCHAR(26) FK → stores.id ON DELETE CASCADE | No | — | Parent store |
| `image_url` | TEXT | No | — | Image URL |
| `display_order` | INTEGER | No | 0 | Sort position |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `store_id`.

#### Table: `headhunt_offers`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `company_id` | VARCHAR(26) FK → companies.id ON DELETE CASCADE | No | — | Sending company |
| `sender_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Recruiter user |
| `staff_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE CASCADE | No | — | Target staff |
| `job_title` | VARCHAR(200) | No | — | Offered position |
| `salary_range` | VARCHAR(100) | Yes | NULL | e.g., "2,500,000 - 3,500,000" |
| `location` | VARCHAR(255) | Yes | NULL | Work location |
| `message` | TEXT | Yes | NULL | Offer message |
| `status` | VARCHAR(20) | No | 'pending' | "pending", "accepted", "rejected", "expired" |
| `accepted_at` | TIMESTAMPTZ | Yes | NULL | When accepted |
| `rejected_at` | TIMESTAMPTZ | Yes | NULL | When rejected |
| `expires_at` | TIMESTAMPTZ | No | — | Typically 30 days from creation |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `company_id`, `sender_id`, `staff_id`, `status`, `expires_at`.

#### Table: `store_staff_offers`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `store_id` | VARCHAR(26) FK → stores.id ON DELETE CASCADE | No | — | Sending store |
| `sender_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Store owner user |
| `staff_id` | VARCHAR(26) FK → staff_profiles.id ON DELETE CASCADE | No | — | Target staff |
| `position` | VARCHAR(200) | No | — | Offered position |
| `salary_range` | VARCHAR(100) | Yes | NULL | Salary range |
| `location` | VARCHAR(255) | Yes | NULL | Work location |
| `start_date` | DATE | Yes | NULL | Proposed start date |
| `message` | TEXT | Yes | NULL | Offer message |
| `status` | VARCHAR(20) | No | 'pending' | "pending", "accepted", "rejected", "expired" |
| `accepted_at` | TIMESTAMPTZ | Yes | NULL | When accepted |
| `rejected_at` | TIMESTAMPTZ | Yes | NULL | When rejected |
| `expires_at` | TIMESTAMPTZ | No | — | Typically 30 days |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `store_id`, `sender_id`, `staff_id`, `status`, `expires_at`.

---

### 6.13 SUBSCRIPTION Domain

#### Table: `subscription_plans`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `plan_name` | VARCHAR(100) | No | — | e.g., "Headhunt Free", "Headhunt Unlimited" |
| `target_role` | VARCHAR(20) | No | — | "company", "staff", "all" |
| `price_per_month` | INTEGER | No | — | JPY per month |
| `features` | TEXT | No | — | JSON array of feature strings |
| `is_active` | BOOLEAN | No | true | Currently available |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `target_role`, `is_active`.

#### Table: `user_subscriptions`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Subscriber |
| `plan_id` | VARCHAR(26) FK → subscription_plans.id ON DELETE SET NULL | Yes | NULL | Subscribed plan |
| `status` | VARCHAR(20) | No | 'active' | "active", "cancelled", "expired" |
| `start_date` | DATE | No | — | Subscription start |
| `renewal_date` | DATE | No | — | Next renewal date |
| `auto_renew` | BOOLEAN | No | true | Auto-renew enabled |
| `cancelled_at` | TIMESTAMPTZ | Yes | NULL | When cancelled |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `user_id`, `plan_id`, `status`, `renewal_date`.

---

### 6.14 ADMIN Domain

#### Table: `reports`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `reporter_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who reported |
| `reported_user_id` | VARCHAR(26) FK → users.id ON DELETE SET NULL | Yes | NULL | Reported user |
| `reported_entity_type` | VARCHAR(20) | No | — | "post", "comment", "user", "review" |
| `reported_entity_id` | VARCHAR(26) | No | — | ID of reported entity |
| `reason` | VARCHAR(50) | No | — | "spam", "harassment", "inappropriate_content", "fraud", "other" |
| `description` | TEXT | Yes | NULL | Free-text description |
| `status` | VARCHAR(20) | No | 'open' | "open", "investigating", "resolved", "dismissed" |
| `admin_notes` | TEXT | Yes | NULL | Admin investigation notes |
| `resolution` | VARCHAR(30) | Yes | NULL | "content_removed", "user_warned", "user_banned", "no_action" |
| `resolved_at` | TIMESTAMPTZ | Yes | NULL | When resolved |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |
| `updated_at` | TIMESTAMPTZ | No | NOW() | Last update |

Indexes: `reporter_id`, `reported_user_id`, `status`, `reason`, `created_at DESC`.

#### Table: `content_moderations`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `admin_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Admin who acted |
| `action_type` | VARCHAR(30) | No | — | "delete_post", "delete_comment", "warn_user", "ban_user", "restore_content" |
| `target_type` | VARCHAR(20) | No | — | "post", "comment", "user", "review" |
| `target_id` | VARCHAR(26) | No | — | ID of target entity |
| `reason` | TEXT | No | — | Reason for action |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: `admin_id`, `target_type`, `created_at DESC`.

---

### 6.15 Daily Check-in (Coin Earning)

#### Table: `daily_checkins`

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | VARCHAR(26) PK | No | — | ULID |
| `user_id` | VARCHAR(26) FK → users.id ON DELETE CASCADE | No | — | Who checked in |
| `streak_day` | SMALLINT | No | — | Current streak day (1–7) |
| `coins_earned` | INTEGER | No | — | Coins earned this check-in (5, 10, 15, 20, 30, 40, 50) |
| `checked_in_date` | DATE | No | — | Calendar date of check-in |
| `created_at` | TIMESTAMPTZ | No | NOW() | Row creation |

Indexes: (`user_id`, `checked_in_date`) unique composite, `user_id`.

---

### 6.16 Entity-Relationship Summary

| From Table | To Table | Relationship | FK Column | On Delete |
|---|---|---|---|---|
| `staff_profiles` | `users` | Many-to-One (unique) | `user_id` | CASCADE |
| `staff_profiles` | `stores` | Many-to-One | `store_id` | SET NULL |
| `staff_portfolio_photos` | `staff_profiles` | Many-to-One | `staff_profile_id` | CASCADE |
| `staff_social_links` | `staff_profiles` | Many-to-One | `staff_profile_id` | CASCADE |
| `posts` | `users` | Many-to-One | `author_id` | CASCADE |
| `comments` | `posts` | Many-to-One | `post_id` | CASCADE |
| `comments` | `users` | Many-to-One | `author_id` | CASCADE |
| `likes` | `users` | Many-to-One | `user_id` | CASCADE |
| `likes` | `posts` | Many-to-One | `post_id` | CASCADE |
| `likes` | `comments` | Many-to-One | `comment_id` | CASCADE |
| `saved_posts` | `users` | Many-to-One | `user_id` | CASCADE |
| `saved_posts` | `posts` | Many-to-One | `post_id` | CASCADE |
| `stories` | `users` | Many-to-One | `author_id` | CASCADE |
| `story_views` | `stories` | Many-to-One | `story_id` | CASCADE |
| `story_views` | `users` | Many-to-One | `viewer_id` | CASCADE |
| `follows` | `users` | Many-to-One | `follower_id` | CASCADE |
| `follows` | `users` | Many-to-One | `followee_id` | CASCADE |
| `user_point_balances` | `users` | One-to-One | `user_id` (PK) | CASCADE |
| `point_transactions` | `users` | Many-to-One | `user_id` | CASCADE |
| `tip_histories` | `users` | Many-to-One | `sender_id` | CASCADE |
| `tip_histories` | `users` | Many-to-One | `receiver_id` | CASCADE |
| `tip_histories` | `gifts` | Many-to-One | `gift_id` | SET NULL |
| `shards` | `users` | Many-to-One | `user_id` | CASCADE |
| `shards` | `staff_profiles` | Many-to-One | `staff_id` | SET NULL |
| `shard_redemptions` | `users` | Many-to-One | `user_id` | CASCADE |
| `live_streams` | `users` | Many-to-One | `host_id` | CASCADE |
| `live_stream_comments` | `live_streams` | Many-to-One | `live_stream_id` | CASCADE |
| `live_stream_comments` | `users` | Many-to-One | `user_id` | CASCADE |
| `live_collab_sessions` | `live_streams` | Many-to-One | `live_stream_id` | CASCADE |
| `collab_participants` | `live_collab_sessions` | Many-to-One | `collab_session_id` | CASCADE |
| `collab_participants` | `users` | Many-to-One | `user_id` | CASCADE |
| `service_menus` | `staff_profiles` | Many-to-One | `staff_id` | CASCADE |
| `bookings` | `users` | Many-to-One | `user_id` | CASCADE |
| `bookings` | `staff_profiles` | Many-to-One | `staff_id` | CASCADE |
| `bookings` | `service_menus` | Many-to-One | `service_id` | SET NULL |
| `coupons` | `staff_profiles` | Many-to-One | `staff_id` | CASCADE |
| `reviews` | `users` | Many-to-One | `reviewer_id` | CASCADE |
| `reviews` | `users` | Many-to-One | `reviewee_id` | CASCADE |
| `reviews` | `bookings` | Many-to-One | `booking_id` | SET NULL |
| `review_images` | `reviews` | Many-to-One | `review_id` | CASCADE |
| `conversations` | `users` | Many-to-One | `user1_id` | CASCADE |
| `conversations` | `users` | Many-to-One | `user2_id` | CASCADE |
| `messages` | `conversations` | Many-to-One | `conversation_id` | CASCADE |
| `messages` | `users` | Many-to-One | `sender_id` | CASCADE |
| `device_tokens` | `users` | Many-to-One | `user_id` | CASCADE |
| `notifications` | `users` | Many-to-One | `user_id` | CASCADE |
| `notifications` | `users` | Many-to-One | `related_user_id` | SET NULL |
| `liver_league_infos` | `staff_profiles` | One-to-One | `staff_id` | CASCADE |
| `gifter_league_infos` | `users` | One-to-One | `user_id` | CASCADE |
| `companies` | `users` | Many-to-One | `owner_id` | CASCADE |
| `stores` | `companies` | One-to-One | `company_id` | CASCADE |
| `store_images` | `stores` | Many-to-One | `store_id` | CASCADE |
| `headhunt_offers` | `companies` | Many-to-One | `company_id` | CASCADE |
| `headhunt_offers` | `users` | Many-to-One | `sender_id` | CASCADE |
| `headhunt_offers` | `staff_profiles` | Many-to-One | `staff_id` | CASCADE |
| `store_staff_offers` | `stores` | Many-to-One | `store_id` | CASCADE |
| `store_staff_offers` | `users` | Many-to-One | `sender_id` | CASCADE |
| `store_staff_offers` | `staff_profiles` | Many-to-One | `staff_id` | CASCADE |
| `user_subscriptions` | `users` | Many-to-One | `user_id` | CASCADE |
| `user_subscriptions` | `subscription_plans` | Many-to-One | `plan_id` | SET NULL |
| `reports` | `users` | Many-to-One | `reporter_id` | CASCADE |
| `reports` | `users` | Many-to-One | `reported_user_id` | SET NULL |
| `content_moderations` | `users` | Many-to-One | `admin_id` | CASCADE |
| `daily_checkins` | `users` | Many-to-One | `user_id` | CASCADE |

---

### 6.17 Migration File Plan

Migrations must be created in dependency order. Tables with no foreign keys are created first; tables with FKs are created after their referenced tables exist.

| Order | Migration File | Tables Created |
|---|---|---|
| 1 | `20260310000001_create_users_table` | users (EXISTING) |
| 2 | `20260310000002_create_refresh_tokens_table` | refresh_tokens (EXISTING) |
| 3 | `20260311000001_create_password_reset_tokens_table` | password_reset_tokens |
| 4 | `20260311000002_create_gifts_table` | gifts |
| 5 | `20260311000003_create_point_packages_table` | point_packages |
| 6 | `20260311000004_create_subscription_plans_table` | subscription_plans |
| 7 | `20260311000005_create_companies_table` | companies |
| 8 | `20260311000006_create_stores_table` | stores |
| 9 | `20260311000007_create_store_images_table` | store_images |
| 10 | `20260311000008_create_staff_profiles_table` | staff_profiles |
| 11 | `20260311000009_create_staff_portfolio_photos_table` | staff_portfolio_photos |
| 12 | `20260311000010_create_staff_social_links_table` | staff_social_links |
| 13 | `20260311000011_create_posts_table` | posts |
| 14 | `20260311000012_create_comments_table` | comments |
| 15 | `20260311000013_create_likes_table` | likes |
| 16 | `20260311000014_create_saved_posts_table` | saved_posts |
| 17 | `20260311000015_create_stories_table` | stories |
| 18 | `20260311000016_create_story_views_table` | story_views |
| 19 | `20260311000017_create_follows_table` | follows |
| 20 | `20260311000018_create_user_point_balances_table` | user_point_balances |
| 21 | `20260311000019_create_point_transactions_table` | point_transactions |
| 22 | `20260311000020_create_tip_histories_table` | tip_histories |
| 23 | `20260311000021_create_shards_table` | shards |
| 24 | `20260311000022_create_shard_redemptions_table` | shard_redemptions |
| 25 | `20260311000023_create_live_streams_table` | live_streams |
| 26 | `20260311000024_create_live_stream_comments_table` | live_stream_comments |
| 27 | `20260311000025_create_live_collab_sessions_table` | live_collab_sessions |
| 28 | `20260311000026_create_collab_participants_table` | collab_participants |
| 29 | `20260311000027_create_service_menus_table` | service_menus |
| 30 | `20260311000028_create_bookings_table` | bookings |
| 31 | `20260311000029_create_coupons_table` | coupons |
| 32 | `20260311000030_create_reviews_table` | reviews |
| 33 | `20260311000031_create_review_images_table` | review_images |
| 34 | `20260311000032_create_conversations_table` | conversations |
| 35 | `20260311000033_create_messages_table` | messages |
| 36 | `20260311000034_create_device_tokens_table` | device_tokens |
| 37 | `20260311000035_create_notifications_table` | notifications |
| 38 | `20260311000036_create_liver_league_infos_table` | liver_league_infos |
| 39 | `20260311000037_create_gifter_league_infos_table` | gifter_league_infos |
| 40 | `20260311000038_create_headhunt_offers_table` | headhunt_offers |
| 41 | `20260311000039_create_store_staff_offers_table` | store_staff_offers |
| 42 | `20260311000040_create_user_subscriptions_table` | user_subscriptions |
| 43 | `20260311000041_create_reports_table` | reports |
| 44 | `20260311000042_create_content_moderations_table` | content_moderations |
| 45 | `20260311000043_create_daily_checkins_table` | daily_checkins |

---

## 7. Configuration

No new environment variables are required for this ticket. The existing database connection variables (`DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_SSLMODE`) and Redis variables (`REDIS_URL`) are sufficient.

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Run AutoMigrate with all model structs | All 38 tables are created without errors |
| 2 | Run AutoMigrate twice (idempotent) | No errors on second run; no duplicate tables |
| 3 | Insert a record with a valid ULID primary key | Record inserted successfully |
| 4 | Insert a record with a duplicate ULID | Unique constraint violation error |
| 5 | Insert a record referencing a non-existent foreign key | Foreign key constraint violation error |
| 6 | Delete a user who has related records (CASCADE) | User and all CASCADE-linked records are deleted |
| 7 | Delete a user who is referenced with SET NULL | FK column set to NULL, related record remains |
| 8 | Insert a review with rating = 0 | CHECK constraint violation |
| 9 | Insert a review with rating = 5 | Record inserted successfully |
| 10 | Insert a follow where follower_id = followee_id | CHECK constraint violation |
| 11 | Insert duplicate like on same post by same user | Unique constraint violation |
| 12 | Run all up migrations in order | All migrations apply successfully |
| 13 | Run all down migrations in reverse | All tables dropped cleanly |
| 14 | Insert seed data after migrations | Seed data inserted without errors |

### 8.2 Flutter Test Cases

Not applicable for this ticket.

---

## 9. Security Checklist

- [ ] All user-facing string columns have reasonable length limits (VARCHAR with max length) to prevent storage abuse
- [ ] Foreign key constraints enforce referential integrity — orphaned records are not possible
- [ ] The `password_hash` column remains `json:"-"` in the GORM model to prevent accidental serialization
- [ ] Sensitive columns (`google_id`, `apple_id`, `token_hash`) are excluded from API responses via `json:"-"` tags
- [ ] CHECK constraints prevent invalid data (e.g., rating out of range, self-follow, tip_share_rate out of bounds)
- [ ] CASCADE deletes are used intentionally — deleting a user removes all their content, which is the intended behavior
- [ ] SET NULL is used for soft references where the related entity should survive the deletion of the referenced entity
- [ ] No raw SQL is constructed from user input in repository methods — all queries use GORM parameterized queries
- [ ] Indexes on frequently queried columns prevent full table scans on large datasets
- [ ] The `status` column on users (active/disabled/banned) is checked by the auth middleware to block disabled accounts

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Go — Model | `internal/model/user.go` | Existing — no changes (User, RefreshToken already defined) |
| Go — Model | `internal/model/password_reset_token.go` | Create — PasswordResetToken struct |
| Go — Model | `internal/model/staff.go` | Create — StaffProfile, StaffPortfolioPhoto, StaffSocialLink structs |
| Go — Model | `internal/model/post.go` | Create — Post, Comment, Like, SavedPost structs |
| Go — Model | `internal/model/story.go` | Create — Story, StoryView structs |
| Go — Model | `internal/model/follow.go` | Create — Follow struct |
| Go — Model | `internal/model/point.go` | Create — PointPackage, UserPointBalance, PointTransaction structs |
| Go — Model | `internal/model/gift.go` | Create — Gift struct |
| Go — Model | `internal/model/tip.go` | Create — TipHistory, Shard, ShardRedemption structs |
| Go — Model | `internal/model/live_stream.go` | Create — LiveStream, LiveStreamComment, LiveCollabSession, CollabParticipant structs |
| Go — Model | `internal/model/booking.go` | Create — ServiceMenu, Booking, Coupon structs |
| Go — Model | `internal/model/review.go` | Create — Review, ReviewImage structs |
| Go — Model | `internal/model/chat.go` | Create — Conversation, Message structs |
| Go — Model | `internal/model/notification.go` | Create — DeviceToken, Notification structs |
| Go — Model | `internal/model/ranking.go` | Create — LiverLeagueInfo, GifterLeagueInfo structs |
| Go — Model | `internal/model/company.go` | Create — Company, Store, StoreImage structs |
| Go — Model | `internal/model/headhunt.go` | Create — HeadhuntOffer, StoreStaffOffer structs |
| Go — Model | `internal/model/subscription.go` | Create — SubscriptionPlan, UserSubscription structs |
| Go — Model | `internal/model/report.go` | Create — Report, ContentModeration structs |
| Go — Model | `internal/model/checkin.go` | Create — DailyCheckin struct |
| Go — Repository | `internal/repository/staff_repository.go` | Create — StaffRepository struct and constructor |
| Go — Repository | `internal/repository/post_repository.go` | Create — PostRepository struct and constructor |
| Go — Repository | `internal/repository/story_repository.go` | Create — StoryRepository struct and constructor |
| Go — Repository | `internal/repository/booking_repository.go` | Create — BookingRepository struct and constructor |
| Go — Repository | `internal/repository/tip_repository.go` | Create — TipRepository struct and constructor |
| Go — Repository | `internal/repository/live_stream_repository.go` | Create — LiveStreamRepository struct and constructor |
| Go — Repository | `internal/repository/chat_repository.go` | Create — ChatRepository struct and constructor |
| Go — Repository | `internal/repository/review_repository.go` | Create — ReviewRepository struct and constructor |
| Go — Repository | `internal/repository/follow_repository.go` | Create — FollowRepository struct and constructor |
| Go — Repository | `internal/repository/notification_repository.go` | Create — NotificationRepository struct and constructor |
| Go — Repository | `internal/repository/ranking_repository.go` | Create — RankingRepository struct and constructor |
| Go — Repository | `internal/repository/point_repository.go` | Create — PointRepository struct and constructor |
| Go — Repository | `internal/repository/headhunt_repository.go` | Create — HeadhuntRepository struct and constructor |
| Go — Repository | `internal/repository/subscription_repository.go` | Create — SubscriptionRepository struct and constructor |
| Go — Repository | `internal/repository/report_repository.go` | Create — ReportRepository struct and constructor |
| Go — Entry | `main.go` | Modify — Add all new model structs to db.AutoMigrate() call |
| DB Migration | `migrations/20260311000001_create_password_reset_tokens_table.up.sql` | Create |
| DB Migration | `migrations/20260311000001_create_password_reset_tokens_table.down.sql` | Create |
| DB Migration | `migrations/20260311000002_create_gifts_table.up.sql` through `migrations/20260311000043_create_daily_checkins_table.down.sql` | Create — 43 new migration pairs (86 files total) |
| Seeds | `seeds/demo_data.sql` | Modify — Add seed data for staff_profiles, gifts, point_packages, and subscription_plans |

---

## 11. Acceptance Criteria

- [ ] All 38 tables defined in Section 6 exist in the PostgreSQL database after running AutoMigrate
- [ ] Every table has a ULID primary key (VARCHAR(26)) except `user_point_balances` which uses `user_id` as PK
- [ ] All foreign key constraints are enforced — inserting a record with a non-existent FK value returns an error
- [ ] CASCADE deletes work correctly: deleting a user removes their posts, comments, likes, follows, bookings, tips, and all other owned records
- [ ] SET NULL deletes work correctly: deleting a gift sets `tip_histories.gift_id` to NULL without deleting the tip record
- [ ] CHECK constraints are enforced: `reviews.rating` rejects values outside 1–5, `follows` rejects self-follow, `stores.tip_share_rate` rejects values outside 0–10
- [ ] Unique constraints are enforced: duplicate staff_number, duplicate follow pair, duplicate like on same post by same user all return errors
- [ ] All migration up files run successfully in order on a clean database
- [ ] All migration down files run successfully in reverse order, leaving no tables behind
- [ ] The `db.AutoMigrate()` call in `main.go` includes all model structs and completes without errors
- [ ] Seed data inserts successfully after migrations, including demo staff profile, sample gifts, and point packages
- [ ] The `users` table schema remains unchanged and is backward-compatible with existing AUTH-01 and AUTH-02 functionality
- [ ] Every GORM model struct has correct `gorm` tags matching the column definitions in Section 6
- [ ] Indexes defined in Section 6 are created on the database (verified via `\di` in psql or equivalent)
