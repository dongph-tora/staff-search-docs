# Change Request Report

**Date:** 2026-03-19
**New spec files:**
- `staff-search-docs/spec/markdown/main_app_specification_ja.md` (v1.3.0)
- `staff-search-docs/spec/markdown/admin_app_specification_ja.md` (v1.2.0)
- `staff-search-docs/spec/markdown/headhunting_app_specification_ja.md` (v1.1.0)

**Compared against:** `staff-search-docs/spec/markdown/staff_search_specification.md` (v2.0)

---

## 1. Issues — Current Spec vs New Spec

### A. Main App Changes (main_app_specification_ja.md)

| # | Ticket | Feature | Current Spec | New Spec | Change Type |
|---|---|---|---|---|---|
| 1 | — | Fan Club System | Not exists | Monthly subscription (500/1,000/2,000/5,000 yen), exclusive content, dedicated badge, priority DM | **New feature** |
| 2 | — | Age Verification System | Not exists | TikTok-style age restrictions: <13 cannot register, 13-15 view/like/comment only, 16-17 DM/download, 18+ live streaming | **New feature** |
| 3 | — | Patrol / Safety Features | Not exists | Banned word filter, stalker detection scoring (view=1pt, comment=3pt, DM=5pt, follow=2pt), 10-type reporting system | **New feature** |
| 4 | — | Stripe Connect Payout System | Not exists | Staff payout via Stripe Connect, min 1,000 yen, max 1,000,000 yen/request, 3 business days, bank account registration, 4 payout APIs | **New feature** |
| 5 | — | Staff Block/Mute Management | Block mgmt exists in user profile menu (Section 2.4) | Full block + mute management with tab UI for both user and staff sides, unblock/unmute actions | **New feature** |
| 6 | — | Terms & Guidelines Screen | Not exists | 3-tab screen: Community guidelines, Live streaming guidelines, Copyright policy | **New feature** |
| 7 | — | Multiple Store Management | Single store per company | Store list screen, add/edit/delete stores, active store switching, pull-to-refresh | **New feature** |
| 8 | — | Store Profile Editing | Basic store info only | Full edit form with industry dropdown, employee count, established date (DatePicker), benefits, commission slider (0-100%) | **New feature** |
| 9 | — | Staff Attendance (Clock In/Out) | Not exists | Toggle button on staff dashboard for attendance management | **New feature** |
| 10 | — | Staff QR Code | Not exists | QR code generation for staff profiles | **New feature** |
| 11 | — | Staff Revenue Dashboard | Basic "Revenue management" mentioned | Full dashboard with daily/monthly graphs, tip/booking/gift/fan club revenue breakdown, store commission calculation | **New feature** |
| 12 | BOOK-01 | Booking System | Single date/time, single service | Multiple menu selection, 3 preference dates (color-coded: 1st=red, 2nd=orange, 3rd=blue), coupon application, price calculation with discounts | **Changed feature** |
| 13 | PAY-* | Tip Revenue Distribution | 70% staff / 30% platform (independent), store share 0-10% | Stripe Connect model: Stripe fee 3.6% + Platform fee 10% + Store commission (0-100%). Example 1,000 yen: freelancer gets 868 yen, store-affiliated (30%) gets 608 yen | **Changed feature** |
| 14 | — | Store Staff Commission Rate | 0-10% range | 0-100% range (slider) | **Changed feature** |
| 15 | LIVE-03 | Live Streaming Requirements | 100+ followers for independent staff | 18+ age, 1,000+ followers, 100+ hours streaming in past 30 days | **Changed feature** |
| 16 | SEARCH-05 | Map Search | Basic "map search" listed | Map with store/company/staff pins, custom markers, info windows, current location button | **Changed feature** |
| 17 | — | Staff Data Model | Basic fields (id, name, avatar, jobTitle, etc.) | Added: category, profileImages (list), experience, latitude, longitude, isLive, qrCode, storeName, companyName, giftAmount, categoryRank, totalStaffInCategory | **Changed data model** |
| 18 | — | Company Data Model | Basic fields | Added: industry, contactEmail, contactPerson, employeeCount, establishedDate, benefits, tipCommissionRate (0-100%), latitude, longitude, logoUrl | **Changed data model** |
| 19 | — | Booking Data Model | Basic fields | Added: userName, staffName, staffImage, menus (list), bookingTime, totalPrice, discountAmount, finalPrice, couponCode, note (3 preferences) | **Changed data model** |
| 20 | — | User Data Model | No birthDate | Added birthDate for age verification | **Changed data model** |
| 21 | — | New Data Models | Not exists | PayoutRecord, BalanceInfo, ConnectAccountStatus, Menu, Coupon models | **New data models** |
| 22 | — | Tech Stack Additions | Flutter, Provider, SharedPreferences | Added: Hive (local DB), Flame (2D game engine), google_mobile_ads (ads SDK) | **Changed feature** |
| 23 | — | Coin System (Web vs App pricing) | Detailed coin pricing table (70-17,500 coins) | Not mentioned in new main spec (still in current spec) | **Potentially removed** |
| 24 | — | Shard System | Detailed shard earning/redemption | Not mentioned in new main spec (still referenced as "かけらコレクション") | **Unchanged** |
| 25 | — | Headhunting Plans | Free (3 offers) + Unlimited (3,000 yen/mo) | Referenced but detailed in headhunting app spec (4 tiers) | **Changed feature** |

