# Product Lifecycle Workflow Platform

[한국어](README.md)

An in-house web platform that turns a manufacturer's paper-and-spreadsheet **engineering change (ECN)** and **new product development** procedures into enforced, auditable workflows, so the same procedure runs the same way no matter who handles it. **In production since 2026-09-08.**

> This is a portfolio showcase repository. The source code belongs to an internal production system and is not published. Screenshots use synthetic data, and company, customer and internal-system names have been generalized.

## Highlights

- **Procedures to rules.** I read the existing company standards clause by clause and turned them into screens, gates and a single rule function (`ecnAdvanceBlocker`) that decides every "can this move forward?" question for steppers, buttons, alarms, dashboards and the AI agent alike.
- **Block vs. record.** There are three tiers: unapproved gates are hard-blocked, incomplete forms are blocked, and missing deliverables may proceed *with a reason*, while the alarm stays on. If you block everything, people start working outside the system.
- **Simplified while in production.** Based on user feedback after go-live, gates went from 3 to 2, approval classes were removed, and the distribution list now derives from the approval line. Step numbers stayed stable for data compatibility.
- **Zero runtime dependencies on the server.** Node 22 built-ins only (`node:http`, `node:sqlite`, `node:crypto`, `node:child_process`). The frontend depends only on React, with routing and state handled in-house.
- **A decade of spreadsheet ledger migrated.** The source rows stay intact, and human edits live in a per-cell overlay with history.
- **AI with guard rails.** A natural-language agent has 31 tools. The LLM never writes SQL and never writes to the DB: writes go through user confirmation and existing UI functions. Permissions are checked server-side by session, never by prompt. Each message makes at most 2 LLM calls and runs at most 3 tools. A provider abstraction plus a "JSON-only" tool protocol lets it run on a gateway without native tool calling.
- **Privacy by design.** Names, customers and document numbers are masked before external LLM calls. Speech-to-text was chosen by data residency, because meeting audio cannot be masked.
- **Shipping safely to live users.** More than 210 production deploys since go-live. Hashed bundles stay in place, so swapping `index.html` back is an instant rollback. A strict version policy and a change ledger with more than 1,300 entries are kept.

## Stack

React 19 · TypeScript 6 · Vite 8 · Node.js 22 (built-ins only) · SQLite (WAL) · Python (reportlab, PyMuPDF, openpyxl, python-docx) · LLM gateway / Gemini / Ollama · domestic-region STT · KRDS-based design system

## Docs (Korean)

[Process redesign](docs/01-process-redesign.md) · [Architecture](docs/02-architecture.md) · [AI design](docs/03-ai-design.md) · [ADRs](docs/04-engineering-decisions.md) · [Operations](docs/05-operations.md)
