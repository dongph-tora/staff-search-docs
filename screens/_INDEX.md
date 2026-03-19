# Screen & Feature Spec Index

This is the master index for all implementation specs in `docs/screens/`. Read this file first before picking up any ticket.

---

## How to Use This Folder

**Before implementing any ticket:**
1. Read `_CONVENTIONS.md` — defines project-wide rules for Go structure, error format, JWT, ApiClient, AuthProvider, navigation, and token storage.
2. Read the relevant spec file for the ticket you are implementing.
3. If the spec lists dependencies, read those specs too.

**Writing a new spec:**
1. Read `_SPEC_RULES.md` — mandatory rules for structure, language, and completeness.
2. Copy `_TEMPLATE.md` and rename it using the convention below.
3. Fill in every `<placeholder>`. Delete OPTIONAL sections that do not apply.
4. Run the self-review checklist in `_SPEC_RULES.md` Rule 11 before finishing.
5. Update this index: change `🔲` to `✅`, add the file to the Files table at the bottom, and update the Progress Tracker count.

**File naming convention:**
```
<GROUP>-<NN>-<short-slug>.md
```
Examples: `AUTH-03-apple-signin.md`, `BOOK-01-create-booking.md`, `LIVE-03-broadcast-screen.md`

---

## Spec Status

Legend: `✅ Specced` · `🔲 Not yet specced`

### AUTH — Authentication

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| AUTH-01 | Email / Password login, register, password reset | ✅ `AUTH-01-email-password.md` | — |
| AUTH-02 | Google Sign-In (iOS + Android + Web) | ✅ `AUTH-02-google-signin.md` | AUTH-01 |
| AUTH-03 | Apple Sign-In (iOS required) | 🔲 | AUTH-01, AUTH-02 |
| AUTH-04 | Registration flow — User screen | 🔲 | AUTH-01 |
| AUTH-05 | Registration flow — Staff screen (job type, photo, intro video) | 🔲 | AUTH-01, STAFF-01 |
| AUTH-06 | Login / Logout full flow + auto-login | 🔲 | AUTH-01 |
| AUTH-07 | Password reset (covered in AUTH-01 Section 5.7–5.9) | ✅ In AUTH-01 | AUTH-01 |
| AUTH-08 | Role-based Access Control (user / staff / admin) | 🔲 | AUTH-01 |
| AUTH-09 | Profile photo upload | 🔲 | AUTH-01, DB-05 |
| AUTH-10 | Email verification after registration | 🔲 | AUTH-01 |

### DB — Backend & Database

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| DB-01 | Database schema design (all tables) | ✅ `DB-01-database-schema.md` | AUTH-01 |
| DB-02 | Migrate all services from LocalStorage to real DB | ✅ `DB-02-migrate-localstorage-to-db.md` | DB-01 |
| DB-03 | Security rules & permissions | 🔲 | AUTH-08 |
| DB-04 | Query indexes & optimization | 🔲 | DB-01 |
| DB-05 | Storage setup for media (images, videos) | ✅ `DB-05-storage-setup-for-media.md` | DB-01 |
| DB-06 | Pagination for lists (infinite scroll) | 🔲 | DB-01 |
| DB-07 | Real-time listeners (chat, notifications, presence) | 🔲 | DB-01 |

### STAFF — Staff Profile Management

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| STAFF-01 | Staff profile CRUD | ✅ `STAFF-01-staff-profile-crud.md` | DB-01 |
| STAFF-02 | Unique 6-digit Staff ID generation | ✅ `STAFF-02-staff-number-generation.md` | STAFF-01 |
| STAFF-03 | Job type selection (20+ categories) | ✅ `STAFF-03-job-type-selection.md` | STAFF-01 |
| STAFF-04 | Online / Offline presence (real-time) | 🔲 | STAFF-01, DB-07 |
| STAFF-05 | Portfolio photo upload | ✅ `STAFF-05-portfolio-photo-upload.md` | STAFF-01, DB-05 |
| STAFF-06 | Intro video upload (max 60s) | 🔲 | STAFF-01, DB-05 |
| STAFF-07 | Staff profile screen connected to DB | ✅ `STAFF-07-staff-profile-screen-connected.md` | STAFF-01 |
| STAFF-08 | Fan history screen | 🔲 | STAFF-01, PAY-08 |

### FEED — Posts & Stories

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| FEED-01 | Create post (image / video, caption) | ✅ `FEED-01-create-post.md` | DB-01, DB-05 |
| FEED-02 | TikTok-style vertical swipe feed | ✅ `FEED-02-tiktok-feed.md` | FEED-01 |
| FEED-03 | Like / Comment / Save | 🔲 | FEED-01 |
| FEED-04 | Stories — create and view (24h expiry) | 🔲 | FEED-01 |
| FEED-05 | Home feed discovery algorithm | 🔲 | FEED-01 |
| FEED-06 | Optimised video player (autoplay, preload) | 🔲 | FEED-02 |

