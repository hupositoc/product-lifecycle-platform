# 03. AI design — "AI prepares, people decide"

[← README](../README.md) · [한국어](ko/03-ai-design.md)

I started from the assumption that AI in a business system will sometimes be wrong. The goal is a structure where **a wrong answer never leaves wrong data behind**.

## 1. AI agent

```
User → natural language → LLM (intent · tool choice) → tool (server code) → existing business logic → DB
```

### Hard lines

| Never | Instead |
|---|---|
| The LLM writes SQL | Reads only execute a predefined tool's `run()` |
| The LLM writes to the DB | The server doesn't execute write tools; the UI shows a confirmation and the **existing intake function** runs |
| Prompts enforce permissions | The server checks the **session** before any tool runs |
| The LLM invents required values | Missing values are asked for; the UI submit validator is the final gate |
| Autonomous loops | Per message: **at most 2 LLM calls and 3 tools** |

### Why required fields exist in two places (on purpose)
The server-side list of required fields is a **hint so the LLM asks good questions**; the client-side validator is **the real gate**. If they disagree, the gate wins: the worst case is one extra question, never a missing value being saved. The single source of truth for required fields is the client-side spec file.

### Tools (31)
| Group | Examples |
|---|---|
| Read | ECN search · details · approval status · pending application · frequently changed parts · models · document search · monthly progress · project search |
| Document Q&A | Questions about document **contents** · answers grounded in procedure clauses · writing guides · menu help · user manual |
| Write (after confirmation) | ECN intake · set scope · create project · draft expense request · approve · personal schedule |
| Suggest | Approval line suggestions · similar past cases · resubmission helper |
| Navigation | Open a screen · open a portal · view/change permissions (admin) |

### A provider layer
The LLM provider changed in production, from an external API to an internal gateway. The UI contract stayed the same and only the provider was swapped. Expecting it to happen again, the agent got a **provider layer that only knows how to call models** from the start; business knowledge (prompts, tools, permissions) sits above it.

The internal gateway has no native tool calling: you open a session, send text and get text back. Tool calls therefore use a **"output JSON only" protocol**, which keeps working when the provider changes.

## 2. Copilot intent detection — hybrid

```
Question → first pass by local rules
         → server masks names, customers and document numbers, then asks the LLM for intent/filters only
         → on failure, fall back to the local rules
         → lookups, aggregation and wording all happen internally
```

- **Principle: company data (customers, BOMs, prices, real names) never leaves the company.**
- Daily call budget, a **10-minute circuit breaker** on HTTP 429, per-reason failure metrics and a no-answer rate.
- **11 fallback conditions** (timeout, empty reply, truncated tokens, schema violation, …) are distinguished, and the badge shows the honest reason. When the quota runs out, it suggests questions the rule engine handles reliably.
- Metrics showed the real reason fallback quality was poor wasn't the model. **The local classifier was a few lines of regex.** It was rewritten as per-intent scoring with slot extraction, a free and offline improvement first.

## 3. Meeting recording → minutes

```
① Upload      — keep the browser's MediaRecorder output (webm/opus)
② Transcribe  — speaker diarization + segment timestamps
③ Review      — the LLM drafts discussion and decisions → follow-ups go onto calendars
```

- **Why three independent APIs**: transcription fails and retries often (network, quotas, long files) and costs money. As long as the original audio is on the server, it can be re-run any number of times. Doing it in one shot would lose the whole meeting on failure.
- **The STT provider was chosen by data location.** A cheaper overseas service existed, but engineering-change meeting audio can't be masked, so I chose a service that **keeps data in the country**.
- Credentials never go into source. They come only from a server config file or environment variables. API responses reveal **only whether a key is configured**, never the key itself.

## 4. Auto-filled fields with an "AI-generated" tag

AI suggestions first appeared as cards with an "Apply" button. In use, that turned out to be just one more button.

→ Values the AI suggests (approval lines, quotes, schedules, completion reports, …) now go **straight into the existing fields**, and that section's header gets a small **"AI-generated"** tag. The section flashes briefly the first time it scrolls into view. People only review and correct. The tag's look is defined once in the canonical UI.

## 5. Document Q&A

You can ask about a document's **contents**, not just its name. Text is extracted from documents in the document store and the company file server, answers are grounded in the relevant passages, and **the source documents are shown with the answer**. Procedure questions cite the relevant clause.
