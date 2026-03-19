# Build Guide — Flutter App (staff-search-app)

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Flutter | 3.35.4 | `flutter upgrade` hoặc FVM |
| Dart | 3.9.2 | đi kèm Flutter |
| Xcode | 15+ | App Store (iOS/macOS) |
| Android Studio | Hedgehog+ | android.com/studio |
| CocoaPods | latest | `sudo gem install cocoapods` |

Kiểm tra:
```bash
flutter doctor
```

---

## 1. Clone & Install Dependencies

```bash
cd staff-vibe-project/staff-search-app
flutter pub get
```

---

## 2. Cấu hình API URL

Mở `lib/config/api_config.dart` và đặt URL backend:

```dart
// Local development
const String kApiBaseUrl = 'http://localhost:3000';

// Nếu test trên Android emulator (trỏ về máy host)
const String kApiBaseUrl = 'http://10.0.2.2:3000';

// Nếu test trên thiết bị thật (cùng WiFi)
const String kApiBaseUrl = 'http://192.168.x.x:3000';  // IP máy dev
```

---

## 3. Chạy App

### iOS Simulator

```bash
open -a Simulator
flutter run
```

### Android Emulator

```bash
flutter emulators --launch <emulator_id>
flutter run
```

### Thiết bị thật

```bash
flutter run -d <device_id>
# Xem danh sách thiết bị:
flutter devices
```

### Web (local)

```bash
flutter run -d chrome
```

---

## 4. Build Production

### Android APK

```bash
flutter build apk --release
# Output: build/app/outputs/flutter-apk/app-release.apk
```

### Android App Bundle (cho Google Play)

```bash
flutter build appbundle --release
# Output: build/app/outputs/bundle/release/app-release.aab
```

### iOS (cần Mac + Apple Developer account)

```bash
flutter build ios --release
# Sau đó mở Xcode để archive và distribute
```

### Web

```bash
flutter build web --release
# Serve locally:
cd build/web && python3 -m http.server 5060 --bind 0.0.0.0
# → http://localhost:5060
```

---

## 5. Cấu hình Google Sign-In (tùy chọn)

Nếu cần Google Sign-In hoạt động:

**Android:** Đặt `google-services.json` vào `android/app/`

**iOS:** Đặt `GoogleService-Info.plist` vào `ios/Runner/`

Cập nhật `GOOGLE_CLIENT_ID` trong backend `.env`.

> Nếu bỏ trống, nút Google Sign-In vẫn hiển thị nhưng sẽ báo lỗi khi tap.

---

## Màn hình có trong Phase 1 MVP

### Auth
| Màn hình | Route | Mô tả |
|----------|-------|-------|
| Login | `/login` | Email/password + Google Sign-In |
| Register | `/register` | Đăng ký tài khoản mới |
| Password Reset | `/password-reset-confirm` | Xác nhận reset password |

### User
| Màn hình | Route | Mô tả |
|----------|-------|-------|
| Home Feed | `/home` | TikTok-style vertical feed |
| Profile Edit | `/profile-edit` | Chỉnh sửa tên, bio, số điện thoại |
| Create Post | `/create-post` | Đăng ảnh/video với caption |

### Staff
| Màn hình | Route | Mô tả |
|----------|-------|-------|
| Staff Registration | `/staff-registration` | Đăng ký profile staff |
| Staff Profile | `/staff-profile` | Xem profile (owner/visitor mode) |
| Portfolio Edit | `/portfolio-edit` | Quản lý ảnh portfolio (tối đa 12) |

---

## Lưu ý khi demo

- Backend phải đang chạy trước khi mở app
- Đảm bảo `kApiBaseUrl` trỏ đúng địa chỉ backend
- Lần đầu mở app, cần đăng ký tài khoản (không có tài khoản demo mặc định ở phase này)
- Feed sẽ trống cho đến khi có staff đăng bài qua Create Post
