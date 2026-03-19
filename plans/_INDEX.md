# Implementation Plans Index

> Mỗi file plan = 1 ticket. Đọc plan trước khi implement. Làm theo thứ tự dưới đây vì có dependency.

## Thứ tự triển khai (theo dependency)

| # | Ticket | Plan File | Depends On | Status |
|---|---|---|---|---|
| 1 | DB-01 — Database Schema | [DB-01-plan.md](./DB-01-plan.md) | AUTH-01 (users table đã có) | 🔲 |
| 2 | AUTH-01 — Email/Password Auth | [AUTH-01-plan.md](./AUTH-01-plan.md) | Backend done; cần Flutter migration | 🔲 |
| 3 | AUTH-02 — Google Sign-In | [AUTH-02-plan.md](./AUTH-02-plan.md) | AUTH-01 | 🔲 |
| 4 | DB-02 — Migrate LocalStorage to DB | [DB-02-plan.md](./DB-02-plan.md) | AUTH-01 | 🔲 |
| 5 | DB-05 — Storage Setup (Media) | [DB-05-plan.md](./DB-05-plan.md) | AUTH-01 | 🔲 |
| 6 | STAFF-02 — Staff Number Generation | [STAFF-02-plan.md](./STAFF-02-plan.md) | DB-01 | 🔲 |
| 7 | STAFF-03 — Job Type Selection | [STAFF-03-plan.md](./STAFF-03-plan.md) | DB-01 | 🔲 |
| 8 | STAFF-01 — Staff Profile CRUD | [STAFF-01-plan.md](./STAFF-01-plan.md) | STAFF-02, STAFF-03, DB-01 | 🔲 |
| 9 | STAFF-05 — Portfolio Photo Upload | [STAFF-05-plan.md](./STAFF-05-plan.md) | STAFF-01, DB-05 | 🔲 |
| 10 | STAFF-07 — Staff Profile Screen Connected | [STAFF-07-plan.md](./STAFF-07-plan.md) | STAFF-01, STAFF-05 | 🔲 |
| 11 | FEED-01 — Create Post | [FEED-01-plan.md](./FEED-01-plan.md) | DB-05, AUTH-01 | 🔲 |
| 12 | FEED-02 — TikTok Feed | [FEED-02-plan.md](./FEED-02-plan.md) | FEED-01, STAFF-03 | 🔲 |

## Ghi chú

- **Backend đã có**: AUTH endpoints (login, register, refresh, logout, me, privacy-policy), User model, migrations cho users + refresh_tokens
- **Flutter hiện tại**: Dùng `LocalAuthService` (SharedPreferences), mock data, chưa có `AuthProvider` hay `ApiClient`
- Go binary: dùng `~/go/bin/go1.26.1`, Fiber v3
- Flutter: `flutter run`, Provider state management