### B. Admin App Changes (admin_app_specification_ja.md)

| # | Ticket | Feature | Current Spec | New Spec | Change Type |
|---|---|---|---|---|---|
| 26 | — | Separate Admin Web App | Admin is part of main app (admin/ screens) | Standalone Flutter web app, port 8181, package: com.stafffinder.admin_app | **New feature** |
| 27 | — | Admin Dashboard Statistics | Basic dashboard mentioned | 8 stat cards: total users, staff, stores, monthly/daily sales, active users, new registrations, pending support + graphs | **New feature** |
| 28 | — | Store/Company Management (Admin) | Not in admin features | Full CRUD: list, search, filter by industry/verification, detail view with staff/offers/revenue, verify/suspend/edit/delete | **New feature** |
| 29 | — | Headhunting Management (Admin) | Not in admin features | Company management, offer management, statistics (monthly offers, acceptance rate, per-company/staff stats) | **New feature** |
| 30 | — | Push Notification Management | Basic "Send push notifications" | Full system: target selection (all/role/individual), title/body/image/link, scheduled delivery, delivery history, re-send | **Changed feature** |
| 31 | — | Announcement Management | Not exists | Create/edit/delete announcements with categories (important/maintenance/feature/event), target audience, publish scheduling, publish/unpublish toggle | **New feature** |
| 32 | — | Report/Analytics Dashboard | Not in admin features | 4-tab reports: User analysis, Sales analysis (booking/tip/gift/fan club/commission), Staff analysis, Live streaming analysis | **New feature** |
| 33 | — | Live Revenue Management | Not exists | Revenue stats (daily/monthly, staff-based, gift-type, platform commission), Fan club management (member stats, revenue, renewal/churn), Top fans ranking | **New feature** |
| 34 | — | SNS Management (Admin) | Basic "SNS integration management" | Full system: post management (CRUD, scheduling), platform integration (Twitter/Instagram/Facebook/LINE), engagement analytics | **Changed feature** |
| 35 | — | Content Moderation Enhancements | Delete posts/comments/reviews | Added: live stream moderation (force end, broadcaster warning/suspension), comment moderation, 3-strike warning system | **Changed feature** |
| 36 | — | Support Chat Enhancements | Basic "Operations support chat" | Full system: ticket management with status (open/in_progress/closed), priority, category, file attachments, templates, quick replies, FAQ | **Changed feature** |
| 37 | — | Admin Roles | Single "Admin" role | 2 roles: admin (full access), moderator (content moderation only) | **Changed feature** |
| 38 | — | Admin Data Models | Not detailed | New models: AdminUser, Notification, Announcement, SupportTicket, ChatMessage | **New data models** |

