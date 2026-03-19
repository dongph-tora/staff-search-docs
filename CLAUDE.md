# CLAUDE.md — docs

This file provides guidance to Claude Code (claude.ai/code) when working in the documentation directory.

## Directory Structure

```
docs/
├── spec/                      # Full product specifications (EN + JP)
│   ├── staff_search_specification.md/.pdf    # Main English spec
│   ├── main_app_specification_ja.md/.pdf     # Main app spec (Japanese)
│   ├── admin_app_specification_ja.md/.pdf    # Admin app spec (Japanese)
│   ├── headhunting_app_specification_ja.md/.pdf  # Headhunting spec (Japanese)
│   └── staff_search_presentation_*.md/.pdf   # Project presentation
├── estimate/
│   └── ESTIMATE.md            # Phase-by-phase dev estimates with vibe coding comparisons
├── screens/                   # Per-feature implementation specs (tickets)
│   ├── CLAUDE.md              # Rules for writing specs — auto-loaded, must follow
│   ├── _INDEX.md              # Master ticket index + progress tracker — read first
│   ├── _CONVENTIONS.md        # Go + Flutter coding conventions — read before implementing
│   ├── _TEMPLATE.md           # Template for new specs — always copy this
│   ├── _SPEC_RULES.md         # Full rules reference (human-readable)
│   ├── _CHANGE_REQUEST_TEMPLATE.md  # Guide for clients to request spec changes via /change-spec
│   └── [ticket files]         # One .md per feature: AUTH-01, AUTH-02, DB-01, etc.
├── plans/                     # Implementation plans per ticket
│   ├── MASTER-PLAN.md         # Overall implementation plan
│   ├── _INDEX.md              # Plan index
│   └── [ticket]-plan.md       # One plan per implemented ticket
├── flow/
│   ├── flow.md                # User flow documentation
│   └── flow.html              # Interactive flow diagram
└── pencil/                    # Design files (.pen format)
    ├── plan_design.md         # Design planning doc
    ├── untitled.pen           # Pencil design file (use pencil MCP tools only)
    └── images/                # Exported design images
```

## Key Files to Read First

1. **`screens/_INDEX.md`** — Master index of all tickets with status. Always start here.
2. **`screens/_CONVENTIONS.md`** — Mandatory Go + Flutter coding conventions.
3. **`spec/staff_search_specification.md`** — Full English product specification.
4. **`estimate/ESTIMATE.md`** — Phase estimates and task breakdown.

## Completed Tickets (specs written)

| Ticket | Feature |
|---|---|
| AUTH-01 | Email/password auth, registration, password reset |
| AUTH-02 | Google OAuth (iOS, Android, Web) |
| DB-01 | Database schema design |
| DB-02 | Migrate local storage to database |
| DB-05 | Storage setup for media |
| FEED-01 | Create post |
| FEED-02 | TikTok-style feed |
| SEARCH-02 | GPS location search |
| STAFF-01 | Staff profile CRUD |
| STAFF-02 | Staff number generation |
| STAFF-03 | Job type selection |
| STAFF-05 | Portfolio photo upload |
| STAFF-07 | Staff profile screen (connected to API) |

## Requesting Spec Changes (For Clients)

When the client provides a new or updated spec file, use `/change-spec` to compare and update:

```
/change-spec staff-search-docs/spec/main_app_specification_ja.md
```

Claude Code will read the new file, compare against all current ticket specs, show a diff report (new/changed/removed features), and update after client approval. See `screens/_CHANGE_REQUEST_TEMPLATE.md` for usage examples.

## Rules for Writing Specs

Full rules are in `screens/CLAUDE.md` (auto-loaded when working in that directory). Key points:
- Always copy `screens/_TEMPLATE.md` for new specs
- English only (no Vietnamese, no Japanese except literal UI text)
- No code blocks — describe in prose, tables, and numbered lists
- Every endpoint needs a full contract (request/response fields, error codes)
- Every error case needs an exact user-facing string
- Update `screens/_INDEX.md` after writing or editing any spec

## Pencil Design Files

`.pen` files are encrypted and must be accessed via **pencil MCP tools only**. Do NOT use Read/Grep on `.pen` files. Use:
- `get_editor_state()` — check current editor state
- `batch_get()` — read .pen file nodes
- `batch_design()` — modify .pen file designs
- `get_screenshot()` — visual validation

## Specs Are Japanese + English

Product specs in `spec/` have both English (`staff_search_specification.md`) and Japanese versions (`*_ja.md`). The English spec is the canonical reference for implementation. Japanese specs are for stakeholder review.
