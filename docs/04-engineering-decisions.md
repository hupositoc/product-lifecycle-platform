# 04. Architecture Decision Records (ADR)

[← README](../README.md) · [한국어](ko/04-engineering-decisions.md)

Each decision records **context → decision → consequence**, so that "why did we do this?" always has an answer.

---

### ADR-01 · The server uses Node built-ins only
- **Context.** Installing or updating packages on the company server is impractical, and remote access is limited to file sharing and HTTP.
- **Decision.** Build the server on `node:http`, `node:sqlite`, `node:crypto` and `node:child_process` only, with zero npm runtime dependencies.
- **Consequence.** Installing means installing Node, with no supply-chain risk. The trade-off: routing, cookies and multipart parsing had to be written and tested by hand.

### ADR-02 · One rule function decides blocking
- **Context.** If the stepper, buttons, dashboard and alarms each decide "can this advance?", they drift apart.
- **Decision.** Rule functions such as `ecnAdvanceBlocker()` and `ecnClosureBlocker()` are the single source of truth. Every decision uses the step **number**, never the step name.
- **Consequence.** Procedure redesigns (3 → 2 gates, approval classes removed) shipped without pausing in-flight cases.

### ADR-03 · Ledger migration as source + per-cell overlay
- **Context.** The ten-year spreadsheet ledger kept being updated in the spreadsheet for a while after migration. Neither a one-off import nor a continuously overwriting sync would do.
- **Decision.** Watch the workbook's signature (hash) to refresh source rows, and store human edits in a **per-cell overlay** on top.
- **Consequence.** Source and edits never mix, and each cell keeps its own history.

### ADR-04 · Optimistic locking for concurrent edits
- **Context.** Several departments keep the same case open at once. Pessimistic locks would stall work whenever someone walks away.
- **Decision.** Compare a `version` column; if it changed, reject the save and reload the latest copy.
- **Consequence.** Concurrent viewing with no overwrite accidents.

### ADR-05 · A read-only contract for the external file server
- **Context.** It holds data for the whole organization. One bug that deletes a file would be hard to recover from.
- **Decision.** Route outbound calls through one `guardedFetch`, return 403 outside the allowed roots on every path, and block a restricted folder across indexing, browsing, search and download. Writes (backup upload) live in a separate module.
- **Consequence.** The integration's reach is visible in one place in the code.

### ADR-06 · Our own filename index ("Everything"-style)
- **Context.** The file server's search API is slow and weak at partial matches.
- **Decision.** Build the filename index in the server DB ahead of time, update it from the change journal, and search normalized names (NFC, lowercase, separators removed) by substring with AND across tokens.
- **Consequence.** Instant search. The index lives only on our server, so the file server is never modified.

### ADR-07 · Design external integrations from measurements
- **Context.** Unlike its docs, the events API ignored `skip`/`page`: every call returned the latest 100 events, and everything in between was lost.
- **Decision.** Use time windows (`from`/`to`); if a window's total exceeds the page, **split it in half** and read again. Advance the watermark only after a full window is read.
- **Consequence.** An index with no gaps. Since then, external APIs are measured before integration, with the findings left in code comments.

### ADR-08 · LLMs behind a provider layer
- **Context.** The provider changed in production (external API → internal gateway), and the gateway lacks tool calling.
- **Decision.** A provider layer that only knows how to call, plus a "JSON-only" tool protocol.
- **Consequence.** The provider was swapped without touching the UI.

### ADR-09 · Choose STT by data residency
- **Context.** Meeting audio can't be masked.
- **Decision.** Prioritize **keeping data in the country** over price when choosing a provider.
- **Consequence.** A higher unit price, but a design that can pass a security review.

### ADR-10 · Alarms are never stored
- **Context.** Stored alarms lose trust the moment they drift from the real state.
- **Decision.** Alarms are derived values computed from state by rules.
- **Consequence.** Schedule overruns, missing deliverables, budget overruns, reply deadlines and pending approvals always match the current state.

### ADR-11 · Store only token hashes
- **Decision.** HttpOnly session cookies; the DB stores only the sha256 of each token; 12-hour sliding expiry. Password re-check right before approving (60-second lock after 5 failures).
- **Consequence.** A leaked DB file can't be used to replay sessions.

### ADR-12 · Rollback comes first
- **Context.** Several releases a day while people work, and no remote server restart.
- **Decision.** Never delete hashed bundles, so restoring `index.html` is a rollback. Back up the served folder before each release and diff build copies file by file. The server restarts on file save via `--watch`.
- **Consequence.** 210+ releases since go-live.
