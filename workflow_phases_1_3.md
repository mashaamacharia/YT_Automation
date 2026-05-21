# YouTube Automation Pipeline — Phases 1–3

**Workflow ID:** `rZkZhmPBL3A8h0Na` (was `r48nJigXV2zsU71E` before re-import with Phase 3)  
**Total Nodes:** 33  
**Timezone:** Africa/Nairobi  
**Execution Order:** v1 (sequential)

---

## Architecture Overview

```
 Schedule (Mon 8AM) ──┐
                       ├── Merge ── Generate job_id ──► Phase 2
 Telegram /start ── Validate ──┘
```

```
 Phase 2 ──► 3 parallel research APIs ──► Error handler ──► Claude ──► 5 ideas ──► Phase 3
                                                                          │
                                                                   [Retry once on parse fail]
                                                                          │
                                                                   [Fallback: raw LLM → Telegram]
```

```
 Phase 3 ──► Telegram sends 5 ideas ──► Wait (24h, webhook resume)
                                              │
                                    ┌─────────┴──────────┐
                                    │ reply within 24h   │ timeout
                                    ▼                    ▼
                             Parse Reply          Send reminder
                                    │           Wait 24h more
                            ┌───────┴───────┐       │
                            │ cancel        │       │
                            ▼               ▼       ▼
                       Notify cancel    Continue   timeout?
                                                   │         │
                                               Auto-cancel  Parse Reply
                                                              │
                                                       Store topic
                                                      ──► Phase 4
```

---

## Phase 1 — Trigger (5 nodes)

### Entry Points

| Node | Type | Details |
|------|------|---------|
| `Schedule — Every Monday 8AM` | ScheduleTrigger | Every Monday, 08:00, Africa/Nairobi |
| `Telegram — Listen for /start` | TelegramTrigger | Watches for incoming messages |
| `Validate — Check Chat ID & /start` | IF (v2.3) | Two conditions (AND): `chat.id == OPERATOR_TELEGRAM_CHAT_ID` AND `text == "/start"` — hard security gate |
| `Merge — Combine Triggers` | Merge (chooseBranch, waitForAll) | Merges both trigger paths into one stream |
| `Generate job_id` | Code (Node) | `crypto.randomBytes(8).toString("hex")` + timestamp. Stores to `$workflow.staticData` for crash recovery |

### Operation
- **Schedule** fires → flows directly to Merge → generates job_id
- **Telegram /start** → validated against env var `OPERATOR_TELEGRAM_CHAT_ID` (rejects all other chat IDs) → merge → job_id
- `$workflow.staticData` records: `job_id`, `pipeline_started_at`, `phase: "phase_1_complete"`
- Output JSON includes `trigger_source: "schedule"` or `"telegram_manual"`

### Error Handling
- Invalid chat ID → falls through to nothing (second output of IF node is empty)
- Merge uses `waitForAll` → won't proceed until both inputs arrive (but schedule trigger always fires first; Telegram trigger is optional and waits forever — **this means the schedule can't proceed until someone also sends /start**). If this is problematic, switch merge mode.

---

## Phase 2 — Research (14 nodes)

### Parallel Research Sources

| Node | API | Query | Timeout | neverError |
|------|-----|-------|---------|------------|
| `Research — YouTube Trending` | YouTube Data API v3 `/search` | q="mystery\|unsolved\|true crime\|dark history", order=viewCount, last 7d, max 10 | 30s | true |
| `Research — Reddit Trending` | Reddit `.json` endpoint | r/UnresolvedMysteries+morbidquestions+interestingasfuck, top/week, limit 10 | 30s | true |
| `Research — Google Trends via SerpApi` | SerpApi `/search` | engine=google_trends, q="mystery,unsolved mystery,true crime", date=now 7-d | 30s | true |

All three are parallel after `Generate job_id` (3 outputs from one node).

### Error Handler: `Handle Research — Merge & Check Failures` (Code node)

**What it does:**
- Iterates all incoming items from the 3 parallel branches
- Classifies each response by structure (YouTube has `.items[].snippet.channelId`, Reddit has `.data.children`, SerpApi has `.interest_over_time` or `.related_queries`)
- Extracts relevant data: YouTube titles/channels, Reddit titles/upvotes, SerpApi interest/related
- Sets `all_failed: true/false` — proceeds to IF check

### Branch: All Sources Failed
- `IF — All Research Sources Failed?` checks `$json.all_failed === true`
- true → `Telegram — Alert All Research Failed`: notifies operator, workflow stops
- false → normal flow

### Claude Idea Generation

**`LLM Aggregator — Claude Generates 5 Ideas`**
- POST to `https://api.anthropic.com/v1/messages`
- Model: `claude-opus-4-5`, max_tokens: 1000
- System prompt: strict JSON-only instruction
- User prompt: inlines all 3 research sources as JSON, requests exactly 5 ideas with specific schema
- neverError: true (catches HTTP errors gracefully)

**`Parse — Validate LLM Ideas JSON`** (Code node)
- Strips markdown code fences, parses JSON
- Validates: must be array of exactly 5 ideas
- Returns `{ error: true/false, retry_needed: true/false }`
- On `retry_needed: true` → retry path

### Retry Logic
- `IF — LLM Ideas Parse Failed?` → true branch goes to retry
- `LLM Aggregator — Retry with Strict Prompt`: identical API call but even stricter system instruction ("Return ONLY the raw JSON object. No other text whatsoever.")
- `Parse — Retry LLM Ideas JSON`: same parsing, but accepts ≥3 ideas (more lenient)
- If retry also fails → `IF — Retry Also Failed?` → true branch

