# 02. Architecture

[← README](../README.md) · [한국어](ko/02-architecture.md)

## Components

```
Browser (React 19 SPA · in-house routing · in-house state store)
   │  fetch — /api/v2/*
   ▼
node server/server.mjs        ← node --watch (restarts when a file is saved)
   ├ ecn10        10-step engineering change · ledger grid
   ├ projects     projects · expense requests
   ├ docs/docFiles document metadata · attachments (session required)
   ├ masterData   users · teams
   ├ auth         session-cookie authentication
   ├ mailer       mail delivery · notification dispatcher
   ├ notifyRules  notification rules (one definition shared with the client)
   ├ agent*       AI agent (tool definitions · LLM providers · document Q&A · resubmission helper)
   ├ meetingRec   recording → transcription → AI review
   ├ fileServer*  company file server relay · filename index · revision lineage
   └ docPreview   shared document preview
   ▼
node:sqlite (WAL, 38 tables)
   + Python workers — XLSX parsing · PDF/DOCX form rendering · BOM diff · EML parsing
```

## Why it's built this way

| Decision | Reason |
|---|---|
| **Zero npm runtime deps on the server** | There is no practical way to install or update packages on the company server. With `node:sqlite` in Node 22, even the database driver is built in |
| **React is the only frontend dependency** | Address routing (back/forward, bookmarks, refresh, shareable links) and the store are written by hand. The workflows are simple state transitions, and direct control reads better |
| **SQLite (WAL)** | One server, tens of users. One file to back up, migrate and roll back, and the production cutover was a DB file migration |
| **Python for documents** | The official forms had to be reproduced exactly, down to blank cells and "(e.g. …)" hint text: "when opened later, it should look like the same template with the information filled in." That needs reportlab, PyMuPDF and openpyxl |
| **Optional features stay optional** | Without AI, transcription or the file server, only those features turn off. Core work is isolated from external outages |

## Data model

| Group | Contents |
|---|---|
| ECN source & history | Migrated ledger · 10-step progress · workflow events · department actions · deletion log |
| Ledger grid overlay | Cells · cell history · rows · columns · cell styles · row heights · per-user views |
| Projects · spend · documents | Projects · expense requests · document metadata · attachments · BOM comparisons · meeting recordings |
| Accounts · sessions | User and team master data; sessions (**only the sha256 of each token is stored**, 12-hour sliding expiry) |
| Audit · notifications · settings | Audit log · read receipts · settings |
| Mail | Intake candidates · outbox |
| AI | Summary batches · chat sessions · messages |

- Projects, expense requests and ECN bodies are stored as **`payload_json` + `version`**; saves compare the version (**optimistic locking**).
- All deletes are **soft deletes**, and every write lands in the audit log.
- The server guarantees unique numbering; concurrent intakes that collide are renumbered by the server.

## Permission model

Two separate systems:

1. **Menu permissions.** 10 menus × read/write/delete = 30 switches per user. Screen guards always go through a single `can(user, menu, action)`. Settings are always denied to non-admins regardless of switches (a hard gate). New users default to read-only.
2. **Approval permissions.** Decided by named approver, title and department on the approval line. A named step requires an exact name match; otherwise the title must match.

Deleting **the last holder of a title referenced by an approval line** triggers a warning. Without that person, approvals would never complete.

## Scheduled jobs

| Job | Interval |
|---|---|
| Ledger source workbook signature watch | 60 s |
| Mailbox (.eml) scan → intake candidates | 60 s |
| Notification mail delivery | 10 min |
| File-server change journal | 5 min |
| AI summary batch | 10 min (only when an LLM is connected) |

Startup order is fixed: ledger signature check → mailbox scan → AI connection attempt and fallback decision → first mail run. If the server restarts under `--watch`, indexing **resumes from folders not yet expanded**.

## Mail

- The client **cannot choose approval-mail recipients**; the server derives them from the approval line.
- Minutes and transfer notices take recipients **by name/ID only**, and the server resolves addresses, which blocks sending to arbitrary addresses.
- Notification rules (who gets what, when) live in **one definition file shared by client and server**.

## Shared UI building blocks

| Piece | Description |
|---|---|
| Canonical UI components | Buttons, inputs, tables and alert boxes are defined once (`src/ui`). No per-screen pixel or `!important` overrides; adjust only by defined steps |
| Monochrome line icons | Replaced color emoji with `currentColor`, 1em, fixed-stroke icons (emoji look different per OS and ignore text color) |
| Identity re-check dialog | A single Promise-based global host. **If the host is missing, the check fails**, so nothing passes without authentication |
| Menu tour | Shown once per account; replay only from settings |
| Responsive layout | Tables and step bars re-flow by width so wrapping never ruins readability; mobile supported |
