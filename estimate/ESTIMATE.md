# ESTIMATE — STAFF SEARCH APP (Staff Search)

> **Cập nhật:** 2026-02
> **Tài liệu tham chiếu:** staff_search_presentation_20260219, staff_search_app_presentation_20251011, staff_sns_tip_app_20251014, staffsearch.pdf, engineer-requirements.txt
> **Nền tảng:** Flutter (iOS + Android + Web)
> **Backend:** Chưa có — xem gợi ý ở Mục 0

---

## MỤC LỤC

- [0. Gợi ý Backend](#0-goi-y-backend)
- [1. Tổng quan trạng thái hiện tại](#1-tong-quan-trang-thai-hien-tai)
- [2. Phân loại công việc](#2-phan-loai-cong-viec)
- [3. Phase 1 — MVP](#3-phase-1--mvp)
- [4. Phase 2 — Tăng trưởng](#4-phase-2--tang-truong)
- [5. Phase 3 — Mở rộng](#5-phase-3--mo-rong)
- [6. Tóm tắt estimate theo phase](#6-tom-tat-estimate-theo-phase)
- [7. Rủi ro & Ghi chú](#7-rui-ro--ghi-chu)

---

## 0. Gợi ý Backend

> Backend đã được khởi tạo: **Go + Fiber v3 + GORM + PostgreSQL**, theo clean architecture (handler → service → repository). Auth layer (login, register, refresh, logout) đã hoàn thành. Cần tiếp tục xây dựng các domain API còn lại.

---

### Yêu cầu kỹ thuật cốt lõi

| Yêu cầu | Mức độ |
| --- | :---: |
| Real-time (chat, live viewer count, tip animation) | CAO |
| Authentication & phân quyền (User / Staff / Admin) | CAO |
| Thanh toán & payout (tip, subscription, rút tiền) | CAO |
| Live streaming (broadcast 1-to-many) | CAO |
| File upload & media storage (ảnh, video) | TRUNG BÌNH |
| Push notifications | TRUNG BÌNH |
| Full-text search | TRUNG BÌNH |
| Admin dashboard & reporting | THẤP (Phase 2) |

---

### Gợi ý ngôn ngữ & framework ⭐

| Ngôn ngữ / Framework | Lý do phù hợp |
| --- | --- |
| **Go (Golang) + Gin / Fiber** | Hiệu năng cao nhất, latency thấp, xử lý concurrency tốt — lý tưởng cho real-time & streaming. Scale dễ dàng lên hàng triệu request |
| **Node.js + Fastify** (TypeScript) | Ecosystem lớn, nhiều lib sẵn có cho payment/streaming, dễ tuyển dụng, phù hợp nếu team quen JS |
| **Python + FastAPI** | Nhanh prototype, phù hợp nếu sau này cần thêm AI/ML matching |

> **Khuyến nghị:** **Go** nếu ưu tiên hiệu năng & scale, **Node.js** nếu ưu tiên tốc độ phát triển và dễ tìm developer.

---

### Kiến trúc đề xuất

```
Flutter App (iOS / Android / Web)
        |
   API Gateway
        |
 +--------------+------------------+------------------+
 |              |                  |                  |
Auth Service  Core API          Payment Service   Notification Service
(JWT/OAuth)  (Go / Node.js)    (Stripe)          (FCM / APNs)
                 |
    +-----------+-----------+
    |           |           |
PostgreSQL   Redis       Object Storage
(main DB)  (cache +     (S3 / R2 — anh,
           session +     video, avatar)
           real-time
           pub/sub)
                            |
                    Streaming Server
                    (Agora / LiveKit / WebRTC)
```

---

### Database

| Công nghệ | Vai trò | Ghi chú |
| --- | --- | --- |
| **PostgreSQL** | Database chính | Lưu users, staff, posts, bookings, tips, reviews. Quan hệ rõ ràng, dễ query phức tạp |
| **Redis** | Cache + Session + Pub/Sub | Lưu session auth, cache hot data (ranking, search), real-time events cho chat & live stream |
| **Elasticsearch** | Full-text search | Search theo tên, job type, kỹ năng — sẽ cần ở Phase 2 khi data lớn |
| **Object Storage** (S3 / R2) | Media files | Lưu ảnh, video, avatar. Cloudflare R2 rẻ hơn S3 và không tính phí bandwidth |

---

### Third-party services (không thay thế được)

| Service | Chức năng | Ghi chú |
| --- | --- | --- |
| **Stripe** | Thanh toán, subscription, payout | Tiêu chuẩn công nghiệp, hỗ trợ JPY, có Stripe Connect cho rút tiền |
| **Agora.io** hoặc **LiveKit** | Live streaming SDK | Agora là lựa chọn nên cho Flutter; LiveKit là open-source có thể self-host |
| **FCM / APNs** | Push notifications | Bắt buộc cho iOS + Android |
| **SendGrid** hoặc **AWS SES** | Email (welcome, reset password) | Dịch vụ email transactional |

---

### Deploy & Infrastructure

| Lựa chọn | Phù hợp khi | Ghi chú |
| --- | --- | --- |
| **Railway / Render** | Phase 1 MVP | Đơn giản nhất, tự động deploy, miễn phí tier có sẵn |
| **AWS (EC2 + RDS + S3)** | Phase 2 trở đi | Linh hoạt hơn, giá tốt hơn khi scale, nhưng phức tạp hơn |
| **Google Cloud Run** | Phase 1-2 | Serverless container, trả theo usage, scale tự động |

> **Khuyến nghị deploy:** Bắt đầu với **Railway hoặc Render** cho Phase 1 (rẻ, nhanh setup). Migrate sang **AWS** hoặc **GCP** khi đạt 10,000+ users.

---

## 1. Tổng quan trạng thái hiện tại

| Hạng mục | Trạng thái | Ghi chú |
| --- | --- | --- |
| **Cấu trúc project Flutter** | ✅ Hoàn thành | Multi-platform: iOS, Android, Web, macOS |
| **Models dữ liệu** | ✅ Hoàn thành | 17 model files |
| **UI màn hình** | ✅ ~70% | 40+ screens, thiếu một số flow quan trọng |
| **Service layer** | ⚠️ Một phần | 11 service files, dùng localStorage thay backend thật |
| **Backend** | ⚠️ Foundation done | Go + Fiber + GORM clean architecture built. AUTH endpoints done (login, register, refresh, logout). Còn thiếu: payment, streaming, chat, search, booking APIs |
| **Xác thực người dùng** | ⚠️ Backend done | Backend auth endpoints hoàn thành (JWT, bcrypt, refresh token rotation). Flutter chưa kết nối |
| **Thanh toán** | ❌ Chưa có | UI có, chưa có payment backend |
| **Live Streaming** | ❌ Chưa có | Màn hình UI có, chưa tích hợp streaming SDK |
| **Push Notifications** | ❌ Chưa có | Chưa setup |
| **QR Code Payment** | ❌ Chưa có | QR generate có, payment flow chưa có |
| **Admin Dashboard** | ❌ Chưa có | Màn hình stub, không có logic |
| **Video Upload** | ❌ Chưa có | Chưa có storage backend |

> **Kết luận:** Project hoàn thành khoảng **~25-30% tổng khối lượng thật sự**. UI shell ~70%, backend foundation (clean architecture + auth) đã xong. Còn thiếu: kết nối Flutter ↔ Backend, payment, streaming, chat, search APIs.

---

## 2. Phân loại công việc

### Chưa làm — Critical (Blocking Production)

- Authentication (đăng nhập / đăng ký thật sự)
- Backend API & database (hiện tại toàn bộ là localStorage và mock data)
- Payment processing (tip, gift, subscription)
- Live streaming SDK
- Push notifications
- File upload (ảnh, video) lên storage

### Làm một phần — Partial (Cần hoàn thiện)

- Chat / Messaging (hiện dùng polling + localStorage, cần real-time backend)
- Booking cancellation (có TODO comment, chưa implement)
- Mode switching User — Staff (có bug)
- Location / GPS search (đang dùng hardcoded data)
- Admin content moderation (UI only)
- Email notifications (chưa có email service)

### Đã làm — Done (Có thể cần refactor)

- Staff profile UI
- Search & filter UI
- Gift catalog UI
- Review & rating UI
- Story viewer UI
- Booking create/view UI
- Ranking screen UI
- Data models (17 files)
- Web pages (login/register/landing page)

---

## Ghi chú về cột Estimate

> Bảng task từ Mục 3 trở đi có **2 cột estimate:**
>
> - **Estimate (Manual)** — Dev tự code hoàn toàn, không dùng AI
> - **🤖 Vibe Coding** — Dev dùng AI agent (Claude Code, Cursor, Copilot...) để scaffold, generate boilerplate, refactor, viết test. Dev chỉ cần **hướng dẫn, review, chỉnh sửa logic đặc thù và test**
>
> Vibe coding tiết kiệm **~65-75%** thời gian ở các task tiêu chuẩn (auth flow, CRUD, boilerplate, screen refactor). Các task phức tạp (streaming infra, AI/ML, system integration) tiết kiệm ít hơn (~30-40%). Lý do: với AI agent, một dev có thể scaffold toàn bộ auth flow, CRUD services, và refactor màn hình trong chưa đến 1 tiếng mỗi task — AI generate code, dev review và chỉnh logic đặc thù. Các task cần tư duy kiến trúc sâu, business logic phức tạp, hoặc debug hệ thống distributed thì tiết kiệm ít hơn (~20-30%).

---

## 3. Phase 1 — MVP

> **Mục tiêu:** Ra mắt sản phẩm với chức năng cốt lõi
> **Timeline dự kiến:** 8-10 tuần
> **Launch target:** 2026 Q1

---

### 3.1 Hệ thống Xác thực (Authentication)

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[AUTH-01]** Auth — Email/Password | Setup auth service, bật Email/Password provider, kết nối Flutter app | 1 ngày | 0.25 ngày |
| **[AUTH-02]** Auth — Google Sign-in | Tích hợp Google OAuth cho iOS + Android + Web | 1.5 ngày | 0.25 ngày |
| **[AUTH-03]** Auth — Apple Sign-in | Tích hợp Apple OAuth (iOS bắt buộc) | 1 ngày | 0.25 ngày |
| **[AUTH-04]** Registration flow — User | Màn hình đăng ký user với validation, lưu vào DB | 1 ngày | 0.25 ngày |
| **[AUTH-05]** Registration flow — Staff | Màn hình đăng ký staff: job type, profile photo, intro video | 1.5 ngày | 0.25 ngày |
| **[AUTH-06]** Login / Logout flow | Luồng đăng nhập, lưu session, tự động đăng nhập | 1 ngày | 0.25 ngày |
| **[AUTH-07]** Password Reset | Email password reset | 0.5 ngày | 0.25 ngày |
| **[AUTH-08]** Role-based Access Control | Phân quyền User / Staff / Admin | 1 ngày | 0.25 ngày |
| **[AUTH-09]** Profile photo upload | Upload ảnh avatar lên storage + lưu URL vào DB | 1 ngày | 0.25 ngày |
| **[AUTH-10]** Email verification | Gửi verification email sau đăng ký | 0.5 ngày | 0.25 ngày |

**Subtotal AUTH:** ~10 ngày manual | **~2.5 ngày vibe coding**

---

### 3.2 Backend & Database

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[DB-01]** Thiết kế database schema | Thiết kế collections/tables: users, staff, posts, tips, bookings, reviews, chats, notifications | 2 ngày | 0.5 ngày |
| **[DB-02]** Migration từ localStorage sang DB thật | Refactor tất cả service files (11 files) từ localStorage sang backend thật | 4 ngày | 0.75 ngày |
| **[DB-03]** Security rules & permissions | Viết rules nghiêm ngặt: read/write permissions theo role | 1.5 ngày | 0.5 ngày |
| **[DB-04]** Query indexes & optimization | Tối ưu query indexes cho search/filter/ranking | 1 ngày | 0.25 ngày |
| **[DB-05]** Storage setup cho media | Setup bucket, security rules cho images, videos | 0.5 ngày | 0.25 ngày |
| **[DB-06]** Pagination cho danh sách | Implement pagination (infinite scroll) cho staff list, posts, reviews | 2 ngày | 0.5 ngày |
| **[DB-07]** Real-time listeners | Setup real-time streams cho chat, notifications, online status | 1.5 ngày | 0.5 ngày |

**Subtotal DB:** ~12.5 ngày manual | **~3.25 ngày vibe coding**

---

### 3.3 Quản lý Profile Staff

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[STAFF-01]** Staff profile CRUD thật | Create/Read/Update/Delete profile với DB thật | 2 ngày | 0.5 ngày |
| **[STAFF-02]** Unique Staff ID / Search Number | Generate unique 6-digit number cho mỗi staff khi đăng ký | 0.5 ngày | 0.25 ngày |
| **[STAFF-03]** Job type selection | Dropdown/picker cho 20+ job categories (đã có mock data, cần kết nối) | 0.5 ngày | 0.25 ngày |
| **[STAFF-04]** Online/Offline status (ON/OFF) | Real-time presence system | 1 ngày | 0.25 ngày |
| **[STAFF-05]** Portfolio photos upload | Upload nhiều ảnh portfolio lên storage | 1 ngày | 0.25 ngày |
| **[STAFF-06]** Intro video upload | Upload video giới thiệu ngắn (max 60s) lên storage | 1.5 ngày | 0.5 ngày |
| **[STAFF-07]** Staff profile screen kết nối DB | Refactor staff_detail_screen.dart từ mock data sang DB thật | 2 ngày | 0.5 ngày |
| **[STAFF-08]** Fan History | Hiển thị danh sách fan / người đã gửi tip nhiều lần | 1 ngày | 0.25 ngày |

**Subtotal STAFF:** ~9.5 ngày manual | **~2.75 ngày vibe coding**

---

### 3.4 Feed & Nội dung (Posts / Stories)

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[FEED-01]** Tạo post với ảnh/video | Màn hình tạo post: chọn media, caption, publish lên DB | 2 ngày | 0.5 ngày |
| **[FEED-02]** TikTok-style vertical swipe feed | Vertical scroll feed với PageView, lazy loading, video autoplay | 3 ngày | 1 ngày |
| **[FEED-03]** Like / Comment / Save | Real-time reactions với DB, counter update | 1.5 ngày | 0.5 ngày |
| **[FEED-04]** Stories tạo và xem | Upload Story (24h tự xóa), story ring indicator, swipe viewer | 2.5 ngày | 0.75 ngày |
| **[FEED-05]** Feed discovery / Home screen | Thuật toán hiển thị: mới nhất, gần đây, đang follow | 2 ngày | 0.5 ngày |
| **[FEED-06]** Video player tối ưu | Autoplay, mute default, progress bar, preload next | 1.5 ngày | 0.5 ngày |

**Subtotal FEED:** ~12.5 ngày manual | **~3.75 ngày vibe coding**

---

### 3.5 Hệ thống Tip / Gift (Payment)

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[PAY-01]** Payment service setup | Đăng ký tài khoản payment, test/live keys, cấu hình webhook | 0.5 ngày | 0.25 ngày |
| **[PAY-02]** Backend — PaymentIntent API | Backend function tạo payment intent, validate amount | 1.5 ngày | 0.5 ngày |
| **[PAY-03]** Flutter Payment SDK | Tích hợp payment SDK, PaymentSheet UI | 2 ngày | 0.5 ngày |
| **[PAY-04]** Fixed tip amounts (MVP) | Các mức tip cố định: 100, 300, 500, 1000, 3000, 5000 yen | 1 ngày | 0.25 ngày |
| **[PAY-05]** Gift catalog kết nối thanh toán | Kết nối 20+ gift items trong tiktok_gift_screen.dart với payment | 1.5 ngày | 0.25 ngày |
| **[PAY-06]** Webhook handler | Backend xử lý webhook: payment succeeded, failed | 1 ngày | 0.25 ngày |
| **[PAY-07]** Tip lên Post & Profile | Cho phép tip từ cả màn hình post lẫn profile | 1 ngày | 0.25 ngày |
| **[PAY-08]** Tip history | Lưu lịch sử tip vào DB, hiển thị tip_history_screen.dart | 1 ngày | 0.25 ngày |
| **[PAY-09]** Staff wallet / balance | Hiển thị số dư nhận được, tổng tip theo tháng | 1 ngày | 0.25 ngày |
| **[PAY-10]** QR Code tip flow | Staff QR → Customer scan → Tip payment flow hoàn chỉnh | 2 ngày | 0.5 ngày |
| **[PAY-11]** Gift animation effects | Flutter animation khi gửi gift (hiệu ứng bay lên màn hình) | 1.5 ngày | 0.5 ngày |
| **[PAY-12]** Withdrawal request (MVP — manual) | Staff request rút tiền, admin xử lý thủ công (tự động ở Phase 2) | 1 ngày | 0.25 ngày |

**Subtotal PAY:** ~16 ngày manual | **~4 ngày vibe coding**

---

### 3.6 Live Streaming

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[LIVE-01]** Streaming SDK setup | Đăng ký streaming service account, cài SDK, cấu hình app ID | 1 ngày | 0.25 ngày |
| **[LIVE-02]** Backend — Token generator | Backend generate streaming token (bảo mật, có expiry) | 1 ngày | 0.25 ngày |
| **[LIVE-03]** Live broadcast — Staff side | Màn hình phát sóng: camera on/off, mic, end stream | 2 ngày | 0.5 ngày |
| **[LIVE-04]** Live viewer — User side | Màn hình xem live: join channel, comment overlay, tip button | 2 ngày | 0.5 ngày |
| **[LIVE-05]** Live stream state DB | Lưu trạng thái live (đang phát, số viewer, start time) vào DB | 1 ngày | 0.25 ngày |
| **[LIVE-06]** Live tip / gift trong stream | Gửi tip/gift khi đang xem live, hiển thị animation real-time cho tất cả viewer | 2 ngày | 0.5 ngày |
| **[LIVE-07]** Live list screen | Danh sách staff đang live, thumbnail, viewer count | 1 ngày | 0.25 ngày |
| **[LIVE-08]** End stream & cleanup | Kết thúc stream, cập nhật DB, disconnect channel | 0.5 ngày | 0.25 ngày |

**Subtotal LIVE:** ~10.5 ngày manual | **~2.75 ngày vibe coding**

---

### 3.7 Search & Matching

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[SEARCH-01]** Text search kết nối DB | Search theo tên staff, job type, keyword — full-text search | 2 ngày | 0.5 ngày |
| **[SEARCH-02]** GPS / Location-based search | Tích hợp Geolocator thật, tính khoảng cách từ DB geo data | 2 ngày | 0.5 ngày |
| **[SEARCH-03]** Filter & Sort | Filter: job type, rating, khoảng cách, online status; Sort: phổ biến, mới nhất | 1.5 ngày | 0.5 ngày |
| **[SEARCH-04]** Search by unique staff number | Tìm kiếm bằng mã số 6 chữ số | 0.5 ngày | 0.25 ngày |
| **[SEARCH-05]** Map view tích hợp | Kết nối map_search_screen.dart với DB + Geolocator thật | 2 ngày | 0.5 ngày |

**Subtotal SEARCH:** ~8 ngày manual | **~2.25 ngày vibe coding**

---

### 3.8 Messaging / Chat

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[CHAT-01]** Chat real-time DB | Refactor chat_service.dart từ localStorage polling sang real-time backend | 2 ngày | 0.5 ngày |
| **[CHAT-02]** Message list / Inbox | Danh sách conversations, unread badge, last message preview | 1.5 ngày | 0.25 ngày |
| **[CHAT-03]** One-on-one chat screen | Send/receive text messages, timestamp, read status | 1.5 ngày | 0.25 ngày |
| **[CHAT-04]** Media in chat | Gửi ảnh trong chat (upload storage, display cached image) | 1 ngày | 0.25 ngày |
| **[CHAT-05]** Chat notifications | Nhận push notification khi có tin nhắn mới | 1 ngày | 0.25 ngày |

**Subtotal CHAT:** ~7 ngày manual | **~1.5 ngày vibe coding**

---

### 3.9 Booking / Appointment

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[BOOK-01]** Booking tạo mới kết nối DB | Refactor tạo booking từ SharedPreferences sang DB | 1.5 ngày | 0.25 ngày |
| **[BOOK-02]** Booking cancellation | Fix TODO: implement logic hủy booking, cập nhật trạng thái DB | 1 ngày | 0.25 ngày |
| **[BOOK-03]** Staff calendar / availability | Staff thiết lập lịch rảnh, user chọn slot | 2.5 ngày | 0.5 ngày |
| **[BOOK-04]** Booking status flow | Pending → Confirmed → Completed → Cancelled với DB | 1 ngày | 0.25 ngày |
| **[BOOK-05]** Booking notifications | Notify staff khi có booking mới, notify user khi confirmed | 0.5 ngày | 0.25 ngày |

**Subtotal BOOK:** ~6.5 ngày manual | **~1.5 ngày vibe coding**

---

### 3.10 Rating & Review

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[REV-01]** Write review kết nối DB | Refactor write_review_screen.dart lưu vào DB | 1 ngày | 0.25 ngày |
| **[REV-02]** Rating aggregate | Tính trung bình rating, cập nhật staff profile sau mỗi review | 1 ngày | 0.25 ngày |
| **[REV-03]** Review hiển thị trên profile | Load reviews từ DB, paginate | 1 ngày | 0.25 ngày |
| **[REV-04]** Mutual review (sau booking) | Sau khi booking complete, trigger prompt đánh giá cả 2 phía | 1 ngày | 0.25 ngày |

**Subtotal REV:** ~4 ngày manual | **~1 ngày vibe coding**

---

### 3.11 Ranking System

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[RANK-01]** Ranking theo tip nhận được | Tính rank từ tổng tip, cập nhật real-time hoặc batch | 1 ngày | 0.25 ngày |
| **[RANK-02]** Ranking screen kết nối DB | Refactor ranking_screen.dart từ mock data sang DB | 1 ngày | 0.25 ngày |
| **[RANK-03]** Ranking theo job category | Top staff theo từng ngành nghề | 1 ngày | 0.25 ngày |

**Subtotal RANK:** ~3 ngày manual | **~0.75 ngày vibe coding**

---

### 3.12 Push Notifications

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[NOTIF-01]** Push notification setup iOS + Android | Cấu hình push service, upload APNs key, test notification | 1 ngày | 0.25 ngày |
| **[NOTIF-02]** Device token quản lý | Lưu device token vào DB khi user login/logout | 0.5 ngày | 0.25 ngày |
| **[NOTIF-03]** Backend — Notification sender | Backend function gửi push notification (new tip, new message, booking) | 1.5 ngày | 0.5 ngày |
| **[NOTIF-04]** Notification screen kết nối | Refactor notifications_screen.dart từ mock sang DB | 1 ngày | 0.25 ngày |
| **[NOTIF-05]** In-app notification badge | Unread count badge trên tab bar | 0.5 ngày | 0.25 ngày |

**Subtotal NOTIF:** ~4.5 ngày manual | **~1.5 ngày vibe coding**

---

### 3.13 Follow / Social Graph

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[SOCIAL-01]** Follow/Unfollow kết nối DB | Refactor follow_service.dart sang DB | 1 ngày | 0.25 ngày |
| **[SOCIAL-02]** Following / Followers list | Màn hình danh sách following/followers | 1 ngày | 0.25 ngày |
| **[SOCIAL-03]** Follow notification | Notify staff khi có người follow mới | 0.5 ngày | 0.25 ngày |

**Subtotal SOCIAL:** ~2.5 ngày manual | **~0.75 ngày vibe coding**

---

### 3.14 Testing & Release

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[TEST-01]** Unit tests — Services | Test các service functions chính | 2 ngày | 0.5 ngày |
| **[TEST-02]** Integration testing | Test các flow chính: auth → tip, auth → booking, live stream | 2 ngày | 0.5 ngày |
| **[TEST-03]** Performance optimization | DB index tuning, image caching, list pagination | 1.5 ngày | 1 ngày |
| **[TEST-04]** Security audit | Kiểm tra security rules, webhook signature, input validation | 1 ngày | 1 ngày |
| **[TEST-05]** iOS App Store submission | Chuẩn bị metadata, screenshots, App Review | 1.5 ngày | 1 ngày |
| **[TEST-06]** Android Play Store submission | Chuẩn bị metadata, screenshots, Review | 1 ngày | 0.75 ngày |
| **[TEST-07]** Bug fixes & stabilization | Fix bugs phát sinh trong testing | 3 ngày | 1 ngày |

**Subtotal TEST:** ~12 ngày manual | **~5.75 ngày vibe coding**

> ⚠️ Testing & release tiết kiệm ít hơn vì cần tư duy thủ công: review logic, verify edge case, submit store.

---

## 4. Phase 2 — Tăng trưởng

> **Timeline dự kiến:** Tháng 3-6 sau launch
> **Mục tiêu:** Monetization, mở rộng tính năng, B2B

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[P2-01]** Custom tip amounts | Cho phép nhập số tiền tùy ý (hiện chỉ có fixed amounts) | 1 ngày | 0.25 ngày |
| **[P2-02]** Tự động rút tiền (Payout) | Staff kết nối bank account, rút tiền tự động qua payment service | 5 ngày | 2.5 ngày |
| **[P2-03]** Instant withdrawal | Rút tiền tức thì (phí bổ sung) | 2 ngày | 1 ngày |
| **[P2-04]** Tip Coupon System | Tạo, phân phối, redeem tip coupons | 3 ngày | 1.25 ngày |
| **[P2-05]** Personal Advertising Feature | Staff trả 30,000 yen/tháng để hiển thị ưu tiên trên top/story | 3 ngày | 1.25 ngày |
| **[P2-06]** Story Ads | Banner ads trong Story viewer, 50,000+ yen/tuần | 2 ngày | 0.75 ngày |
| **[P2-07]** Staff Subscription Plan | Billing 4,000 yen/tháng hoặc 6,500 yen/tháng qua subscription | 3 ngày | 1.25 ngày |
| **[P2-08]** Live stream recording | Record stream → lưu cloud storage → archive viewing sau khi kết thúc | 5 ngày | 2.5 ngày |
| **[P2-09]** Video compression & streaming | Tối ưu video upload: compression trước upload, HLS adaptive streaming | 3 ngày | 1.5 ngày |
| **[P2-10]** AI-powered matching | Gợi ý staff dựa trên hành vi user | 8 ngày | 5 ngày |
| **[P2-11]** Admin Dashboard | Quản lý user/staff, duyệt nội dung, xem doanh thu, export report | 6 ngày | 2.5 ngày |
| **[P2-12]** Enterprise Plan — Admin panel | Dashboard riêng cho chuỗi cửa hàng: quản lý nhân viên, đánh giá tổng hợp | 7 ngày | 3 ngày |
| **[P2-13]** Evaluation reports (Enterprise) | Báo cáo hiệu suất staff theo tháng (tip, rating, engagement) | 3 ngày | 1.25 ngày |
| **[P2-14]** Company badge / verification | Badge xác minh cho doanh nghiệp tham gia chính thức | 1 ngày | 0.5 ngày |
| **[P2-15]** Headhunt / Offer system | Kết nối headhunt_screen.dart và headhunt_service.dart với DB | 2 ngày | 0.75 ngày |
| **[P2-16]** Tax deduction support | Tích hợp thông tin thuế cho tip (Nhật Bản) | 3 ngày | 1.5 ngày |
| **[P2-17]** Email notification service | Gửi email: welcome, booking, payment, weekly digest | 3 ngày | 1 ngày |
| **[P2-18]** Gifter Level / Tier System | Bronze → Silver → Gold → Platinum fan levels với rewards | 2 ngày | 0.75 ngày |
| **[P2-19]** Content moderation AI | Auto-detect inappropriate content | 4 ngày | 2.5 ngày |
| **[P2-20]** Analytics dashboard | Event tracking, traction metrics, funnel analysis | 2 ngày | 0.75 ngày |

**Subtotal Phase 2:** ~67 ngày manual | **~32 ngày vibe coding**

---

## 5. Phase 3 — Mở rộng

> **Timeline dự kiến:** Tháng 7-18 sau launch
> **Mục tiêu:** International, AI, B2C/B2B scale

| Task | Mô tả | Estimate (Manual) | 🤖 Vibe Coding |
| --- | --- | :---: | :---: |
| **[P3-01]** Multilingual support | i18n cho Tiếng Nhật, Tiếng Anh, Tiếng Trung, Tiếng Hàn | 5 ngày | 2 ngày |
| **[P3-02]** Multi-currency payment | Thanh toán đa tiền tệ: JPY, USD, TWD, KRW | 3 ngày | 1.5 ngày |
| **[P3-03]** Streaming infrastructure scale | WebRTC SFU hoặc RTMP+HLS cho >500 concurrent viewers | 10 ngày | 7 ngày |
| **[P3-04]** RTMP + HLS archive streaming | RTMP ingest → HLS encode → CDN distribution | 8 ngày | 5.5 ngày |
| **[P3-05]** AI matching algorithm v2 | Deep learning model dựa trên interaction history | 15 ngày | 9 ngày |
| **[P3-06]** Skill share feature | Staff dạy/chia sẻ kỹ năng qua video paid content | 7 ngày | 3 ngày |
| **[P3-07]** Payment API cho đối tác | API cho POS, HR systems tích hợp với Staff Search | 10 ngày | 5 ngày |
| **[P3-08]** HR data integration | Tích hợp với hệ thống quản lý nhân sự của chuỗi cửa hàng | 8 ngày | 4 ngày |
| **[P3-09]** Subscription / Recurring tip | User đăng ký fan subscription hàng tháng cho staff | 4 ngày | 1.75 ngày |
| **[P3-10]** Web app responsive | Tối ưu Flutter Web cho desktop + mobile browser | 5 ngày | 2 ngày |

**Subtotal Phase 3:** ~75 ngày manual | **~40.75 ngày vibe coding**

---

## 6. Tóm tắt estimate theo phase

### Phase 1 — MVP (Chi tiết)

| Nhóm tính năng | Số task | Manual (ngày) | 🤖 Vibe Coding (ngày) | Tiết kiệm |
| --- | :---: | :---: | :---: | :---: |
| Authentication | 10 | 10 | 2.5 | 75% |
| Backend & Database | 7 | 12.5 | 3.25 | 74% |
| Staff Profile | 8 | 9.5 | 2.75 | 71% |
| Feed & Posts | 6 | 12.5 | 3.75 | 70% |
| Tip / Gift / Payment | 12 | 16 | 4 | 75% |
| Live Streaming | 8 | 10.5 | 2.75 | 74% |
| Search & Matching | 5 | 8 | 2.25 | 72% |
| Messaging / Chat | 5 | 7 | 1.5 | 79% |
| Booking / Appointment | 5 | 6.5 | 1.5 | 77% |
| Rating & Review | 4 | 4 | 1 | 75% |
| Ranking | 3 | 3 | 0.75 | 75% |
| Push Notifications | 5 | 4.5 | 1.5 | 67% |
| Follow / Social | 3 | 2.5 | 0.75 | 70% |
| Testing & Release | 7 | 12 | 5.75 | 52% |
| **TỔNG PHASE 1** | **88 tasks** | **~119 ngày** | **~38.25 ngày** | **~68%** |

> **Manual:** 1 dev ~6 tháng | 2 devs ~3 tháng
> **🤖 Vibe Coding:** 1 dev ~2 tháng | 2 devs ~3-4 tuần

---

### Phase 2 — Tăng trưởng

| | Manual (ngày) | 🤖 Vibe Coding (ngày) | Tiết kiệm |
| --- | :---: | :---: | :---: |
| **TỔNG PHASE 2** | **~67 ngày** | **~32 ngày** | **~52%** |

> **Manual:** 2 devs ~1.5-2 tháng | **🤖 Vibe Coding:** 2 devs ~3 tuần

---

### Phase 3 — Mở rộng

| | Manual (ngày) | 🤖 Vibe Coding (ngày) | Tiết kiệm |
| --- | :---: | :---: | :---: |
| **TỔNG PHASE 3** | **~75 ngày** | **~41 ngày** | **~45%** |

> Phase 3 tiết kiệm ít hơn vì nhiều task phức tạp: AI model, streaming infra, system integration.

---

### Tổng thể toàn dự án

| Phase | Manual (ngày dev) | 🤖 Vibe Coding (ngày dev) | Timeline Manual (2 devs) | Timeline Vibe (2 devs) |
| --- | :---: | :---: | :---: | :---: |
| Phase 1 — MVP | ~119 ngày | ~38 ngày | ~3 tháng | ~1 tháng |
| Phase 2 — Tăng trưởng | ~67 ngày | ~32 ngày | ~1.5-2 tháng | ~3 tuần |
| Phase 3 — Mở rộng | ~75 ngày | ~41 ngày | ~2 tháng | ~1 tháng |
| **TỔNG** | **~261 ngày dev** | **~111 ngày dev** | **~7-8 tháng** | **~2.5-3 tháng** |

> 1 ngày dev = 8 giờ làm việc thực tế
> Vibe coding tiết kiệm trung bình **~57%** tổng thời gian (lên đến ~68-75% cho Phase 1 standard tasks, ~45-52% cho Phase 2-3 complex tasks)
> Dev vẫn cần review output của AI, test kỹ logic nghiệp vụ và xử lý các edge case phức tạp

---

## 7. Rủi ro & Ghi chú

### Rủi ro kỹ thuật

| Rủi ro | Mức độ | Giải pháp đề xuất |
| --- | :---: | --- |
| Streaming latency cao khi nhiều người xem | Cao | Dùng CDN + load test sớm trước launch |
| Payment service approval & verification | Trung bình | Chuẩn bị tài liệu doanh nghiệp sớm, thử nghiệm Sandbox |
| DB cost tăng khi >10,000 users | Trung bình | Thiết kế schema tối ưu reads, dùng denormalization hợp lý |
| Storage cost cao với video | Cao | Giới hạn video size, dùng compression, xem xét CDN |
| App Store rejection (live streaming, payment) | Trung bình | Đọc kỹ App Review Guidelines về tipping, tuân thủ IAP nếu cần |
| Apple IAP requirement cho in-app purchase | Cao | Tip qua QR (external) có thể tránh được 30% Apple fee |

### Ghi chú quan trọng

1. **Apple IAP:** App Store yêu cầu dùng IAP nếu digital goods được tiêu thụ trong app. Tips gửi qua QR (ra ngoài app) có thể exempt. Cần tham khảo Apple Review trước khi release.

2. **Security rules:** Đây là điểm quan trọng nhất về bảo mật — cần viết từ đầu, không để default.

3. **Backup files:** Dự án có nhiều file backup (staff_app_backup/, staff_app_main.dart, etc.) — nên dọn dẹp trước khi bắt đầu phát triển thật.

4. **State management:** Tài liệu yêu cầu Riverpod nhưng code hiện tại dùng Provider — nên thống nhất trước Phase 1.

5. **Localization:** App hiện tại mix tiếng Nhật và tiếng Anh — cần chuẩn hóa string keys từ sớm.

6. **Câu hỏi cần làm rõ trước khi bắt đầu:**
   - Số lượng kết nối live stream đồng thời tối đa dự kiến? (ảnh hưởng đến streaming infra)
   - Tip model: one-time only hay có subscription?
   - Kế hoạch ra nước ngoài khi nào? (ảnh hưởng đến payment currency)
   - Team hiện tại có backend developer chưa? (quyết định lựa chọn stack)

---

*File này được tạo từ phân tích tài liệu docs/ và codebase thực tế của dự án.*
*Cập nhật lần cuối: 2026-02*
