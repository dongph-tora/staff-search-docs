# Build Guide — Backend (staff-search-api)

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Go | 1.26.1 | `~/go/bin/go1.26.1` (đã cài) |
| PostgreSQL | 14+ | `brew install postgresql@14` |
| Redis | 7+ | `brew install redis` |
| golang-migrate | latest | `brew install golang-migrate` |

> **Lưu ý:** Luôn dùng `~/go/bin/go1.26.1`, không dùng system `go` (có thể là 1.25.x).

---

## 1. Clone & Setup

```bash
cd staff-vibe-project/staff-search-api
```

---

## 2. Cấu hình Environment

```bash
cp .env.example .env
```

Mở `.env` và điền các giá trị:

```env
# Server
APP_ENV=development
APP_PORT=3000
APP_BASE_URL=http://localhost:3000

# PostgreSQL — tạo DB trước (xem bước 3)
DATABASE_URL=postgres://staffsearch:staffsearch@localhost:5432/staffsearch?sslmode=disable

# Redis
REDIS_URL=redis://localhost:6379/0

# JWT — đổi secret trong production
JWT_SECRET=dev-secret-change-in-prod
JWT_ACCESS_EXPIRY=3600
JWT_REFRESH_EXPIRY=2592000

# Google OAuth (bỏ trống nếu không cần Google Sign-In)
GOOGLE_CLIENT_ID=

# SMTP (bỏ trống → dùng NoOpEmailSender, không gửi email thật)
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
SMTP_FROM=noreply@staffsearch.com

# Cloudflare R2 Storage (bỏ trống nếu không cần upload ảnh)
STORAGE_PROVIDER=r2
STORAGE_BUCKET=staffsearch-media
STORAGE_REGION=auto
STORAGE_ACCESS_KEY_ID=
STORAGE_SECRET_ACCESS_KEY=
STORAGE_ENDPOINT=
STORAGE_PUBLIC_URL=https://media.staffsearch.jp
STORAGE_URL_EXPIRY_SECONDS=900
```

> Để chạy local demo không cần điền `STORAGE_*` và `SMTP_*`.

---

## 3. Tạo PostgreSQL Database

```bash
# Khởi động PostgreSQL
brew services start postgresql@14

# Tạo user và database
psql postgres -c "CREATE USER staffsearch WITH PASSWORD 'staffsearch';"
psql postgres -c "CREATE DATABASE staffsearch OWNER staffsearch;"
```

---

## 4. Khởi động Redis

```bash
brew services start redis
```

---

## 5. Chạy Database Migrations

```bash
# Up — tạo tất cả tables
migrate -path migrations -database "postgres://staffsearch:staffsearch@localhost:5432/staffsearch?sslmode=disable" up

# Kiểm tra tables đã tạo
psql postgres://staffsearch:staffsearch@localhost:5432/staffsearch -c "\dt"
```

---

## 6. Seed Demo Data (tùy chọn)

```bash
psql postgres://staffsearch:staffsearch@localhost:5432/staffsearch -f seeds/demo_data.sql
```

---

## 7. Chạy Server

### Development (hot reload)

```bash
~/go/bin/fiber dev
```

Server sẽ tự restart khi có thay đổi code.

### Production build

```bash
~/go/bin/go1.26.1 build -o bin/api ./...
./bin/api
```

### Run trực tiếp (không build)

```bash
~/go/bin/go1.26.1 run main.go
```

---

## 8. Kiểm tra server đang chạy

```bash
curl http://localhost:3000/health
# → {"status":"ok"}
```

---

## Các lệnh hữu ích

```bash
# Build kiểm tra lỗi compile
~/go/bin/go1.26.1 build ./...

# Chạy tests
~/go/bin/go1.26.1 test ./...

# Tidy dependencies
~/go/bin/go1.26.1 mod tidy

# Rollback migrations (1 bước)
migrate -path migrations -database $DATABASE_URL down 1
```

---

## API Endpoints (Phase 1 MVP)

| Method | Path | Mô tả |
|--------|------|-------|
| POST | `/api/v1/auth/register` | Đăng ký tài khoản |
| POST | `/api/v1/auth/login` | Đăng nhập |
| POST | `/api/v1/auth/refresh` | Refresh access token |
| POST | `/api/v1/auth/logout` | Đăng xuất |
| POST | `/api/v1/auth/google` | Google Sign-In |
| POST | `/api/v1/auth/password-reset/request` | Yêu cầu reset password |
| POST | `/api/v1/auth/password-reset/confirm` | Xác nhận reset password |
| GET | `/api/v1/users/me` | Lấy thông tin user hiện tại |
| PATCH | `/api/v1/users/me` | Cập nhật profile |
| GET | `/api/v1/staff/job-categories` | Danh sách 21 job categories |
| POST | `/api/v1/staff/profile` | Tạo staff profile |
| PATCH | `/api/v1/staff/profile` | Cập nhật staff profile |
| GET | `/api/v1/staff/me` | Lấy staff profile của mình |
| GET | `/api/v1/staff/:userID` | Lấy staff profile công khai |
| POST | `/api/v1/staff/portfolio/photos` | Thêm ảnh portfolio |
| DELETE | `/api/v1/staff/portfolio/photos/:id` | Xóa ảnh portfolio |
| PATCH | `/api/v1/staff/portfolio/photos/reorder` | Sắp xếp lại ảnh |
| POST | `/api/v1/posts` | Tạo post |
| GET | `/api/v1/posts/feed` | Lấy feed (cursor pagination) |
| GET | `/api/v1/posts/:postID` | Lấy 1 post |
| POST | `/api/v1/media/upload-url` | Lấy presigned URL để upload |
| DELETE | `/api/v1/media` | Xóa file khỏi storage |
