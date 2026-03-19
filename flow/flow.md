  AUTH-01 — Login / Password

  [App Launch]
      ├─→ LoginScreen (AUTH-01)                    ← unauthenticated entry
      │       ├─→ [login success] → HomeScreen
      │       └─→ "パスワードをお忘れ" → PasswordResetConfirmScreen (AUTH-07)
      │
      └─→ HomeScreen → Tab 4 (Profile)
              → ProfileSettingsScreen
                      → PasswordChangeScreen

  ---
  Profile screens

  HomeScreen → Tab 4 → ProfileScreen
      ├─→ プロフィール設定 → ProfileSettingsScreen
      │       └─→ パスワード変更 → PasswordChangeScreen
      └─→ Edit button → ProfileEditScreen   ⚠️  route không tìm thấy trong codebase

  ---
  FEED-02 — Home Feed

  [Login success] → HomeScreen (FEED-02)   ← auto-redirect
  FeedScreen (FEED-02)                     ⚠️  không tìm thấy navigator vào màn này

  ---
  FEED-01 — Create Post

  HomeScreen → FAB "/create-post" → CreatePostScreen feed/ (FEED-01)

  StaffDashboard → Tab 1 → StaffPostsManagementScreen (FEED-01)
  StaffDashboard → (create button?) → staff/CreatePostScreen (FEED-01)   ⚠️  chưa rõ entry

  ---
  STAFF-07 — Staff Profile

  HomeScreen → tap staff card (StaffCard widget) → StaffDetailScreen Visitor (STAFF-07)
  SearchScreen → tap result → StaffDetailScreen Visitor
  RankingScreen → tap card → StaffDetailScreen Visitor
  MapSearchScreen → tap pin → StaffDetailScreen Visitor
  StaffDashboard → "スタッフプレビュー確認" → StaffDetailScreen Visitor

  ProfileScreen → "スタッフモード" button
      ├─ (staff registered)   → StaffDashboardScreen (STAFF-01)
      └─ (not registered yet) → StaffRegistrationScreen

  Navigator.pushNamed(context, '/staff-profile', args: userID) → StaffProfileScreen Owner (STAFF-07)
      ├─→ "Edit Profile" → pushNamed('/staff-registration')   ← trỏ sai route?
      └─→ "Edit Portfolio" → PortfolioEditScreen (STAFF-05)

  ---
  STAFF-01 — Staff Dashboard & Profile CRUD

  ProfileScreen → スタッフモード (staff registered)
      → StaffDashboardScreen (STAFF-01)
          ├─ Tab 1 → StaffPostsManagementScreen (FEED-01)
          ├─ Tab 2 → StaffBookingsScreen
          │       └─→ button → StaffServiceManagementScreen (STAFF-01)
          ├─ Tab 3 → StaffTipsScreen
          └─ Tab 4 → StaffProfileEditScreen

  ---
  STAFF-02 — Auto-generate unique 6-digit Staff ID  [backend only, no screen]

  Triggered internally when StaffService.CreateProfile is called (STAFF-01).
  crypto/rand → zero-pad 6 digits → uniqueness check → retry up to 10x
  Result stored in staff_profiles.staff_number, displayed on StaffProfileScreen (STAFF-07).

  ---
  STAFF-03 — Job Category selection  [no dedicated screen — widget only]

  Widget: JobCategoryDropdown  → embedded in StaffRegistrationScreen
                                → embedded in StaffProfileEditScreen
  Widget: JobCategoryChips     → embedded in HomeScreen (below AppBar)
                                → embedded in SearchScreen (filter row)

  Data source: GET /api/v1/staff/job-categories  (21 categories, cached per session) (STAFF-01)

  ---
  Các vấn đề cần chú ý

  ┌───────────────────────────────────────────────────────────────────────────────────────┬──────────┐
  │                                        Vấn đề                                         │ Màn hình │
  ├───────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ ProfileEditScreen không có route nào navigate vào                                     │ Profile  │
  ├───────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ FeedScreen (feed/feed_screen.dart) không được navigate tới                            │ FEED-02  │
  ├───────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ StaffProfileScreen Owner → "Edit Profile" gọi /staff-registration thay vì edit screen │ STAFF-07 │
  ├───────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ staff/create_post_screen.dart — entry point chưa rõ                                   │ FEED-01  │
  └───────────────────────────────────────────────────────────────────────────────────────┴──────────┘