### PAY — Tips, Gifts & Payments

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| PAY-01 | Payment service account setup | 🔲 | — |
| PAY-02 | Backend — PaymentIntent API | 🔲 | PAY-01 |
| PAY-03 | Flutter Payment SDK integration | 🔲 | PAY-02 |
| PAY-04 | Fixed tip amounts (100 / 300 / 500 / 1000 / 3000 / 5000 ¥) | 🔲 | PAY-03 |
| PAY-05 | Gift catalog connected to payment | 🔲 | PAY-03 |
| PAY-06 | Webhook handler (payment succeeded / failed) | 🔲 | PAY-02 |
| PAY-07 | Tip from post & profile screens | 🔲 | PAY-04 |
| PAY-08 | Tip history | 🔲 | PAY-04, DB-01 |
| PAY-09 | Staff wallet / balance screen | 🔲 | PAY-08 |
| PAY-10 | QR Code tip flow | 🔲 | PAY-04 |
| PAY-11 | Gift animation effects (fly-up animation) | 🔲 | PAY-05 |
| PAY-12 | Withdrawal request (MVP — manual admin) | 🔲 | PAY-09 |

### LIVE — Live Streaming

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| LIVE-01 | Agora SDK setup and configuration | 🔲 | — |
| LIVE-02 | Backend — Agora token generator | 🔲 | LIVE-01 |
| LIVE-03 | Live broadcast screen — Staff side | 🔲 | LIVE-02 |
| LIVE-04 | Live viewer screen — User side | 🔲 | LIVE-02 |
| LIVE-05 | Live stream state in DB (status, viewer count) | 🔲 | LIVE-02, DB-01 |
| LIVE-06 | Tips & gifts during live stream (real-time animation) | 🔲 | LIVE-04, PAY-05 |
| LIVE-07 | Live list screen (who is currently live) | 🔲 | LIVE-05 |
| LIVE-08 | End stream & cleanup | 🔲 | LIVE-03 |

### SEARCH — Search & Discovery

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| SEARCH-01 | Text search (name, job type, keyword) | 🔲 | DB-01 |
| SEARCH-02 | GPS / Location-based search | ✅ `SEARCH-02-gps-location-search.md` | DB-01 |
| SEARCH-03 | Filter & Sort | 🔲 | SEARCH-01 |
| SEARCH-04 | Search by unique staff number (6 digits) | 🔲 | STAFF-02 |
| SEARCH-05 | Map view integration | 🔲 | SEARCH-02 |

### CHAT — Messaging

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| CHAT-01 | Real-time chat backend | 🔲 | DB-07 |
| CHAT-02 | Message list / Inbox | 🔲 | CHAT-01 |
| CHAT-03 | One-on-one chat screen | 🔲 | CHAT-01 |
| CHAT-04 | Media in chat (image upload) | 🔲 | CHAT-01, DB-05 |
| CHAT-05 | Chat push notifications | 🔲 | CHAT-01, NOTIF-01 |

### BOOK — Booking & Appointments

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| BOOK-01 | Create booking connected to DB | 🔲 | DB-01 |
| BOOK-02 | Booking cancellation | 🔲 | BOOK-01 |
| BOOK-03 | Staff calendar / availability slots | 🔲 | BOOK-01 |
| BOOK-04 | Booking status flow (pending → confirmed → completed) | 🔲 | BOOK-01 |
| BOOK-05 | Booking notifications | 🔲 | BOOK-01, NOTIF-01 |

### REV — Reviews & Ratings

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| REV-01 | Write review connected to DB | 🔲 | BOOK-04 |
| REV-02 | Rating aggregate on staff profile | 🔲 | REV-01 |
| REV-03 | Reviews displayed on staff profile | 🔲 | REV-01 |
| REV-04 | Mutual review prompt after booking completes | 🔲 | BOOK-04 |

### RANK — Rankings

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| RANK-01 | Ranking by tip received (real-time or batch) | 🔲 | PAY-08 |
| RANK-02 | Ranking screen connected to DB | 🔲 | RANK-01 |
| RANK-03 | Ranking by job category | 🔲 | RANK-01 |

### NOTIF — Push Notifications

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| NOTIF-01 | Push notification setup (iOS + Android) | 🔲 | — |
| NOTIF-02 | Device token management | 🔲 | NOTIF-01, AUTH-01 |
| NOTIF-03 | Backend notification sender | 🔲 | NOTIF-01 |
| NOTIF-04 | Notification screen connected to DB | 🔲 | NOTIF-03 |
| NOTIF-05 | In-app unread badge | 🔲 | NOTIF-04 |

### SOCIAL — Follow & Social Graph

