# Phase 4 — Script Generation

**Workflow ID:** `9QjxbrMpek4zAId8`  
**Total Nodes (Phases 1–4):** 46  
**Phase 4 Nodes:** 13 (including sticky note)

---

## Flow Diagram

```
Code — Store Chosen Topic  (Phase 3 exit)
  │
  ▼
LLM — Write Script (Claude)           first attempt, 60s timeout
  │
  ▼
Code — Validate Script JSON           parse + 5 structural checks
  │
  ▼
IF — Script Validation Failed?
  │ true (failed)           │ false (passed)
  ▼                         ▼
LLM — Write Script          Code — Store Script in Static Data  ──► Phase 5
  Retry (Strict Prompt)
  │
  ▼
Code — Validate Retry Script JSON     same checks, marks final_failure on fail
  │
  ▼
IF — Retry Script Also Failed?
  │ true (final fail)       │ false (passed)
  ▼                         ▼
Telegram — Send Raw       Code — Store Script in Static Data  ──► Phase 5
  Script Output
  │
  ▼
Wait — Operator Script Decision       webhook resume, no timeout
  │
  ▼
Code — Parse Operator Script Reply
  │
  ▼
IF — Operator Cancelled Script?
  │ true (/skip)            │ false (/retry)
  ▼                         ▼
Telegram — Notify         LLM — Write Script (Claude)   (loops back to top)
  Script Generation
  Cancelled
```

---

## Nodes

### `LLM — Write Script (Claude)` — `p4-script-writer`
- **Type:** HTTP Request (POST)
- **URL:** `https://api.anthropic.com/v1/messages`
- **Model:** `claude-opus-4-5`, `max_tokens: 4000`
- **Timeout:** 60s (longer than research phase — script is 4000 tokens)
- **neverError:** true — HTTP errors handled downstream in validation node
- **System prompt:** Professional mystery scriptwriter, returns raw JSON only
- **Topic injection:** `$workflow.staticData.chosen_topic` — reads from staticData, not `$json`, so it survives the /retry loop correctly

**Full script schema requested:**
```json
{
  "title": "max 60 chars, curiosity-gap style",
  "description": "SEO-rich, 150 words",
  "tags": ["10 tags"],
  "thumbnail_text": "max 4 words",
  "chapters": "0:00 Introduction\n...",
  "scenes": [
    {
      "scene_id": 1,
      "narration": "full narration text",
      "duration_seconds": 8,
      "visual_keyword": "specific descriptive keyword",
      "mood": "tense|eerie|dramatic|calm|shocking",
      "is_hook": true,
      "clip_marker": "HOOK"
    }
  ],
  "full_narration": "complete script as one continuous string",
  "short_clip_scenes": [1, 8, 15]
}
```

---

### `Code — Validate Script JSON` — `p4-validate-script`
Five sequential checks. Returns `{ error: true/false, final_failure: false }`.

| Check | Condition | Error message |
|-------|-----------|---------------|
| API error | `statusCode >= 400` or `.error` exists | `Claude API error: {code}` |
| JSON parse | `JSON.parse()` throws | `JSON parse failed: {e.message}` |
| Scene count | `scenes.length < 15` | `Too few scenes: {n}` |
| Duration | `totalDuration < 240 \|\| > 420` | `Duration out of range: {n}s` |
| Narration length | `full_narration.length < 500` | `Narration too short: {n} chars` |
| Markers | Missing HOOK, CLIMAX, or ENDING | `Missing scene markers. HOOK:... CLIMAX:... ENDING:...` |

On success returns `{ error: false, script, totalDuration, job_id }`.

---

### `IF — Script Validation Failed?` — `p4-check-error`
- **true** (output 0) → retry path
- **false** (output 1) → `Code — Store Script in Static Data`

---

### `LLM — Write Script Retry (Strict Prompt)` — `p4-script-retry`
Identical API call but system prompt is reinforced:

> "IMPORTANT: Return ONLY the raw JSON object. No other text whatsoever. No markdown. No code fences. Pure JSON only."

