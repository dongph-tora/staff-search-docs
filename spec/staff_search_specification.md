# Staff Finder Application Specification

**Version:** 2.0  
**Last Updated:** 2025  
**Application Name:** Staff Finder  
**Package Name:** `com.stafffinder.finder`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Key Features](#2-key-features)
3. [User Roles](#3-user-roles)
4. [Coin & Point System](#4-coin--point-system)
5. [Live Streaming System](#5-live-streaming-system)
6. [Live League System](#6-live-league-system)
7. [Live Collab & Battle System](#7-live-collab--battle-system)
8. [Tip (Gift) System](#8-tip-gift-system)
9. [Headhunting System](#9-headhunting-system)
10. [Store Management System](#10-store-management-system)
11. [Booking System](#11-booking-system)
12. [Messaging](#12-messaging)
13. [Review System](#13-review-system)
14. [Admin Features](#14-admin-features)
15. [Tech Stack](#15-tech-stack)
16. [Data Models](#16-data-models)
17. [Screen List](#17-screen-list)
18. [Service List](#18-service-list)
19. [Future Roadmap](#19-future-roadmap)
20. [Summary](#20-summary)

---

## 1. Overview

### 1.1 Purpose

Staff Finder is a matching platform connecting staff in the food & beverage and beauty service industries with customers. It is a mobile application integrating diverse features including TikTok-style live streaming, tipping, headhunting, and booking management.

### 1.2 Key Highlights

- **TikTok-style Live Streaming:** Solo, collab (up to 4 people), and battle broadcasts
- **Tipping System:** Coin purchases and gift sending with real-time monetization
- **Live League System:** 8-tier ranking (Bronze to Legend)
- **Shard System:** Tip rewards and collectible items
- **Headhunting:** Corporate staff scouting feature
- **Store Management:** Store owner staff management and offer sending
- **Booking System:** Staff booking and reservation feature
- **SNS Features:** Follow, posts, stories, likes, and comments

### 1.3 Target Platforms

- **Android:** API Level 35 (Android 15)
- **Web:** Browser preview supported
- **Flutter SDK:** 3.35.4
- **Dart SDK:** 3.9.2

---

## 2. Key Features

### 2.1 Authentication

- **User Registration:** Email, password, username, profile picture
- **Login:** Email/password authentication
- **Staff Registration:** Switch from regular user to staff account
- **Profile Editing:** Basic info, bio, job title, work location, SNS links
- **Demo Accounts:**
  - Staff: `staff-demo@example.com` / `demo123`
  - Regular User: `demo@example.com` / `demo123`

### 2.2 Staff Search & Discovery

- **Home Screen:** Recommended staff list, category filters
- **Search:** Search by name, job title, location, or tags
- **Map Search:** Location-based nearby staff search
- **Filter Settings:** Age, gender, job title, area, follower count
- **Rankings:** Popular staff, tip rankings, live league rankings

### 2.3 SNS Features

- **Follow/Followers:** Follow staff, view follower lists
- **Posts:** Image/video posts, likes, comments
- **Stories:** 24-hour limited posts with viewer display
- **Likes & Favorites:** Add staff to favorites
- **Notifications:** Follow, like, comment, tip, and message notifications

### 2.4 User Profile

The user profile screen includes the following menu items:

1. **Headhunting:** Company management, staff scouting
2. **Company Management:** Register/edit company info, manage offers
3. **Store Management:** Register stores, manage staff, send store offers
4. **Booking History:** User booking list, booking details
5. **Wallet (Coin Balance):** Display current coin balance, navigate to purchase screen
6. **Earn Coins (Free):** Earn coins via check-in and ad viewing
7. **Tip History:** List of sent tips
8. **Reviews:** List of written reviews
9. **Settings:** Profile editing, notification settings, privacy settings, password change
10. **Rankings:** Staff ranking, tip ranking, league ranking
11. **Help & Support:** FAQ, inquiries, support chat
12. **Block Management:** List of blocked users

---

## 3. User Roles

### 3.1 Regular User

- Search and follow staff
- View posts, like, and comment
- Watch live streams
- Send tips
- Book staff
- Send messages
- Post reviews

### 3.2 Staff

In addition to regular user permissions:

- Detailed profile editing (job title, work location, SNS links, etc.)
- Create posts and stories
- Live streaming (solo, collab, battle)
- Receive tips
- Manage bookings
- Create and manage coupons
- Manage service menu
- Staff dashboard
- Revenue management (tips, battle rewards)
- Manage received offers (headhunting and store offers)

### 3.3 Company User

In addition to regular user permissions:

- Register and manage company information
- Send headhunting offers to staff
- Manage offers (sent, accepted, rejected)
- View list of affiliated staff

### 3.4 Store Owner

In addition to company user permissions:

- Register and manage store information
- Manage store staff
- Send store offers to staff
- Set tip share rate (0–10%)
- Manage live streaming permissions for affiliated staff

### 3.5 Admin

- User management (regular, staff, company, admin)
- Content moderation (delete posts, comments)
- Report management
- Send push notifications
- SNS integration management
- Operations support chat

---

## 4. Coin & Point System

### 4.1 TikTok-style Coin System

Coins are used as the in-app currency. Prices differ between the web and app versions.

#### 4.1.1 Coin Price Table

| Coins | Web Price | App Price | Difference | Popular |
|-------|-----------|-----------|------------|---------|
| 70 coins | ¥129 | ¥140 | +¥11 (+8.5%) | |
| 350 coins | ¥649 | ¥700 | +¥51 (+7.9%) | |
| 700 coins | ¥1,299 | ¥1,400 | +¥101 (+7.8%) | 🔥 Popular |
| 1,400 coins | ¥2,599 | ¥2,800 | +¥201 (+7.7%) | |
| 3,500 coins | ¥6,499 | ¥7,000 | +¥501 (+7.7%) | |
| 7,000 coins | ¥12,999 | ¥14,000 | +¥1,001 (+7.7%) | |
| 17,500 coins | ¥32,499 | ¥35,000 | +¥2,501 (+7.7%) | |

**Pricing rationale:**
- **Web version (cheaper):** Avoids app store fees (~30%), savings passed to users
- **App version (pricier):** Reflects Apple/Google platform fees

#### 4.1.2 Cost Per Coin

- Web version 700 coins: ~¥1.86/coin (best value)
- App version 700 coins: ~¥2.00/coin

#### 4.1.3 Coin Purchase Flow

1. Tap "Wallet (Coin Balance)" on the profile screen
2. View current coin balance and 7 packages
3. Select a package
4. Confirm price, coin count, and cost-per-coin in dialog
5. Execute Stripe payment (demo mode)
6. Coins added to balance
7. Display completion message

#### 4.1.4 Coin Expiry

- **No expiry:** Purchased coins never expire

### 4.2 Free Coin Earning

#### 4.2.1 Check-in Bonus

Earn coins daily with consecutive logins:

| Day | Coins Earned |
|-----|-------------|
| Day 1 | 5 coins |
| Day 2 | 10 coins |
| Day 3 | 15 coins |
| Day 4 | 20 coins |
| Day 5 | 30 coins |
| Day 6 | 40 coins |
| Day 7 | 50 coins |

- Streak resets if login is missed
- 7-day streak earns a total of 170 coins

#### 4.2.2 Ad Viewing

- Earn 50 coins per 15-second video ad
- Maximum 10 times per day (cap: 500 coins/day)
- Coins awarded immediately after ad completion

#### 4.2.3 Free Earn Screen

Profile → Tap "Earn Coins (Free)":

- Check-in button (today's reward, streak days display)
- Ad viewing button (today's count, remaining views display)
- Earning history list

### 4.3 Coin Balance Management

```dart
class UserPointBalance {
  final String userId;
  final int totalPoints;      // Total coins earned
  final int purchasedPoints;  // Purchased coins
  final int bonusPoints;      // Bonus coins (currently unused)
  final int usedPoints;       // Spent coins
  final DateTime lastUpdated;
  int get availablePoints => totalPoints - usedPoints; // Available coins
}
```

### 4.4 Headhunting Plans (Subscription)

Monthly plans for company users:

#### 4.4.1 Free Plan

- **Price:** ¥0/month
- **Benefits:**
  - Up to 3 headhunting uses
  - Basic offer sending

#### 4.4.2 Unlimited Plan (Popular)

- **Price:** ¥3,000/month
- **Benefits:**
  - Unlimited headhunting
  - Priority support
  - Applicant analytics
  - Bulk staff management

---

## 5. Live Streaming System

### 5.1 Overview

TikTok-style live streaming. Staff can interact with fans in real time and monetize through tips.

### 5.2 Stream Types

#### 5.2.1 Solo Stream

- Single-person live broadcast
- Start instantly via "Start Live Stream" button on staff dashboard
- Session created automatically, transitions to broadcast screen

#### 5.2.2 Collab Stream

- **Duet:** 2-person joint stream
- **Trio:** 3-person joint stream
- **Squad:** 4-person joint stream

#### 5.2.3 Battle Stream

- **1v1 Battle:** 2-person competition
- **2v2 Battle:** Team competition
- **4-person Battle Royale:** All-vs-all competition

### 5.3 Live Streaming Permissions

#### 5.3.1 Basic Requirements

- Staff account required
- 100+ followers (independent staff)

#### 5.3.2 Exception for Store-Affiliated Staff

- No follower count requirement for store-affiliated staff
- Store owner grants streaming permission
- Store owner receives 0–10% tip share

### 5.4 Key Live Stream Screen Features

#### 5.4.1 Viewer Side

- **Live Viewer Count:** Real-time concurrent viewers
- **Comments:** Send messages in chat format
- **Tips:** Send gift items (consumes coins)
- **Likes:** Heart animation
- **Share:** Share stream URL

#### 5.4.2 Broadcaster Side

- **Viewer Count:** Current and cumulative viewers
- **Comment Display:** Scrolling view
- **Tip Reception:** Real-time gift notification
- **Camera Switch:** Front/rear camera toggle
- **Mic On/Off:** Audio mute
- **End Stream:** Terminate session

### 5.5 Collab & Battle Screen Layouts

#### Solo Stream

```
┌─────────────────────┐
│                     │
│  Broadcaster (Full) │
│                     │
│                     │
└─────────────────────┘
```

#### Duet (2 people)

```
┌──────────┬──────────┐
│          │          │
│  Host 1  │  Host 2  │
│          │          │
│          │          │
└──────────┴──────────┘
```

#### Trio (3 people)

```
┌──────────┬──────────┐
│          │          │
│  Host 1  │  Host 2  │
│          │          │
├──────────┴──────────┤
│                     │
│       Host 3        │
└─────────────────────┘
```

#### Squad (4 people)

```
┌──────────┬──────────┐
│          │          │
│  Host 1  │  Host 2  │
├──────────┼──────────┤
│          │          │
│  Host 3  │  Host 4  │
└──────────┴──────────┘
```

### 5.6 Live Stream Start Flow

#### 5.6.1 Solo Stream

1. Tap "Start Live Stream" on staff dashboard
2. Solo session created automatically (transitions to `CreateCollabScreen`)
3. Broadcast screen opens immediately (after permission check)
4. Live stream begins

#### 5.6.2 Collab & Battle Stream

1. Tap "Start Collab Stream" on staff dashboard
2. Mode selection screen (solo, duet, trio, squad)
3. Select battle type (none, 1v1, 2v2, 4-player battle)
4. Create session
5. Invite participants (QR code, share link)
6. Stream starts once all participants are ready

### 5.7 Live Streaming Service

Key features provided by `LiveCollabService`:

- Session creation and management
- Participant management (invite, join, leave)
- Tip processing
- Comment send/receive
- Viewer count tracking
- Battle score aggregation

---

## 6. Live League System

### 6.1 Overview

TikTok-style live league system. Staff and gifter ranks are determined by total tip amount and reset weekly/monthly.

### 6.2 League Ranks (8 Tiers)

| Rank | Required Coins | Emoji | Color |
|------|---------------|-------|-------|
| Bronze | 0 – 999 | 🥉 | `#CD7F32` |
| Silver | 1,000 – 4,999 | 🥈 | `#C0C0C0` |
| Gold | 5,000 – 19,999 | 🥇 | `#FFD700` |
| Platinum | 20,000 – 49,999 | 💎 | `#E5E4E2` |
| Diamond | 50,000 – 99,999 | 💠 | `#B9F2FF` |
| Master | 100,000 – 499,999 | 👑 | `#9370DB` |
| Grand Master | 500,000 – 999,999 | ⭐ | `#FF1493` |
| Legend | 1,000,000+ | 🏆 | `#FF4500` |

### 6.3 Liver (Staff) League

Rank determined by total tip coins received by staff.

**Data fields:**
- Current league rank
- Total coins received
- Weekly coins received
- Monthly coins received
- League ranking position
- Progress rate to next league

**Reset timing:**
- Weekly: Every Monday at 00:00
- Monthly: 1st of every month at 00:00

### 6.4 Gifter League

Rank determined by total tip coins sent by users.

**Data fields:**
- Current league rank
- Total coins sent
- Weekly coins sent
- Monthly coins sent
- League ranking position
- Progress rate to next league

### 6.5 League Ranking Screen

Profile → Tap "Live League":

- **Liver League:** Staff received coin rankings
- **Gifter League:** User sent coin rankings
- **Weekly Ranking:** This week's top 10
- **Monthly Ranking:** This month's top 10
- **My Rank:** Current rank and position
- **Progress Bar:** Progress to next league

### 6.6 League System Implementation

Key features provided by `LiveLeagueService`:

- Fetch league info (liver, gifter)
- Automatic rank-up on tip receipt
- Weekly/monthly ranking retrieval
- Weekly/monthly reset processing

---

## 7. Live Collab & Battle System

### 7.1 Shard System

When a user sends a tip, they receive shards. Shards are collectible items exchangeable for rewards.

#### 7.1.1 Shard Earn Rate

| Tip Coins | Shards Earned | Bonus Rate |
|-----------|--------------|------------|
| 100 coins | 1 shard | — |
| 500 coins | 6 shards | +20% |
| 1,000 coins | 15 shards | +50% |
| 5,000 coins | 80 shards | +60% |
| 10,000 coins | 170 shards | +70% |

**Formula:**

```
Base shards   = tip coins / 100
Bonus shards  = base shards × bonus rate
Total shards  = base shards + bonus shards
```

#### 7.1.2 Shard Redemption Rewards

| Shards Required | Reward |
|----------------|--------|
| 50 shards | Bronze Badge |
| 100 shards | Silver Badge |
| 500 shards | Gold Badge |
| 1,000 shards | Limited Stamp Set |
| 5,000 shards | Platinum Frame |
| 10,000 shards | Diamond Icon |

### 7.2 Shard Collection Screen

Profile → Tap "Shard Collection":

- **Balance:** Current shard count
- **Earn History:** Shard earn records per tip
- **Reward List:** Available rewards with previews
- **Redeem Button:** Exchange shards for rewards
- **Collection:** Display of earned rewards

### 7.3 Battle System

#### 7.3.1 Battle Types

- **None:** Regular collab stream
- **1v1 Battle:** 2 broadcasters compete by tip amount
- **2v2 Battle:** 2 teams (2 people each) compete by total team tips
- **4-Player Battle Royale:** All 4 participants compete individually

#### 7.3.2 Battle Score Calculation

- Each participant's score = total tip coins received
- Scoreboard updated in real time
- Winner determined when battle ends

#### 7.3.3 Battle States

- **Waiting:** Before battle start
- **Ready:** All participants ready
- **Active:** Battle in progress
- **Finished:** Battle ended, results displayed

#### 7.3.4 Battle Rewards

- **Winner:** Bonus shards (+50%)
- **All participants:** Earn base shards

### 7.4 Collab Session Management

```dart
class LiveCollabSession {
  final String sessionId;
  final String hostUserId;
  final CollabMode mode;           // solo, duet, trio, squad
  final BattleType battleType;     // none, 1vs1, 2vs2, freeForAll
  final BattleStatus battleStatus; // waiting, ready, active, finished
  final List<CollabParticipant> participants;
  final int currentViewers;        // Concurrent viewer count
  final DateTime startedAt;
  final DateTime? endedAt;
}
```

### 7.5 Concurrent Viewer Count Display

- Concurrent viewer count displayed at the top of the live screen
- **Real-time updates:** Instantly reflects viewers joining/leaving
- **Icon:** 👁 count

---

## 8. Tip (Gift) System

### 8.1 Overview

Users send gifts to staff using coins to support staff monetization.

### 8.2 Gift Items

**Standard Gifts:**

| Gift Name | Coins | Emoji |
|-----------|-------|-------|
| Heart | 10 | ❤️ |
| Rose | 50 | 🌹 |
| Cake | 100 | 🎂 |
| Bouquet | 500 | 💐 |
| Diamond | 1,000 | 💎 |
| Crown | 5,000 | 👑 |
| Rocket | 10,000 | 🚀 |

Special gifts (event-limited, seasonal, etc.) can also be added.

### 8.3 Tip Sending Flow

1. Tap the gift button on the staff detail or live stream screen
2. Gift item list is displayed
3. Select a gift
4. Confirm dialog (gift name, coin cost, balance)
5. Tap the send button
6. Coins consumed, gift sent
7. Gift notification sent to staff
8. Animation displayed (during live stream)
9. Sender earns shards

### 8.4 Tip Revenue Distribution

#### 8.4.1 Base Distribution (Independent Staff)

| Recipient | Share |
|-----------|-------|
| Staff | 70% |
| Platform | 30% |

**Example:** 10,000 coins (~¥18,600 web) tip
- Staff receives: 7,000 coins (~¥13,020)
- Platform: 3,000 coins (~¥5,580)

#### 8.4.2 Store-Affiliated Staff Distribution

Store owner's configured share rate (0–10%) is applied.

**Example:** Store share rate of 5%

| Recipient | Share |
|-----------|-------|
| Staff | 65% |
| Store | 5% |
| Platform | 30% |

**Example:** 10,000 coin tip (store share rate 5%)
- Staff receives: 6,500 coins (~¥12,090)
- Store receives: 500 coins (~¥930)
- Platform: 3,000 coins (~¥5,580)

#### 8.4.3 Store Share Rate Setting

- Store owner sets the rate (0–10%) in the management screen
- Default is 0% (no store share)
- Applied to all store-affiliated staff

### 8.5 Tip History

Profile → Tap "Tip History":

- List of sent tips (date/time, staff name, gift name, coin amount)
- Total coins sent
- Monthly/weekly summaries
- Per-staff summaries

### 8.6 Gifter Level System

Gifter level increases based on total tip amount:

| Level | Required Total (Coins) | Reward |
|-------|----------------------|--------|
| 1 | 0 – 999 | — |
| 2 | 1,000 – 4,999 | Silver Badge |
| 3 | 5,000 – 19,999 | Gold Badge |
| 4 | 20,000 – 49,999 | Platinum Badge |
| 5 | 50,000+ | Diamond Badge |

---

## 9. Headhunting System

### 9.1 Overview

A feature for company users to send scout offers to staff, facilitating recruitment.

### 9.2 Company Registration

Profile → Tap "Company Management" → Register Company:

- Company name
- Company type (restaurant, beauty salon, other)
- Address
- Phone number
- Email address
- Website URL
- Company description
- Register as store: Checkbox (enables store features)

### 9.3 Sending a Headhunting Offer

Company management screen → "Send Offer to Staff":

1. Search for staff
2. Select staff
3. Fill in offer details:
   - Job title
   - Salary range
   - Work location
   - Message
4. Send offer

### 9.4 Offer Management

Company management screen → "Offer Management":

- **Sent:** List of sent offers
- **Accepted:** Offers accepted by staff
- **Rejected:** Offers rejected by staff
- **Expired:** Offers past their expiry date

### 9.5 Staff Side (Receiving)

Profile → "Received Offers":

- List of headhunting offers
- Offer detail view
- **Accept Button:** Accept the offer
- **Decline Button:** Decline the offer
- **Company Details Button:** View company information

---

## 10. Store Management System

### 10.1 Overview

A feature for store owners to register their store and manage affiliated staff.

### 10.2 Store Registration

Checking "Register as Store" during company registration enables store features.

**Store information:**
- Store name
- Store type (restaurant, beauty salon, other)
- Address
- Phone number
- Business hours
- Closed days
- Store description
- Store images

### 10.3 Store Staff Management

Profile → Tap "Store Management":

- **Affiliated Staff List:** Current affiliated staff
- **Send Offer to Staff:** Scout new staff
- **Offer Management:** Sent, accepted, rejected offers
- **Tip Share Rate Setting:** Set rate in 0–10% range
- **Live Stream Permission Management:** Bulk manage streaming rights for affiliated staff

### 10.4 Sending a Store Staff Offer

Store management screen → "Send Offer to Staff":

1. Search for staff
2. Select staff
3. Fill in offer details:
   - Job title
   - Salary range
   - Work location
   - Start date
   - Message
4. Send offer

### 10.5 Differences Between Store Offers and Headhunting Offers

| Item | Store Offer | Headhunting Offer |
|------|------------|------------------|
| Sender | Store owner | Company user |
| Purpose | Hire store-affiliated staff | General scouting |
| Benefits | Tip share, streaming rights | Salary, benefits, etc. |
| Management screen | Store Management | Company Management |

### 10.6 Benefits for Store-Affiliated Staff

#### 10.6.1 Live Streaming Rights

- **No follower requirement:** Store-affiliated staff can stream immediately
- Store owner manages permissions

#### 10.6.2 Tip Share

- Receive a share of tips at the rate set by the store owner (0–10%)
- Staff earnings decrease, but they gain store backing

---

## 11. Booking System

### 11.1 Overview

A feature for users to make reservations (bookings) with staff.

### 11.2 Bookable Staff

Staff who enable "Accept Bookings" in their settings can receive reservations.

### 11.3 Booking Creation Flow

Staff detail screen → Tap "Book Now":

1. Select date and time (calendar view)
2. Select service menu (staff's registered services)
3. Enter number of people and notes
4. Confirmation screen
5. Confirm booking (Stripe payment or pay later)

### 11.4 Booking Management

#### 11.4.1 User Side

Profile → Tap "Booking History":

- **Upcoming:** Future bookings
- **Completed:** Past bookings
- **Cancelled:** Cancelled bookings
- Booking detail view (date/time, staff name, service, price)

#### 11.4.2 Staff Side

Staff dashboard → "Booking Management":

- **Today's Bookings:** Today's reservation list
- **Upcoming Bookings:** Future reservations
- **Past Bookings:** Past reservations
- Booking detail view
- **Approve/Reject:** Approve or reject a booking
- **Mark Complete:** Mark as complete after service delivery

### 11.5 Service Menu Management

Staff dashboard → "Menu Management":

- Menu list (service name, price, duration, description)
- Add/edit/delete menus
- Image upload

### 11.6 Coupon Management

Staff dashboard → "Coupon Management":

- Coupon list (coupon name, discount rate/amount, expiry date)
- Create/edit/delete coupons
- Auto-generate coupon codes
- Usage history

---

## 12. Messaging

### 12.1 Overview

Users and staff (or users among themselves) can send and receive messages.

### 12.2 Message Sending Flow

1. Staff detail screen → Tap "Message" button
2. Chat screen opens
3. Enter text, attach images
4. Tap send button
5. Message delivered to recipient in real time

### 12.3 Messaging Features

- **Text Messages:** Standard text sending
- **Image Sending:** Camera capture or gallery selection
- **Stickers:** Emoji stickers
- **Read Receipts:** Read/unread status
- **Notifications:** New message notifications

### 12.4 Message List

Profile → "Messages":

- Recent chat list
- Unread message count
- Last message preview
- Tap chat room to open individual chat screen

### 12.5 Staff Message Management

Staff dashboard → "Check Messages":

- List of received messages
- Unread message count
- Reply

---

## 13. Review System

### 13.1 Overview

Users can post reviews (ratings and comments) for staff.

### 13.2 Review Posting Flow

Staff detail screen → Tap "Write a Review":

1. Select star rating (1–5 stars)
2. Enter comment
3. Attach image (optional)
4. Tap post button
5. Review is published

### 13.3 Review Display

Staff detail screen:

- **Average Rating:** Average of all star ratings (e.g. 4.5★)
- **Review Count:** Total number of reviews
- **Review List:** Latest reviews displayed
- Review details (poster name, star rating, comment, images, post date/time)

### 13.4 My Reviews

Profile → "Reviews":

- List of reviews you have posted
- Edit/delete reviews

---

## 14. Admin Features

### 14.1 Admin Dashboard

Admin-only management screen:

- **User Management:** List/edit/delete regular users, staff, companies, and admins
- **Content Moderation:** Delete posts, comments, and reviews
- **Report Management:** Handle user reports
- **Push Notifications:** Send to all users or specific users
- **SNS Integration Management:** Instagram, Twitter, Facebook, etc. integration status
- **Operations Support Chat:** Handle user inquiries

### 14.2 Admin Login

Admin-only login screen (`/admin/login`):

- Admin ID and password
- Demo admin account: `admin@example.com` / `admin123`

### 14.3 User Management

- Display user list (ID, name, email, role, registration date)
- User detail view
- Edit user (name, email, role change)
- Delete user (soft delete)
- Search users (name, email, role)

### 14.4 Content Moderation

- Display post list (image, video, text, poster, post date/time)
- Delete posts
- Display comment list
- Delete comments
- Display review list
- Delete reviews

### 14.5 Push Notification Sending

- **Target:** All users, specific users, specific roles
- **Title:** Notification title
- **Body:** Notification message
- **Action:** Destination on notification tap (screen, URL)
- **Scheduling:** Send immediately or at a specified date/time

---

## 15. Tech Stack

### 15.1 Frontend

- **Flutter:** 3.35.4
- **Dart:** 3.9.2
- **UI Framework:** Material Design 3
- **State Management:** Provider

### 15.2 Backend (Demo Mode)

- **Local Storage:** SharedPreferences (demo data storage)
- **Firebase Admin SDK:** `/opt/flutter/firebase-admin-sdk.json` (backend services)
- **Google Services:** `/opt/flutter/google-services.json` (Android app)

### 15.3 Payments

- **Stripe:** Demo mode (test environment)
  - Card number: `4242 4242 4242 4242`
  - Expiry: Any future date
  - CVC: Any 3 digits

### 15.4 Database (Future Implementation)

- **Firebase Firestore:** Cloud database
- **Hive:** Local database (offline support)

### 15.5 Notifications

- **Firebase Cloud Messaging (FCM):** Push notifications

### 15.6 Storage

- **Firebase Storage:** Cloud storage for images, videos, and files

### 15.7 Authentication

- **Firebase Authentication:** Email/password auth (future implementation)
- **SharedPreferences:** Demo authentication (current)

---

## 16. Data Models

### 16.1 Main Model List

#### 16.1.1 User

```dart
class User {
  final String id;
  final String name;
  final String email;
  final String? avatarUrl;
  final String? bio;
  final bool isStaff;
  final bool isCompany;
  final DateTime createdAt;
}
```

#### 16.1.2 Staff

```dart
class Staff {
  final String id;
  final String name;
  final String? avatarUrl;
  final String jobTitle;
  final String location;
  final String? bio;
  final List<String> skills;
  final double rating;
  final int reviewCount;
  final int followersCount;
  final bool isAvailable;
  final Map<String, String>? socialLinks;
}
```

#### 16.1.3 Company

```dart
class Company {
  final String id;
  final String name;
  final String companyType;
  final String address;
  final String phoneNumber;
  final String email;
  final String? websiteUrl;
  final String description;
  final bool isStore;    // Registered as store
  final String ownerId;
  final String ownerName;
  final List<String> staffIds;
  final DateTime createdAt;
  final DateTime updatedAt;
}
```

#### 16.1.4 PointPackage

```dart
class PointPackage {
  final String id;
  final String name;
  final int points;
  final int webPrice;
  final int appPrice;
  final String platform;   // 'web' or 'app'
  final bool isPopular;
  int get price;           // Price for current platform
  int get totalPoints;     // Coins earned
  double get pricePerCoin; // Cost per coin
}
```

#### 16.1.5 UserPointBalance

```dart
class UserPointBalance {
  final String userId;
  final int totalPoints;
  final int purchasedPoints;
  final int bonusPoints;
  final int usedPoints;
  final DateTime lastUpdated;
  int get availablePoints => totalPoints - usedPoints;
}
```

#### 16.1.6 LeagueRank

```dart
enum LeagueRank {
  bronze, silver, gold, platinum, diamond, master, grandmaster, legend
}
```

#### 16.1.7 LiverLeagueInfo

```dart
class LiverLeagueInfo {
  final String staffId;
  final String staffName;
  final LeagueRank currentLeague;
  final int totalCoinsReceived;
  final int weeklyCoins;
  final int monthlyCoins;
  final int rank;
  final int totalShards;
  final DateTime lastUpdated;
}
```

#### 16.1.8 GifterLeagueInfo

```dart
class GifterLeagueInfo {
  final String userId;
  final String userName;
  final LeagueRank currentLeague;
  final int totalCoinsSent;
  final int weeklyCoins;
  final int monthlyCoins;
  final int rank;
  final int totalShards;
  final DateTime lastUpdated;
}
```

#### 16.1.9 LiveShard

```dart
class LiveShard {
  final String id;
  final String userId;
  final String staffId;
  final int shardCount;
  final DateTime earnedAt;
  final String source;  // 'gift', 'daily_bonus', 'event'
  final int giftCoins;
}
```

#### 16.1.10 CollabMode

```dart
enum CollabMode {
  solo, duet, trio, squad
}
```

#### 16.1.11 BattleType

```dart
enum BattleType {
  none, oneVsOne, twoVsTwo, freeForAll
}
```

#### 16.1.12 LiveCollabSession

```dart
class LiveCollabSession {
  final String sessionId;
  final String hostUserId;
  final CollabMode mode;
  final BattleType battleType;
  final BattleStatus battleStatus;
  final List<CollabParticipant> participants;
  final int currentViewers;  // Concurrent viewer count
  final DateTime startedAt;
  final DateTime? endedAt;
}
```

#### 16.1.13 CollabParticipant

```dart
class CollabParticipant {
  final String userId;
  final String userName;
  final String? avatarUrl;
  final bool isHost;
  final bool isMuted;
  final bool isCameraOff;
  final DateTime joinedAt;
  final int giftCoinsReceived;  // Coins received during battle
}
```

#### 16.1.14 TipHistory

```dart
class TipHistory {
  final String id;
  final String senderId;
  final String senderName;
  final String receiverId;
  final String receiverName;
  final int amount;
  final String message;
  final DateTime sentAt;
}
```

#### 16.1.15 Booking

```dart
class Booking {
  final String id;
  final String userId;
  final String staffId;
  final DateTime bookingDate;
  final String serviceType;
  final String status;  // 'pending', 'confirmed', 'completed', 'cancelled'
  final String? notes;
  final DateTime createdAt;
}
```

#### 16.1.16 Review

```dart
class Review {
  final String id;
  final String userId;
  final String userName;
  final String staffId;
  final int rating;
  final String comment;
  final List<String>? imageUrls;
  final DateTime createdAt;
}
```

#### 16.1.17 HeadhuntOffer

```dart
class HeadhuntOffer {
  final String id;
  final String companyId;
  final String companyName;
  final String staffId;
  final String staffName;
  final String jobTitle;
  final String salaryRange;
  final String location;
  final String message;
  final String status;  // 'pending', 'accepted', 'rejected', 'expired'
  final DateTime createdAt;
  final DateTime expiresAt;
}
```

#### 16.1.18 StoreStaffOffer

```dart
class StoreStaffOffer {
  final String id;
  final String storeId;
  final String storeName;
  final String staffId;
  final String staffName;
  final String position;
  final String salaryRange;
  final String location;
  final DateTime startDate;
  final String message;
  final String status;  // 'pending', 'accepted', 'rejected', 'expired'
  final DateTime createdAt;
  final DateTime expiresAt;
}
```

---

## 17. Screen List

### 17.1 Auth & Profile

- Login screen (`login_screen.dart`)
- User registration screen (`register_screen.dart`)
- Profile screen (`profile_screen.dart`)
- Profile edit screen (`profile_edit_screen.dart`)
- Profile settings screen (`profile_settings_screen.dart`)
- Notification settings screen (`notification_settings_screen.dart`)
- Privacy settings screen (`privacy_settings_screen.dart`)
- Password change screen (`password_change_screen.dart`)

### 17.2 Staff Search & Discovery

- Home screen (`home_screen.dart`)
- Search screen (`search_screen.dart`)
- Map search screen (`map_search_screen.dart`)
- Filter settings screen (`filter_settings_screen.dart`)
- Staff detail screen (`staff_detail_screen.dart`)
- Ranking screen (`ranking_screen.dart`)
- Favorites screen (`favorites_screen.dart`)
- Following screen (`following_screen.dart`)

### 17.3 Staff Management

- Staff dashboard (`staff/staff_dashboard_screen.dart`)
- Staff profile edit (`staff/staff_profile_edit_screen.dart`)
- Posts management (`staff/staff_posts_screen.dart`)
- Tip history (`staff/staff_tips_screen.dart`)
- Booking management (`staff/staff_booking_management_screen.dart`)
- Coupon management (`staff/staff_coupon_management_screen.dart`)
- Menu management (`staff/staff_menu_management_screen.dart`)
- Earnings screen (`staff/earnings_screen.dart`)
- Received offers (`staff_received_offers_screen.dart`)

### 17.4 Coins & Points

- Coin purchase screen (`point_purchase_screen.dart`)
- Coin earn screen (`point_earn_screen.dart`)
- Send tip screen (`send_tip_screen.dart`)
- Tip history screen (`tip_history_screen.dart`)

### 17.5 Live Streaming

- Live stream list (`live_stream_list_screen.dart`)
- Live feed (`live_feed_screen.dart`)
- Live stream screen (`live_stream_screen.dart`)
- Live broadcaster screen (`live_broadcaster_screen.dart`)
- Live viewer screen (`live_viewer_screen.dart`)
- Live league screen (`live_league_screen.dart`)
- Shard collection screen (`live_shard_screen.dart`)
- Collab & battle screen (`live_collab_battle_screen.dart`)
- Create collab screen (`create_collab_screen.dart`)

### 17.6 Company & Store Management

- Company registration screen (`company/company_registration_screen.dart`)
- Company management screen (`company/company_management_screen.dart`)
- Company offers screen (`company/company_offers_screen.dart`)
- Send headhunting offer (`company/send_headhunting_offer_screen.dart`)
- Store staff management (`company/company_staff_management_screen.dart`)
- Send store staff offer (`company/send_store_staff_offer_screen.dart`)
- Store staff offer list (`company/store_staff_offers_list_screen.dart`)

### 17.7 Booking

- User booking screen (`booking/user_booking_screen.dart`)
- Booking list screen (`booking/user_booking_list_screen.dart`)
- Booking detail screen (`booking/booking_detail_screen.dart`)

### 17.8 Messaging

- Message list screen (`messages_screen.dart`)
- Chat screen (`chat_screen.dart`)
- New message screen (`create_message_screen.dart`)
- User chat screen (`user_chat_screen.dart`)
- Staff chat screen (`staff_chat_screen.dart`)

### 17.9 Reviews

- Write review screen (`write_review_screen.dart`)
- My reviews screen (`my_reviews_screen.dart`)

### 17.10 Admin

- Admin login (`admin/screens/admin_login_screen.dart`)
- Admin dashboard (`admin/screens/admin_dashboard_screen.dart`)
- User management (`admin/screens/users_management_screen.dart`)
- Staff management (`admin/screens/staff_management_screen.dart`)
- Content moderation (`admin/screens/content_moderation_screen.dart`)
- Report management (`admin/screens/reports_screen.dart`)
- Push notification sender (`admin/screens/admin_push_notification_screen.dart`)
- SNS integration management (`admin/screens/sns_management_screen.dart`)
- Operations support chat (`admin/screens/admin_support_chat_screen.dart`)

### 17.11 Other

- Notification list screen (`notifications_screen.dart`)
- Help & support screen (`help_support_screen.dart`)
- Support chat (`support_chat_screen.dart`)
- Block management screen (`user_block_management_screen.dart`)
- Privacy policy (`privacy_policy_screen.dart`)
- Story viewer (`story_viewer_screen.dart`)
- Post detail screen (`post_detail_screen.dart`)

---

## 18. Service List

### 18.1 Auth & User Management

- `AuthService` (`services/auth_service.dart`)
- `LocalAuthService` (`services/local_auth_service.dart`)

### 18.2 Staff Management

- `FavoriteService` (`services/favorite_service.dart`)
- `FollowService` (`services/follow_service.dart`)
- `StoryService` (`services/story_service.dart`)

### 18.3 Coins & Tips

- `PaymentService` (`services/payment_service.dart`)
- `TipService` (`services/tip_service.dart`)
- `GifterService` (`services/gifter_service.dart`)

### 18.4 Live Streaming

- `LiveStreamService` (`services/live_stream_service.dart`)
- `LiveLeagueService` (`services/live_league_service.dart`)
- `LiveCollabService` (`services/live_collab_service.dart`)

### 18.5 Company & Store

- `CompanyService` (`services/company_service.dart`)
- `HeadhuntService` (`services/headhunt_service.dart`)
- `StoreStaffOfferService` (`services/store_staff_offer_service.dart`)

### 18.6 Booking & Reviews

- `LocalBookingService` (`services/local_booking_service.dart`)
- `ReviewService` (`services/review_service.dart`)

### 18.7 Messaging & Notifications

- `ChatService` (`services/chat_service.dart`)
- `NotificationService` (`services/notification_service.dart`)
- `FCMService` (`services/fcm_service.dart`)

### 18.8 Other

- `LocationService` (`services/location_service.dart`)
- `MediaService` (`services/media_service.dart`)
- `ExportService` (`services/export_service.dart`)
- `AppModeService` (`services/app_mode_service.dart`)

---

## 19. Future Roadmap

### 19.1 Payment Integration

- **Stripe Production:** Payment Intent and Webhook integration
- **Extended Coin Usage:** Uses beyond tipping (premium features, ad removal, etc.)

### 19.2 Live Streaming Enhancements

- **WebRTC Integration:** Real-time video/audio communication
- **Stream Quality Settings:** Resolution and bitrate adjustment
- **Recording Feature:** Archive storage for live streams

### 19.3 League System Enhancements

- **Rank-Up Animation:** Animation on league promotion
- **Ranking Rewards:** Weekly/monthly top ranker rewards
- **Badge Display:** Show badges on profile and in chat

### 19.4 Shard System Enhancements

- **More Rewards:** A wider variety of reward items
- **Limited-Time Rewards:** Special event-exclusive rewards
- **Shard Gacha:** Random reward draws

### 19.5 Headhunting Enhancements

- **Matching Algorithm:** AI-based staff recommendations
- **Auto Offer Sending:** Automatic offers to matching staff
- **Interview Scheduling:** Schedule interviews after offer acceptance

### 19.6 Store Management Enhancements

- **Revenue Analytics:** Aggregate and visualize tips and booking revenue
- **Staff Performance:** Individual staff performance analysis
- **Shift Management:** Register and manage work shifts

### 19.7 AI Features

- **Chatbot:** Automated responses and FAQ handling
- **Content Recommendations:** Staff recommendations based on user interests
- **Inappropriate Content Detection:** AI-powered auto moderation

---

## 20. Summary

Staff Finder is a next-generation staff matching platform integrating TikTok-style live streaming, tipping, headhunting, store management, and more.

### Key Highlights

- **TikTok-like UX:** Live streaming, coins, leagues, shards, and battle systems
- **Monetization:** A mechanism for staff to earn revenue through tips
- **Corporate Integration:** Headhunting and store management to support hiring
- **SNS Features:** Follow, posts, stories, likes, and comments
- **Booking & Reviews:** Direct bookings and reviews between users and staff

### Technical Strengths

- High-quality cross-platform development with Flutter 3.35.4
- Highly scalable backend via Firebase integration
- Smooth payment experience with Stripe
- Modern UI with Material Design 3

### Future Vision

- Real-time live streaming via WebRTC
- AI-powered matching, recommendations, and moderation
- Additional monetization features (premium plans, ad revenue, etc.)

---

*End of document*