| Ticket | Description | Spec | Depends On |
|---|---|---|---|
| SOCIAL-01 | Follow / Unfollow connected to DB | 🔲 | DB-01 |
| SOCIAL-02 | Following / Followers list screen | 🔲 | SOCIAL-01 |
| SOCIAL-03 | Follow push notification | 🔲 | SOCIAL-01, NOTIF-01 |

### TEST — Testing & Release

| Ticket | Description | Spec |
|---|---|---|
| TEST-01 | Unit tests — Services | 🔲 |
| TEST-02 | Integration testing | 🔲 |
| TEST-03 | Performance optimization | 🔲 |
| TEST-04 | Security audit | 🔲 |
| TEST-05 | iOS App Store submission | 🔲 |
| TEST-06 | Android Play Store submission | 🔲 |
| TEST-07 | Bug fixes & stabilization | 🔲 |

---

## Dependency Graph — Phase 1 Critical Path

```
AUTH-01 ──────────────────────────────────────────────────────► AUTH-08
   │                                                                │
   ├──► AUTH-02 (Google)                                           │
   ├──► AUTH-03 (Apple)                                            │
   ├──► AUTH-04 (User register screen)                             │
   └──► AUTH-05 ──► STAFF-01 ──► STAFF-02..08                     │
                                                                   │
DB-01 ◄────────────────────────────────────────────────── AUTH-01 ─┘
  │
  ├──► DB-05 (Storage) ──► STAFF-05/06, FEED-01, CHAT-04
  ├──► DB-07 (Realtime) ──► CHAT-01, STAFF-04
  │
  ├──► FEED-01 ──► FEED-02..06
  ├──► PAY-01 ──► PAY-02 ──► PAY-03 ──► PAY-04..12
  ├──► LIVE-01 ──► LIVE-02 ──► LIVE-03..08
  ├──► BOOK-01 ──► BOOK-02..05 ──► REV-01..04
  ├──► SEARCH-01..05
  └──► NOTIF-01 ──► NOTIF-02..05
```

---

## Progress Tracker

| Group | Specced | Total | % |
|---|---|---|---|
| AUTH | 2 | 10 | 20% |
| DB | 3 | 7 | 43% |
| STAFF | 5 | 8 | 63% |
| FEED | 2 | 6 | 33% |
| PAY | 0 | 12 | 0% |
| LIVE | 0 | 8 | 0% |
| SEARCH | 1 | 5 | 20% |
| CHAT | 0 | 5 | 0% |
| BOOK | 0 | 5 | 0% |
| REV | 0 | 4 | 0% |
| RANK | 0 | 3 | 0% |
| NOTIF | 0 | 5 | 0% |
| SOCIAL | 0 | 3 | 0% |
| **Total** | **13** | **81** | **16%** |

---

## Files in This Folder

| File | Purpose |
|---|---|
| `_INDEX.md` | This file — master index and reading guide |
| `_CONVENTIONS.md` | Project-wide coding conventions (read before everything else) |
| `_SPEC_RULES.md` | Mandatory rules for writing specs — structure, language, completeness |
| `_TEMPLATE.md` | Blank spec template — copy this when writing a new spec |
| `AUTH-01-email-password.md` | Email/password login, register, password reset, PP consent modal |
| `AUTH-02-google-signin.md` | Google OAuth for iOS, Android, Web |
| `DB-01-database-schema.md` | Full database schema — all 38 tables, columns, indexes, FK relationships, migration plan |
| `DB-02-migrate-localstorage-to-db.md` | User profile update endpoint; AppModeService migration pattern; SharedPreferences deprecation strategy |
| `DB-05-storage-setup-for-media.md` | Cloudflare R2/S3 presigned URL upload API; Flutter UploadService; storage configuration |
| `STAFF-01-staff-profile-crud.md` | Staff profile create, update, get (own + public) — GORM model, handler, service, repository |
| `STAFF-02-staff-number-generation.md` | Unique 6-digit staff number generation algorithm using crypto/rand with uniqueness check |
| `STAFF-03-job-type-selection.md` | 21 job categories — backend list endpoint, Flutter dropdown widget, category chip filter row |
| `STAFF-05-portfolio-photo-upload.md` | Portfolio photo add, delete, reorder — R2 presigned upload, 12-photo limit |
| `STAFF-07-staff-profile-screen-connected.md` | Staff profile screen with live API data, owner/visitor modes, loading/error states |
| `FEED-01-create-post.md` | Post creation with image/video upload via R2 presigned URL, caption, feed API endpoints |
| `FEED-02-tiktok-feed.md` | Full-screen vertical PageView feed, video auto-play, cursor pagination, category filtering |
| `SEARCH-02-gps-location-search.md` | GPS-based distance display on staff cards; lat/lng added to StaffProfileResponse and UpdateStaffProfileRequest; client-side distance filter |