### Final Fallback
- `Telegram — Send Raw LLM Output to Operator`: sends raw LLM text (first 3000 chars) to Telegram with instruction to reply `/manualtopic [custom topic]`
- Workflow **does not stop here** — operator must still get to Phase 3 somehow (this is a gap: if all research and all LLM attempts fail, the workflow proceeds to Phase 3 with no valid ideas)

### Success Path
- Parse succeeds (first try or retry) → `Store Ideas in Static Data` (Code node)
- Persists `$workflow.staticData.ideas`, sets `phase: "phase_2_complete"`
- Outputs `{ job_id, ideas }` → Phase 3

---

## Phase 3 — Human Topic Selection (12 nodes)

### The First Human Touchpoint

**`Telegram — Send 5 Ideas for Selection`**
- Format: nicely formatted Markdown with emoji prefixes (1️⃣–5️⃣), title, search volume, unique angle
- Includes `$execution.resumeUrl` in the message — this is the webhook URL that, when POSTed to, will resume the Wait node with the operator's reply
- Uses `parse_mode: Markdown`

### Wait Mechanics

**`Wait — 24h for Topic Reply`**
- `resume: webhook` — execution pauses, n8n creates a unique `$execution.resumeUrl`
- `responseMode: onReceived` — as soon as the webhook is hit, execution resumes
- `limitWaitTime: true`, 24 hours — if no webhook arrives in 24h, the timeout branch fires
- `webhookSuffix: "topic-reply"` — custom suffix appended to the webhook path

### The Two-Stage Timeout (Why Two Waits)

An n8n Wait node can't natively send a reminder mid-wait. The workaround is:
1. **First Wait (24h):** if timeout fires → `IF — Timed Out at 24h?` checks `$json.body?.message?.text`
   - **Empty body** (= timeout) → sends reminder → starts second Wait
   - **Has body** (= operator replied) → forwards to Parse Reply
2. **Second Wait (24h more):** same pattern, but
   - Timeout → `Telegram — Auto-Cancel: No Response` → workflow stops
   - Reply received → same Parse Reply node

**Key insight:** Both wait paths converge on the same `Code — Parse Topic Reply` node, avoiding code duplication.

### Parse Reply Logic
```
reply = trim($json.body.message.text)

if reply is empty or "/skip":
    → { action: "cancel" }

if reply is a number 1-5 and ideas[num-1] exists:
    → { action: "proceed", topic, topic_summary }
    → stores to staticData

else (custom topic typed by operator):
    → { action: "proceed", topic: reply, topic_summary: "" }
    → stores to staticData
```

### Cancel Branch
- `IF — Action is Cancel?` → true → `Telegram — Notify Pipeline Cancelled` ("Pipeline cancelled. ✅") → workflow stops via no further connections
- `/skip`, blank reply, or anything that doesn't match 1-5 → cancel path

### Proceed Branch
- `Code — Store Chosen Topic`: persists `chosen_topic`, `chosen_topic_summary` to `$workflow.staticData`, sets `phase: "phase_3_complete"`
- Output: `{ job_id, topic, topic_summary }` → ready for Phase 4

---

## StaticData Crash Recovery

Throughout phases 1-3, `$workflow.staticData` maintains:

| Key | Set by | Purpose |
|-----|--------|---------|
| `job_id` | Generate job_id | Unique run identifier |
| `pipeline_started_at` | Generate job_id | ISO timestamp |
| `phase` | Multiple nodes | Current pipeline phase for resumption |
| `ideas` | Store Ideas in Static Data | 5 Claude-generated ideas |
| `chosen_topic` | Parse Reply / Store Chosen Topic | Final topic choice |
| `chosen_topic_summary` | Parse Reply / Store Chosen Topic | Summary of chosen topic |

If n8n crashes and workflow restarts, staticData survives and can be used to determine where to resume (though current implementation doesn't have resume logic — it's for Phase 13 analytics and future recovery).

---

## Credentials & Environment Variables

| Variable | Used By | Purpose |
|----------|---------|---------|
| `OPERATOR_TELEGRAM_CHAT_ID` | Validate + Telegram nodes | Hard-coded allowed chat ID |
| `telegram_bot_token` (credential) | Telegram trigger + send nodes | Bot authentication |
| `ANTHROPIC_API_KEY` (env var) | Both LLM Aggregator nodes | Claude API key |
| `YOUTUBE_API_KEY` (env var) | YouTube Trending node | Google API key |
| `SERPAPI_KEY` (env var) | Google Trends node | SerpApi key |

---

## Known Gaps & Edge Cases

1. **Merge waits for both triggers** — if schedule fires, execution sits at Merge waiting for Telegram /start to arrive (which may never come). Consider changing Merge mode if schedule-only runs are intended.
2. **No ideas in Phase 3** — if both LLM attempts fail, Phase 2 still proceeds to Phase 3 with null/empty ideas. Phase 3's Parse Reply checks `ideas[num-1]` which would be undefined for any numeric reply.
3. **Resume URL exposure** — the webhook URL is sent in Telegram, which is technically a public URL. Anybody with the URL can resume the execution with arbitrary data. Mitigation: n8n's base URL should be over HTTPS.
4. **No Phase 4 yet** — the "proceed" branch outputs to nothing; Phase 4 (Claude script generation) is the next build target.
