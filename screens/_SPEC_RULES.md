# Spec Writing Rules

These rules are mandatory for Claude and any contributor when writing or editing a spec file in `docs/screens/`. They exist to ensure every spec is consistent, complete, and immediately implementable without follow-up questions.

---

## Rule 1 — Always Start From the Template

Every new spec MUST be created by copying `_TEMPLATE.md`. Never write a spec from scratch. The template defines the required section order and ensures nothing is accidentally omitted.

---

## Rule 2 — Language

All spec content MUST be written in **English**. This applies to:
- All section headings
- All prose descriptions
- All table content
- All wireframe labels
- All error messages and toast strings
- Comments inside ASCII diagrams

The only exceptions are UI labels that are displayed in Japanese in the actual app (e.g. `スタッフサーチ` inside a wireframe). These may remain in Japanese because they represent the literal on-screen text.

---

## Rule 3 — No Code Blocks

Specs are descriptive documents, not implementation files. They MUST NOT contain:
- Dart / Flutter code
- Go code
- SQL statements (except in migration section, which uses a table format instead)
- JSON examples that go beyond a field name / type table
- Any fenced code block (``` ... ```)

All implementation details are expressed as prose, numbered steps, or tables. If you feel the urge to write code, replace it with a sentence describing what the code does.

---

## Rule 4 — Required Sections

Every spec MUST include these sections, in this order, regardless of the ticket type:

| # | Section | Required |
|---|---|---|
| 0 | Header metadata table (Type, Priority, Estimate, Dependencies, Platforms, Conventions link) | Always |
| 1 | Overview + end-to-end flow diagram | Always |
| 2 | User Flows (happy paths + error cases table) | Always |
| 3 | UI / Screen (wireframe + element specs + changes required) | Always |
| 4 | Frontend — Flutter (files to create/modify + responsibilities) | Always |
| 5 | Backend — Go + Fiber (endpoint table + per-endpoint contract) | Always |
| 6 | Database | Only if schema changes |
| 7 | Configuration | Only if new env vars |
| 8 | Testing (backend + flutter test case tables) | Always |
| 9 | Security Checklist | Always for auth, payments, or user data features |
| 10 | File Map | Always |
| 11 | Acceptance Criteria | Always |

If a section is not applicable (e.g. a pure-frontend ticket with no backend changes), write the section heading and one line: *"Not applicable for this ticket."* Do not silently omit sections.

---

## Rule 5 — Every Endpoint Must Have a Full Contract

For every backend endpoint listed in Section 5, you MUST provide:

- Method + full route path
- Whether JWT is required
- Request body field table (field, type, required, validation) — or "No body" for GET
- Response field table for every success status code
- Error response table (HTTP status, error code, condition) — must include at minimum `400 bad_request`, `401 unauthorized` (if auth required), and `500 server_error`
- Handler responsibilities as a numbered list of steps

A spec that lists an endpoint without its full contract is incomplete and must not be marked ✅ in `_INDEX.md`.

---

## Rule 6 — Error Messages Must Be Exact Strings

Every error case in Section 2 (User Flows) must specify the exact string shown to the user in the SnackBar or dialog. Do not write "show an error message" — write `"Google authentication failed. Please try again."` Exact strings allow frontend and QA to work without ambiguity.

---

## Rule 7 — File Map Must Be Complete

Section 10 (File Map) must list every file that will be created or modified as a result of this ticket — both Go and Flutter. A file that is not in the File Map will likely be missed during implementation.

If a file is modified, state what specifically changes, not just "Modify".

---

## Rule 8 — Acceptance Criteria Must Be Testable

Every item in Section 11 (Acceptance Criteria) must be a statement that can be verified by running the app or an automated test. Avoid vague criteria.

| Not acceptable | Acceptable |
|---|---|
| "App works correctly" | "A new user can register and is navigated to `/home` upon success" |
| "Errors are handled" | "A 401 response shows a red SnackBar with the exact error message from the server" |
| "Looks good on all platforms" | "The Google Sign-In button renders with a white background and `#DADCE0` border on iOS, Android, and Web" |

Minimum 8 acceptance criteria per spec. Complex features should have 12–15.

---

## Rule 9 — Dependencies Must Reference Real Specs

If a ticket depends on another ticket, the dependency MUST reference the specific spec file and section where the shared infrastructure is defined.

| Not acceptable | Acceptable |
|---|---|
| "Depends on AUTH-01" | "Depends on AUTH-01 — `ApiClient` and `AuthProvider` are created there (Sections 4.1–4.4)" |
| "Uses the standard error format" | "Uses the standard error envelope defined in `_CONVENTIONS.md` Section 2" |

---

## Rule 10 — Update the Index After Every Spec

After writing or updating a spec, you MUST update `_INDEX.md`:

1. Change the ticket's status from `🔲` to `✅`
2. Add or update the file link in the status table
3. Update the Progress Tracker count for the relevant group
4. Add the file to the "Files in This Folder" table at the bottom

A spec that exists on disk but is not reflected in `_INDEX.md` is effectively invisible to future readers.

---

## Rule 11 — Self-Review Before Marking Complete

Before marking a spec as done, run through this checklist:

- [ ] All 11 required sections are present and filled in (no `<placeholder>` text remaining)
- [ ] No code blocks anywhere in the document
- [ ] Every endpoint has a full contract (request, response, errors, handler steps)
- [ ] Every error case has an exact user-facing string
- [ ] File Map lists every file (create or modify) for both Go and Flutter
- [ ] Acceptance Criteria has at least 8 testable items
- [ ] Dependencies reference specific spec files and sections
- [ ] `_INDEX.md` has been updated
- [ ] Language is English throughout (except literal Japanese UI text)

If any item is unchecked, the spec is not complete.