### C. Headhunting App Changes (headhunting_app_specification_ja.md)

| # | Ticket | Feature | Current Spec | New Spec | Change Type |
|---|---|---|---|---|---|
| 39 | — | Separate Headhunting Web App | Headhunting is part of main app | Standalone Flutter web app, port 9090, package: com.stafffinder.headhunting_app | **New feature** |
| 40 | — | Headhunting Dashboard | Not exists | 7 stat cards (sent/pending/accepted/rejected offers, messages, unread, notifications), recent activity, quick actions | **New feature** |
| 41 | — | Enhanced Staff Search (HH) | Basic search in company mgmt | 8 categories, area search, detailed filters (gender/age/rating/experience/online), paginated grid with staff cards | **Changed feature** |
| 42 | — | Enhanced Offer Management (HH) | Basic offer list | Tabbed view (all/pending/accepted/rejected), search/filter/sort, detailed dialog with response message, delete, message staff | **Changed feature** |
| 43 | — | Company-Staff Messaging | Not in headhunting | Direct messaging between company and staff, thread-based, real-time, read receipts, message deletion | **New feature** |
| 44 | — | Headhunting Plans (4-tier) | Free (3 offers) + Unlimited (3,000 yen/mo) | 4 tiers: Free (0 yen, 3/mo), Basic (9,800 yen, 20/mo), Pro (29,800 yen, 100/mo), Enterprise (custom, unlimited) | **Changed feature** |
| 45 | — | Announcement Reception (HH) | Not exists | Receive announcements from admin, category filter (important/maintenance/feature/event), read/unread management | **New feature** |
| 46 | — | Notification System (HH) | Not exists | Offer response/message/system notifications, type filter, read/unread filter, notification settings (push/email ON/OFF) | **New feature** |
| 47 | — | HeadhuntingOffer Model Changes | salaryRange (string), basic fields | Changed to: salary (int), added jobDescription, benefits, responseMessage, respondedAt | **Changed data model** |
| 48 | — | New HH Data Models | Not exists | Plan, Message, Announcement, Notification models for headhunting app | **New data models** |

---

## 2. Impact Scope

