# CLAUDE.md — Screen & Feature Specs

This directory contains all implementation specs for the staffsearch platform. Claude Code must follow every rule in this file when reading, writing, or updating any spec.

---

## Reading Order (Before Implementing Any Ticket)

Always read these files first, in this order:

1. `_INDEX.md` — find the ticket, check its status, identify its dependencies
2. `_CONVENTIONS.md` — mandatory Go and Flutter coding conventions
3. The spec file for the ticket being implemented
4. Any spec files listed as dependencies

---

## Rules for Writing a New Spec

### R1 — Start From the Template

Always copy `_TEMPLATE.md` as the starting point. Never write a spec from scratch or freehand.

### R2 — English Only

All content must be in English. No Vietnamese, no Japanese — except for literal on-screen UI text that appears in Japanese in the actual app (e.g. `スタッフサーチ` inside a wireframe).

### R3 — No Code Blocks

Specs describe what to build, not how to build it. Never include:
- Dart or Flutter code
- Go code
- SQL statements
- JSON samples
- Any fenced code block (``` ... ```)

Replace any urge to write code with a prose sentence, a numbered step list, or a table.

### R4 — Required Sections in Order

Every spec must contain all of these sections in this exact order:

| # | Section | When Required |
|---|---|---|
| 0 | Header metadata table | Always |
| 1 | Overview + end-to-end flow diagram | Always |
| 2 | User Flows (happy paths + error cases table) | Always |
| 3 | UI / Screen (wireframe + element table + changes list) | Always |
| 4 | Frontend — Flutter (files to create/modify + responsibilities) | Always |
| 5 | Backend — Go + Fiber (endpoint table + per-endpoint contract) | Always |
| 6 | Database | Only if schema changes |
| 7 | Configuration | Only if new env vars |
| 8 | Testing (backend test table + Flutter test table) | Always |
| 9 | Security Checklist | Always for auth, payments, or user data |
| 10 | File Map | Always |
| 11 | Acceptance Criteria | Always |

If a section does not apply, write the heading and one line: *"Not applicable for this ticket."* Never silently omit a section.

### R5 — Every Endpoint Needs a Full Contract

For every endpoint in Section 5, provide:
- Method + full route path + whether JWT is required
- Request body field table: field / type / required / validation
- Response field table for every success status
- Error response table: HTTP status / error code / condition — must include at minimum `400 bad_request`, `401 unauthorized` (if auth required), `500 server_error`
- Handler responsibilities as a numbered step list

### R6 — Error Messages Must Be Exact Strings

Every error case in Section 2 must show the exact string displayed to the user. Do not write "show an error". Write `"Google authentication failed. Please try again."` Exact strings prevent ambiguity for frontend and QA.

### R7 — File Map Must Be Exhaustive

Section 10 must list every file that will be created or modified — both Go backend and Flutter frontend. For modified files, state specifically what changes, not just "Modify".

### R8 — Acceptance Criteria Must Be Testable

Every item in Section 11 must be verifiable by running the app or an automated test. Minimum 8 items. No vague criteria like "app works correctly".

### R9 — Dependencies Must Reference Specific Files and Sections

Do not write "depends on AUTH-01". Write "depends on AUTH-01 — `ApiClient` and `AuthProvider` created in Sections 4.1–4.4".

### R10 — Update the Index After Every Spec

After writing or editing a spec:
1. Change the ticket from `🔲` to `✅` in `_INDEX.md`
2. Add the file to the "Files in This Folder" table
3. Update the Progress Tracker count for the group

### R11 — Self-Review Checklist Before Finishing

Before considering a spec complete, verify:

- [ ] No `<placeholder>` text remaining anywhere
- [ ] No code blocks anywhere
- [ ] Every endpoint has a full contract (R5)
- [ ] Every error case has an exact string (R6)
- [ ] File Map lists every file for both Go and Flutter (R7)
- [ ] Acceptance Criteria has at least 8 testable items (R8)
- [ ] Dependencies reference specific files and sections (R9)
- [ ] `_INDEX.md` has been updated (R10)
- [ ] Language is English throughout (R2)

---

## Requesting Spec Changes (For Clients)

When the client provides a new or updated spec file:

```
/change-spec staff-search-docs/spec/main_app_specification_ja.md
```

Claude Code reads the new file, compares against all current ticket specs, shows a diff report (new/changed/removed features), and updates after client approval — following all rules (R2–R11). See `_CHANGE_REQUEST_TEMPLATE.md` for usage examples.

---

## Files in This Directory

| File | Purpose |
|---|---|
| `CLAUDE.md` | This file — auto-loaded rules for Claude Code |
| `_INDEX.md` | Master ticket index, reading guide, progress tracker |
| `_CONVENTIONS.md` | Go and Flutter coding conventions |
| `_SPEC_RULES.md` | Full rules reference (human-readable, same content as this file) |
| `_TEMPLATE.md` | Blank spec template — always copy this for new specs |
| `_CHANGE_REQUEST_TEMPLATE.md` | Guide for clients to request spec changes via `/change-spec` |
| `AUTH-01-email-password.md` | Email/password auth, registration, password reset, PP consent |
| `AUTH-02-google-signin.md` | Google OAuth for iOS, Android, Web |
