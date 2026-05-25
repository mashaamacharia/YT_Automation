# Phase 5 — Voiceover Generation

**Workflow ID:** `DdZPlfQhHcj8FPmN`  
**Total Nodes (Phases 1–5):** 69  
**Phase 5 Nodes:** 23 (including sticky note)

---

## Flow Diagram

```
Code — Store Script in Static Data  (Phase 4 exit)
  │
  ▼
ElevenLabs — Generate Voiceover       POST /v1/text-to-speech/{voice_id}, 120s timeout
  │
  ▼
Code — Check TTS Response             classify HTTP status
  │
  ▼
IF — TTS Request Failed?
  │ false (ok)              │ true (error)
  ▼                         ▼
R2 — Upload Audio MP3    IF — Rate Limited (429)?
                           │ true              │ false (401/403/other)
                           ▼                  ▼
                    Wait — 60s          Telegram — Alert ElevenLabs Auth Failed
                    Rate Limit Backoff  (stop)
                           │
                           └──► ElevenLabs — Generate Voiceover  (retry loop)

R2 — Upload Audio MP3        PUT to R2, binary body
  │
  ▼
Code — Check R2 Upload Result       derives public URL from env vars
  │
  ▼
IF — R2 Upload Failed?
  │ false (ok)              │ true
  ▼                         ▼
FFmpeg — Get Audio Duration  Code — R2 Upload Retry Counter   (max 3)
                               │
                               ▼
                          IF — R2 Upload Max Retries Exceeded?
                            │ true                │ false
                            ▼                     ▼
                    Telegram — Alert        Wait — Upload Retry Backoff
                    R2 Upload Failed        (5s / 15s / 45s)
                    (stop)                        │
                                                  └──► R2 — Upload Audio MP3  (retry)

FFmpeg — Get Audio Duration      POST /get-duration → { total_seconds }
  │
  ▼
Code — Classify Audio Duration   sets duration_status: ok | too_short | too_long
  │
  ▼
Switch — Duration Status
  │ ok                │ too_short           │ too_long
  ▼                   ▼                     ▼
Code — Store     Telegram — Alert      Telegram — Alert Audio Too Long
Audio URL        Audio Too Short       (includes resume URL)
(→ Phase 6)      (stop)                      │
                                        Wait — Operator Long Audio Decision
                                              │
                                        Code — Parse Long Audio Reply
                                              │
                                        IF — Operator Approved Long Audio?
                                          │ true (/approve)   │ false (/regenerate)
                                          ▼                   ▼
                                    Code — Store          LLM — Write Script (Claude)
                                    Audio URL             (loops back to Phase 4)
                                    (→ Phase 6)
```

---

## Nodes

### `ElevenLabs — Generate Voiceover` — `p5-tts`
- **Type:** HTTP Request (POST)
- **URL:** `https://api.elevenlabs.io/v1/text-to-speech/{{ $env.ELEVENLABS_VOICE_ID }}`
- **Headers:** `xi-api-key`, `Content-Type: application/json`, `Accept: audio/mpeg`
- **Timeout:** 120s — narration can be 300+ words; TTS inference takes time
- **neverError:** true — error classification handled downstream
- **responseFormat:** `arraybuffer` — captures binary MP3 data for direct R2 upload
- **Body:**
```json
{
  "text": "{{ $workflow.staticData.script.full_narration }}",
  "model_id": "eleven_multilingual_v2",
  "voice_settings": {
    "stability": 0.75,
    "similarity_boost": 0.85,
    "style": 0.40,
    "use_speaker_boost": true
  }
}
```

---

### `Code — Check TTS Response` — `p5-check-tts-error`
Reads `$response.statusCode` from the HTTP response metadata.

| Status | `tts_error` | `error_type` |
|--------|-------------|--------------|
| 200 | false | null |
| 429 | true | `rate_limit` |
| 401 / 403 | true | `auth_fail` |
| Other 4xx/5xx | true | `api_error` |

---

### `IF — TTS Request Failed?` — `p5-if-tts-error`
- **false** → `R2 — Upload Audio MP3`
- **true** → `IF — Rate Limited (429)?`

### `IF — Rate Limited (429)?` — `p5-if-rate-limit`
- **true** → `Wait — 60s Rate Limit Backoff` → loops back to TTS
- **false** → `Telegram — Alert ElevenLabs Auth Failed` → stop

---

### `R2 — Upload Audio MP3` — `p5-upload-r2`
- **Method:** PUT
- **URL:** `https://{{ $env.R2_ACCOUNT_ID }}.r2.cloudflarestorage.com/{{ $env.R2_BUCKET_NAME }}/audio/{{ $workflow.staticData.job_id }}.mp3`
- **Body:** binary (`dataPropertyName: "data"`) — passes arraybuffer from TTS directly
- **Content-Type:** `audio/mpeg`
- **Timeout:** 120s — MP3 files can be 10–30 MB

### `Code — Check R2 Upload Result` — `p5-check-upload`
On 2xx: constructs `audio_url = $env.R2_PUBLIC_URL + '/audio/' + job_id + '.mp3'`, stores to `$workflow.staticData.audio_url`, returns `{ upload_error: false, audio_url }`.  
On non-2xx: returns `{ upload_error: true, status }`.