| # | Issue | Frontend | Backend | Database | UI/UX | Other Tickets Affected |
|---|---|---|---|---|---|---|
| 1 | Fan Club | New screens (fan club join, plan select, exclusive content), badge widget | New endpoints (create/join/cancel fan club, list members, exclusive content) | New tables: fan_clubs, fan_club_memberships, fan_club_content | Fan club join button on staff detail, badge on profile | PAY-*, STAFF-* |
| 2 | Age Verification | Registration flow, feature restriction checks | Age verification in registration, feature access middleware | users table (add birth_date) | Registration form, restricted UI for minors | AUTH-01, AUTH-04 |
| 3 | Patrol / Safety | Banned word filter on input, stalker detection service, report dialog (10 reasons) | Moderation service, word filter API, stalker scoring, report handling | New tables: banned_words, interaction_scores, reports | Report button on profiles/posts/messages | CHAT-*, FEED-*, LIVE-* |
| 4 | Stripe Connect Payout | Payout screen (3 tabs: request/history/rules), bank account setup | 4 new Stripe Connect APIs, webhook handler | New tables: payout_records, connect_accounts, balance_info | Staff dashboard new menu items | PAY-09, PAY-12 |
| 5 | Block/Mute Mgmt | Block management screen with tabs (block/mute), unblock/unmute | Block/mute APIs, query filters for blocked users | blocks table (add mute flag) | Staff dashboard + profile menu additions | CHAT-*, FEED-* |
| 6 | Terms & Guidelines | New 3-tab screen | Static content or CMS endpoints | — | Profile menu addition | — |
| 7 | Multiple Store Mgmt | Store list screen, store edit screen | Store CRUD endpoints, active store switching | companies table (add active flag per user) | Company management flow rewrite | STAFF-*, PAY-* |
| 8 | Store Profile Edit | Full edit form with date picker, industry dropdown, slider | Store update endpoint with new fields | companies table (new fields) | Store management UI overhaul | — |
| 9 | Staff Attendance | Toggle button on dashboard | Attendance endpoints (clock in/out, status) | New table: attendance_records | Staff dashboard UI | STAFF-01 |
| 10 | Staff QR Code | QR code display widget | QR code generation endpoint | staff_profiles (add qr_code field) | Staff profile screen | STAFF-01, STAFF-07 |
| 11 | Revenue Dashboard | New dashboard screen with graphs | Revenue aggregation endpoints | Views/materialized views for revenue | Staff dashboard new screen | PAY-08, PAY-09 |
| 12 | Booking (3 prefs) | Rewrite booking screen (multi-menu, 3 dates, coupons) | Booking API changes (menus array, 3 prefs in note, coupon validation) | bookings table (new columns), menus table, coupons table | Major booking UX change | BOOK-01..05 |
| 13 | Payment Distribution | Payment service rewrite | Stripe Connect integration, fee calculation | tip_transactions (new fee columns) | Staff earnings display | PAY-01..12 |
| 14 | Commission Rate 0-100% | Slider max change | Validation change | companies.tip_commission_rate max 100 | Minor UI change | — |
| 15 | Live Requirements | Permission check update | Live permission endpoint update | — | UI messaging for requirements | LIVE-03 |
| 16 | Map Search Enhanced | Map with multiple pin types | Store/company location endpoints | lat/lng on companies | Map screen enhancements | SEARCH-02, SEARCH-05 |
| 17-21 | Data Model Changes | Model class updates | API response/request updates | Schema migrations | — | DB-01, STAFF-01, BOOK-01 |
| 22 | Tech Stack | pubspec.yaml additions | — | Hive local DB | — | — |
| 25 | HH Plans (4-tier) | Plan management screen | Plan endpoints, subscription billing | New table: headhunting_plans, subscriptions | Pricing page | PAY-* |
| 26 | Separate Admin App | New Flutter web app project | Admin-specific API endpoints | — | Standalone web panel | All admin tickets |
| 27-38 | Admin Features | Admin screens (dashboard, store mgmt, HH mgmt, notifications, announcements, reports, live revenue, SNS, enhanced moderation, support) | 15+ new admin API endpoints | New tables: notifications, announcements, admin_users | Full admin panel | NOTIF-*, RANK-* |
| 39 | Separate HH App | New Flutter web app project | HH-specific API endpoints | — | Standalone web panel | — |
| 40-48 | HH App Features | HH screens (dashboard, search, offers, messaging, plans, announcements, notifications) | 10+ new HH API endpoints | Plans table, messages table | Full HH panel | CHAT-*, PAY-* |

---

## 3. Tasks and Estimated Effort

### A. Main App New Features

