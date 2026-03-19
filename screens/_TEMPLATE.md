# [TICKET-NN] Feature Name

> **How to use this template:**
> Copy this file, rename it to match your ticket (e.g. `BOOK-01-create-booking.md`),
> then replace every placeholder in angle brackets `<like this>` with real content.
> Delete this instruction block and any sections marked OPTIONAL that do not apply.
> Update `_INDEX.md`: change `🔲` to `✅` and add the file to the Files table.

---

| Field | Value |
|---|---|
| **Type** | Feature / Screen / Flow / Infrastructure |
| **Priority** | P1 MVP · P2 Growth · P3 Scale |
| **Estimate** | `<X days manual>` / `<Y days vibe coding>` |
| **Dependencies** | `<TICKET-NN>` — `<reason>`. None if standalone. |
| **Platforms** | iOS · Android · Web — remove any that do not apply |
| **Conventions** | See [`_CONVENTIONS.md`](./_CONVENTIONS.md) for error format, JWT, ApiClient, AuthProvider, navigation, and token storage rules. |

---

## 1. Overview

One paragraph describing what this feature does, why it exists, and which user problem it solves. Mention the key actors (user, staff, admin) and the core data involved.

### End-to-end flow

```
[Flutter App]                    [Go API]                 [External Service?]
     │                               │                           │
     │── <action> ──────────────────►│                           │
     │                               │── <backend step> ────────►│
     │                               │◄── <response> ────────────│
     │                               │── <DB write>              │
     │◄── <response> ────────────────│                           │
     │── <local state update>        │                           │
     │── navigate to <route>         │                           │
```

---

## 2. User Flows

### 2.1 Happy Path — <primary scenario name>

1. <Step 1>
2. <Step 2>
3. ...

### 2.2 Happy Path — <secondary scenario, if any> [OPTIONAL]

...

### 2.3 Error Cases

| Scenario | Behaviour |
|---|---|
| <scenario> | <what the UI does> |
| Network error | Show red SnackBar: *"Unable to connect. Check your network and retry."* |
| Server error (500) | Show red SnackBar: *"Something went wrong. Please try again later."* |

---

## 3. UI / Screen

### 3.1 <Screen Name> (`<file_name>.dart`)

> Current location: `lib/screens/<path>/<file_name>.dart`
> Status: **Modify existing** / **Create new**

```
┌─────────────────────────────────────────┐
│  <AppBar title>                         │
│                                         │
│  <Describe layout here using            │
│   ASCII wireframe — show key            │
│   widgets and their positions>          │
│                                         │
│  [  Primary Action Button  ]            │
└─────────────────────────────────────────┘
```

Key UI elements:

| Element | Type | Behaviour |
|---|---|---|
| `<widget name>` | `<ElevatedButton / TextFormField / ...>` | `<what it does>` |

Changes required: *(list specific code changes if modifying an existing file)*

- <change 1>
- <change 2>

### 3.2 <Second Screen, if any> [OPTIONAL]

...

---

## 4. Frontend — Flutter

### 4.1 Files to Create

| File | Purpose |
|---|---|
| `lib/<path>/<file>.dart` | <one-line description> |

*(Delete this section if no new files are needed)*

### 4.2 Files to Modify

| File | Change |
|---|---|
| `lib/<path>/<file>.dart` | <what changes and why> |

### 4.3 <Service / Provider> Responsibilities [OPTIONAL]

If a new service or significant new method is introduced, describe its responsibilities here in prose — what it does, what it calls, what it returns. No code.

### 4.4 Form Validation Rules [OPTIONAL — include only if the screen has a form]

| Field | Required | Rule | Error Message |
|---|---|---|---|
| `<field name>` | Yes / No | `<validation rule>` | `"<message shown to user>"` |

---

## 5. Backend — Go + Fiber

### 5.1 Endpoints Overview

| Method | Route | Description | Auth Required |
|---|---|---|---|
| `<METHOD>` | `/api/v1/<path>` | <description> | Yes / No |