Topic still pulled from `$workflow.staticData.chosen_topic`.

---

### `Code — Validate Retry Script JSON` — `p4-validate-retry`
Same structural checks as first-attempt validation, but sets `final_failure: true` instead of `final_failure: false` on any failure. This is what the downstream IF node checks to distinguish retry from first-attempt failures.

---

### `IF — Retry Script Also Failed?` — `p4-check-retry`
- Checks `$json.final_failure === true`
- **true** → Telegram fallback path
- **false** → `Code — Store Script in Static Data`

---

### `Telegram — Send Raw Script Output to Operator` — `p4-send-raw-script`
Sends to `$env.OPERATOR_TELEGRAM_CHAT_ID`:
- Topic name
- Job ID
- Failure reason
- First 2000 chars of raw LLM output
- Available commands: `/retry` or `/skip`
- `$execution.resumeUrl` included so the operator can wire a Telegram reply handler to resume execution

---

### `Wait — Operator Script Decision` — `p4-wait-operator`
- `resume: webhook` — pauses execution indefinitely
- **No timeout** — operator decides when to act, no auto-cancel at this stage
- `webhookSuffix: "script-decision"` — unique path per execution

---

### `Code — Parse Operator Script Reply` — `p4-parse-op-reply`
```
/retry  → { action: "retry" }
/skip   → { action: "cancel" }
blank   → { action: "cancel" }
other   → { action: "cancel" }  ← conservative default
```

---

### `IF — Operator Cancelled Script?` — `p4-if-skip`
- Checks `$json.action === "cancel"`
- **true** → `Telegram — Notify Script Generation Cancelled` → workflow stops
- **false** (action = "retry") → loops back to `LLM — Write Script (Claude)`

---

### `Code — Store Script in Static Data` — `p4-store-script`
**Shared terminus for all three success paths** (first attempt, retry, operator-approved retry).

Persists to `$workflow.staticData`:
- `script` — full script object
- `total_duration` — sum of all `scene.duration_seconds`
- `phase` — `"phase_4_complete"`

Outputs `{ job_id, script, totalDuration }` for Phase 5.

---

## StaticData After Phase 4

| Key | Value |
|-----|-------|
| `script` | Full script object (title, description, tags, scenes, full_narration, etc.) |
| `total_duration` | Integer seconds (expected 270–330) |
| `phase` | `"phase_4_complete"` |

All prior keys (`job_id`, `ideas`, `chosen_topic`, `chosen_topic_summary`) remain intact.

---

## Error Handling Summary

| Failure | Handling |
|---------|----------|
| Claude API HTTP error (4xx/5xx) | Caught in validate node → retry path |
| Malformed JSON response | Caught in validate node → retry path |
| Too few scenes (< 15) | Caught in validate node → retry path |
| Duration out of range | Caught in validate node → retry path |
| Narration too short | Caught in validate node → retry path |
| Missing HOOK/CLIMAX/ENDING markers | Caught in validate node → retry path |
| Retry also fails | Telegram alert + wait for operator |
| Operator replies /skip | Pipeline cancelled cleanly |
| Operator replies /retry | Loops back to first LLM call (fresh generation) |

---

## Key Design Decisions

1. **Topic from staticData, not $json** — the `/retry` loop reconnects to `LLM — Write Script (Claude)` directly. Using `$workflow.staticData.chosen_topic` means the topic is always available regardless of how many times the loop runs.

2. **Single `Store Script` node** — all three success paths (first attempt, retry, operator-approved retry) converge on one node. No duplicate store logic.

3. **No timeout on operator Wait** — unlike Phase 3 where an auto-cancel after 48h makes sense, a failed script is a blocking problem. The operator should decide when to retry or abandon.

4. **Timeout increased to 60s** — the script call requests 4000 tokens vs 1000 for research. 30s is too tight for a cold Claude API response at that token budget.

5. **`final_failure` flag** — the retry validation node sets this flag explicitly. This allows the IF node to check it directly rather than re-checking `error` (which is also true on the first-attempt path). Keeps the routing logic unambiguous.