| # | Issue | Task | Files to Change | Estimate (days) |
|---|---|---|---|---|
| 1.1 | Fan Club | Write spec SOCIAL-04 (Fan Club) | SOCIAL-04-fan-club.md | 0.5 |
| 1.2 | Fan Club | Backend: fan club CRUD, membership, content endpoints | handler, service, repository, migration | 2 |
| 1.3 | Fan Club | Frontend: join screen, plan selection, exclusive content, badge | 4-5 new screens, widgets | 2 |
| 1.4 | Fan Club | Stripe subscription integration for monthly plans | payment service | 1 |
| 2.1 | Age Verification | Write spec AUTH-11 (Age Verification) | AUTH-11-age-verification.md | 0.5 |
| 2.2 | Age Verification | Backend: birth_date on registration, age-based middleware | auth handler, middleware | 1 |
| 2.3 | Age Verification | Frontend: DOB picker on registration, feature restrictions | registration screen, age service | 1 |
| 3.1 | Patrol / Safety | Write spec SAFETY-01 (Patrol System) | SAFETY-01-patrol-system.md | 0.5 |
| 3.2 | Patrol / Safety | Backend: word filter, stalker scoring, report handling | 3 new services, migration | 2 |
| 3.3 | Patrol / Safety | Frontend: report dialog, word filter on inputs, warning UI | patrol service, report screen | 1.5 |
| 4.1 | Stripe Connect Payout | Write spec PAY-13 (Stripe Connect Payout) | PAY-13-stripe-connect-payout.md | 0.5 |
| 4.2 | Stripe Connect Payout | Backend: 4 Stripe Connect APIs, webhook, account onboarding | 4 endpoints, stripe service | 2 |
| 4.3 | Stripe Connect Payout | Frontend: payout screen (3 tabs), bank account setup | 2 new screens | 1.5 |
| 5.1 | Block/Mute | Write spec SOCIAL-05 (Block/Mute Management) | SOCIAL-05-block-mute.md | 0.25 |
| 5.2 | Block/Mute | Backend + Frontend implementation | handler, screen, service | 1.5 |
| 6.1 | Terms & Guidelines | Write spec + implement (static content) | New screen, profile menu | 0.5 |
| 7.1 | Multiple Store Mgmt | Write spec STORE-01 (Multi-Store) | STORE-01-multi-store.md | 0.5 |
| 7.2 | Multiple Store Mgmt | Backend + Frontend implementation | store endpoints, 2 new screens | 2 |
| 8.1 | Store Profile Edit | Update existing spec + implement | store edit screen, API update | 1 |
| 9.1 | Staff Attendance | Write spec STAFF-09 (Attendance) | STAFF-09-attendance.md | 0.25 |
| 9.2 | Staff Attendance | Backend + Frontend implementation | attendance endpoints, dashboard toggle | 1 |
| 10.1 | Staff QR Code | Write spec + implement | QR generation, profile widget | 0.5 |
| 11.1 | Revenue Dashboard | Write spec STAFF-10 (Revenue Dashboard) | STAFF-10-revenue-dashboard.md | 0.5 |
| 11.2 | Revenue Dashboard | Backend + Frontend implementation | aggregation endpoints, dashboard screen with charts | 2 |

### B. Main App Changed Features

| # | Issue | Task | Files to Change | Estimate (days) |
|---|---|---|---|---|
| 12.1 | Booking (3 prefs) | Update BOOK-01 spec | BOOK-01-create-booking.md | 0.5 |
| 12.2 | Booking (3 prefs) | Backend: menu selection, 3 dates, coupon APIs | booking handler, menu service, coupon service | 2 |
| 12.3 | Booking (3 prefs) | Frontend: rewrite booking screen | create_booking_screen.dart | 1.5 |
| 13.1 | Payment Distribution | Update PAY-* specs | Multiple PAY specs | 0.5 |
| 13.2 | Payment Distribution | Backend: Stripe Connect fee model | payment service, tip handler | 1.5 |
| 14.1 | Commission Rate | Update validation (0-100%) | Company model, API validation | 0.25 |
| 15.1 | Live Requirements | Update LIVE-03 spec | LIVE-03 spec, permission check | 0.5 |
| 17.1 | Data Model Changes | Update DB-01 schema spec | DB-01-database-schema.md | 0.5 |
| 17.2 | Data Model Changes | Run migrations for all model changes | 5+ migration files | 1 |
| 22.1 | Tech Stack | Add Hive, Flame, google_mobile_ads to pubspec | pubspec.yaml | 0.25 |

### C. Admin App (New Standalone App)