### `Code — R2 Upload Retry Counter` — `p5-upload-retry-counter`
Uses `$workflow.staticData.r2_upload_attempt` as persistent counter across retries.

| Attempt | Backoff | `give_up` |
|---------|---------|-----------|
| 1 | 5s | false |
| 2 | 15s | false |
| 3 | 45s | false |
| 4+ | — | **true** |

### `Wait — Upload Retry Backoff` — `p5-wait-upload-backoff`
- `resume: timeInterval`
- Duration is dynamic: `={{ $json.waitSeconds }}` (5 / 15 / 45)
- On completion → loops back to `R2 — Upload Audio MP3`

---

### `FFmpeg — Get Audio Duration` — `p5-get-duration`
- **Method:** POST `$env.FFMPEG_SERVER_URL/get-duration`
- **Body:** `{ "audio_url": "{{ $json.audio_url }}" }`
- Returns `{ total_seconds: 312 }`

### `Code — Classify Audio Duration` — `p5-check-duration`
Reads `total_seconds`, stores to `$workflow.staticData.total_seconds`, returns `duration_status`:

| Condition | `duration_status` |
|-----------|-------------------|
| < 180s | `too_short` |
| 180–420s | `ok` |
| > 420s | `too_long` |

### `Switch — Duration Status` — `p5-switch-duration`
Three named outputs: `ok` → store, `too_short` → alert stop, `too_long` → alert + wait.

---

### `Telegram — Alert Audio Too Short` — `p5-alert-too-short`
Sends duration + topic + Job ID. **No resume link** — this is a hard stop. Operator must send `/start` to begin a new run with a longer script.

### `Telegram — Alert Audio Too Long` — `p5-alert-too-long`
Sends duration + Job ID + `$execution.resumeUrl`. Options presented:
- `/approve` — continue with the long audio as-is
- `/regenerate` — discard audio, loop back to Phase 4 LLM for a new script

### `Wait — Operator Long Audio Decision` — `p5-wait-long-audio`
- `resume: webhook`, no timeout
- `webhookSuffix: "long-audio-decision"`

### `Code — Parse Long Audio Reply` — `p5-parse-long-reply`
```
/approve     → { action: "approve" }
/regenerate  → { action: "regenerate" }
other/blank  → { action: "approve" }  ← conservative default
```

### `IF — Operator Approved Long Audio?` — `p5-if-approve`
- **true** (`/approve`) → `Code — Store Audio URL in Static Data`
- **false** (`/regenerate`) → `LLM — Write Script (Claude)` (Phase 4 re-entry)

---

### `Code — Store Audio URL in Static Data` — `p5-store-audio`
**Shared terminus** for both `ok` and `approved-long` paths.

Persists to `$workflow.staticData`:
- `audio_url` — R2 public URL of the MP3
- `r2_upload_attempt` — reset to 0 for future runs
- `phase` — `"phase_5_complete"`

Outputs `{ job_id, audio_url, total_seconds }` for Phase 6.

---

## StaticData After Phase 5

| Key | Value |
|-----|-------|
| `audio_url` | `https://{R2_PUBLIC_URL}/audio/{job_id}.mp3` |
| `total_seconds` | Integer — actual audio duration from FFmpeg |
| `r2_upload_attempt` | Reset to 0 |
| `phase` | `"phase_5_complete"` |

---

## Error Handling Summary

| Failure | Handling |
|---------|----------|
| ElevenLabs 429 | Wait 60s → retry TTS (loops until success) |
| ElevenLabs 401/403 | Telegram alert → hard stop |
| ElevenLabs other 4xx/5xx | Treated as auth fail → Telegram alert → stop |
| R2 upload fail (attempt 1) | Wait 5s → retry |
| R2 upload fail (attempt 2) | Wait 15s → retry |
| R2 upload fail (attempt 3) | Wait 45s → retry |
| R2 upload fail (attempt 4+) | Telegram alert → hard stop |
| FFmpeg /get-duration fail | Returns no `total_seconds` → classified as `error` → falls through Switch fallback (no connection = stop) |
| Audio too short (< 180s) | Telegram alert → hard stop |
| Audio too long (> 420s) | Telegram alert + wait → /approve continues, /regenerate loops to Phase 4 |

---

## Key Design Decisions

1. **`arraybuffer` response format on TTS** — ElevenLabs returns raw binary MP3. Capturing as `arraybuffer` lets n8n pass the binary body directly to the R2 PUT without any base64 conversion or intermediate storage.

2. **R2 URL derived from env vars** — the PUT response body from R2 is empty on success (standard S3 behaviour). The public URL is therefore constructed deterministically: `R2_PUBLIC_URL + '/audio/' + job_id + '.mp3'`. No parsing of the response needed.

3. **Retry counter in staticData** — using `$workflow.staticData.r2_upload_attempt` means the counter survives node transitions. It is reset to 0 after successful store so it doesn't affect future runs.

4. **`/regenerate` loops to Phase 4 LLM, not Phase 4 Store** — if the operator asks to regenerate, we want a completely fresh script, not the same one re-stored. The loop target is `LLM — Write Script (Claude)` which pulls topic from staticData.

5. **120s timeout on both TTS and R2 upload** — at ElevenLabs Starter tier, a 5-minute narration (~750 words) can take 15–40s to generate. 30s would frequently time out. R2 uploads of a 20 MB MP3 over a typical VPS connection also warrant headroom.