### 5.2 <METHOD> /api/v1/<path>

**Request body:** *(delete if no body, e.g. GET)*

| Field | Type | Required | Validation |
|---|---|---|---|
| `<field>` | string / int / bool | Yes / No | <rule> |

**Response `<2xx status>`:**

| Field | Type | Description |
|---|---|---|
| `<field>` | string / int / bool | <description> |

**Error responses:**

| HTTP | `error` code | Condition |
|---|---|---|
| `400` | `bad_request` | Malformed body |
| `401` | `unauthorized` | Missing or invalid JWT |
| `<NNN>` | `<code>` | <condition> |
| `500` | `server_error` | Unexpected internal error |

**Handler responsibilities** *(numbered steps — no code):*

1. <step 1>
2. <step 2>
3. ...

*(Repeat section 5.2 pattern for each additional endpoint)*

### 5.N Service Responsibilities [OPTIONAL — only for new service methods]

`<ServiceName>`:

- `<MethodName>(args) (return, error)` — <what it does in one sentence>

---

## 6. Database [OPTIONAL — only if schema changes are needed]

### 6.1 Migration

**File:** `migrations/<YYYYMMDDHHMMSS>_<description>.up.sql`
**Reverse:** `migrations/<YYYYMMDDHHMMSS>_<description>.down.sql`

New table / altered table — `<table_name>`:

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `VARCHAR(26)` PK | No | — | ULID, generated in Go |
| `<column>` | `<type>` | Yes / No | `<default>` | <description> |

New indexes: *(describe any non-obvious indexes)*

---

## 7. Configuration [OPTIONAL — only if new env vars are needed]

| Variable | Default | Description |
|---|---|---|
| `<VAR_NAME>` | `<default or "required">` | <description> |

---

## 8. Testing

### 8.1 Backend Test Cases

| # | Scenario | Expected Result |
|---|---|---|
| 1 | <happy path> | `<2xx>`, <what happens> |
| 2 | Missing required field | `400 bad_request` |
| 3 | <auth missing> | `401 unauthorized` |
| 4 | <edge case> | <expected> |

### 8.2 Flutter Test Cases

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | <happy path> | <what the UI does> |
| 2 | Network error | Red SnackBar shown; no navigation |
| 3 | <validation failure> | Inline error shown; no API call made |

---

## 9. Security Checklist [OPTIONAL — include for any feature touching auth, payments, or user data]

- [ ] <security requirement 1>
- [ ] <security requirement 2>
- [ ] Input validated server-side, not only client-side
- [ ] No sensitive data returned in error messages

---

## 10. File Map

| Layer | File | Change |
|---|---|---|
| Flutter — UI | `lib/screens/<path>/<file>.dart` | Create / Modify |
| Flutter — Service | `lib/services/<file>.dart` | Create / Modify |
| Flutter — Provider | `lib/providers/<file>.dart` | Modify |
| Flutter — Model | `lib/models/<file>.dart` | Create / Modify |
| Go — Handler | `internal/handler/<domain>_handler.go` | Create / Modify |
| Go — Service | `internal/service/<domain>_service.go` | Create / Modify |
| Go — Repository | `internal/repository/<domain>_repository.go` | Create / Modify |
| Go — Model | `internal/model/<domain>.go` | Create / Modify |
| Go — Router | `router/router.go` | Modify |
| DB Migration | `migrations/<timestamp>_<desc>.up.sql` | Create |
| DB Migration | `migrations/<timestamp>_<desc>.down.sql` | Create |
| Config | `.env.example` | Modify |

*(Remove rows that do not apply to this ticket)*

---

## 11. Acceptance Criteria

- [ ] <criterion 1 — specific, testable>
- [ ] <criterion 2>
- [ ] All API endpoints return the standard error envelope for non-2xx responses
- [ ] Loading state is shown while any async action is in progress
- [ ] Errors are displayed via red SnackBar per conventions
- [ ] <criterion N>