| # | Issue | Task | Files to Change | Estimate (days) |
|---|---|---|---|---|
| 26.1 | Admin App Setup | Create new Flutter web project + routing | New project scaffold | 1 |
| 27.1 | Admin Dashboard | Dashboard screen with 8 stats + graphs | dashboard screen, admin APIs | 1.5 |
| 28.1 | Store/Company Mgmt (Admin) | Write spec ADMIN-01 | ADMIN-01-store-company-mgmt.md | 0.5 |
| 28.2 | Store/Company Mgmt (Admin) | Backend + Frontend | CRUD screens, admin endpoints | 2 |
| 29.1 | HH Management (Admin) | Write spec ADMIN-02 | ADMIN-02-headhunting-mgmt.md | 0.5 |
| 29.2 | HH Management (Admin) | Backend + Frontend | Company/offer mgmt screens | 1.5 |
| 30.1 | Push Notifications | Write spec ADMIN-03 | ADMIN-03-push-notifications.md | 0.5 |
| 30.2 | Push Notifications | Backend: FCM integration, scheduling, history | notification service, FCM handler | 2 |
| 30.3 | Push Notifications | Frontend: notification send screen, history | 1 new screen | 1 |
| 31.1 | Announcement Mgmt | Write spec ADMIN-04 | ADMIN-04-announcements.md | 0.5 |
| 31.2 | Announcement Mgmt | Backend + Frontend | CRUD endpoints, screen | 1.5 |
| 32.1 | Reports/Analytics | Write spec ADMIN-05 | ADMIN-05-reports.md | 0.5 |
| 32.2 | Reports/Analytics | Backend: aggregation queries | analytics endpoints | 2 |
| 32.3 | Reports/Analytics | Frontend: 4-tab report screen with graphs | report screen, chart widgets | 2 |
| 33.1 | Live Revenue Mgmt | Write spec ADMIN-06 | ADMIN-06-live-revenue.md | 0.5 |
| 33.2 | Live Revenue Mgmt | Backend + Frontend | revenue endpoints, screen | 1.5 |
| 34.1 | SNS Management | Enhanced SNS management screen | SNS service, screen | 1.5 |
| 35.1 | Content Moderation | Enhanced moderation + 3-strike system | moderation service, warning system | 1.5 |
| 36.1 | Support Chat | Enhanced support chat with tickets, templates | chat service, ticket system | 2 |
| 37.1 | Admin Roles | Admin + moderator role system | auth middleware, role check | 0.5 |

### D. Headhunting App (New Standalone App)

| # | Issue | Task | Files to Change | Estimate (days) |
|---|---|---|---|---|
| 39.1 | HH App Setup | Create new Flutter web project + routing | New project scaffold | 1 |
| 40.1 | HH Dashboard | Dashboard screen with 7 stats | dashboard screen | 1 |
| 41.1 | Enhanced Staff Search | Search screen with 8 categories, filters, pagination | search screen | 1.5 |
| 42.1 | Enhanced Offer Mgmt | Tabbed offer management screen | offer management screen | 1.5 |
| 43.1 | Company-Staff Messaging | Write spec HH-01 | HH-01-company-messaging.md | 0.5 |
| 43.2 | Company-Staff Messaging | Backend + Frontend | messaging endpoints, chat screen | 2 |
| 44.1 | 4-Tier Plans | Plan management screen + billing | plan screen, Stripe subscription | 1.5 |
| 45.1 | Announcement Reception | Announcement list/detail screen | announcement screen | 0.5 |
| 46.1 | Notification System | Notification list/detail + settings | notification screen, settings | 1 |

---

## Summary

| Category | Count |
|---|---|
| **Total issues** | **48** |
| **New features** | **25** |
| **Changed features** | **14** |
| **Changed data models** | **7** |
| **New data models** | **2** |
| **Potentially removed** | **0** |

### Estimated Effort by Area

| Area | Specs (days) | Implementation (days) | Total (days) |
|---|---|---|---|
| Main App — New Features | 4.0 | 22.0 | **26.0** |
| Main App — Changed Features | 1.5 | 6.5 | **8.0** |
| Admin App (New) | 3.0 | 19.0 | **22.0** |
| Headhunting App (New) | 0.5 | 9.0 | **9.5** |
| **Grand Total** | **9.0** | **56.5** | **65.5 days** |

### Top Priority Changes (Impact Order)

1. **Stripe Connect Payout** — Critical for monetization
2. **Fan Club System** — New revenue stream
3. **Booking System Rewrite** — Core UX improvement (3 preferences, menus, coupons)
4. **Age Verification** — Legal/compliance requirement
5. **Admin App** — Operational necessity
6. **Headhunting Plans (4-tier)** — Revenue model change
7. **Patrol / Safety Features** — User safety compliance
8. **Separate Headhunting App** — Business operation tool
