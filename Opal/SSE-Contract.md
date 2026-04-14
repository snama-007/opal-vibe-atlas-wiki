# Opal SSE Contract — Definitive Reference

## Two Streams, Two Purposes

The server has **two completely separate SSE mechanisms**. The FE must use the right one for each flow.

```
┌──────────────────────────────────────────────────────────────┐
│  CHAT TURN (progress %)                                       │
│  POST /v2/opal/assist/stream                                  │
│  Transport: fetch() + ReadableStream                          │
│  Lifetime: ONE turn — opens on POST, closes after turn_complete│
│  Events flow through: TurnStreamer (in-process async queue)    │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  REGEN + OUT-OF-BAND EVENTS                                   │
│  GET /v2/opal/assist/stream/{session_id}                      │
│  Transport: EventSource (native browser SSE)                  │
│  Lifetime: persistent — auto-reconnects, 60s idle timeout     │
│  Events flow through: Redis pub/sub channel                   │
└──────────────────────────────────────────────────────────────┘
```

**They are NOT interchangeable.** Chat turn events do NOT flow through Redis.
Regen events do NOT flow through the POST stream.

---

## Stream 1: Chat Turn (POST `/v2/opal/assist/stream`)

### How to connect

```js
const res = await fetch("/v2/opal/assist/stream", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ session_id: sessionId, prompt: userMessage }),
})
const reader = res.body.getReader()
const decoder = new TextDecoder()
let buffer = ""
```

### Wire format

```
id: sess-abc:1
data: {"type":"turn_started","session_id":"sess-abc","request_id":"req-xyz","payload":{...}}

id: sess-abc:2
data: {"type":"analyzer_done","session_id":"sess-abc","request_id":"req-xyz","payload":{...}}

```

### Event sequence (in order)

| # | `type` | When | `progress_pct` | Key `payload` fields |
|---|--------|------|----------------|---------------------|
| 1 | `turn_started` | Immediately on stream open | `5` | `prompt_preview` |
| 1a | `thinking_started` | Before first LLM call | `10` | `intent_mode` |
| 2 | `analyzer_done` | Intent analysis + entity extraction done | `25` | `intent_mode`, `safety_blocked`, `latency_ms` |
| 2a | `section_started` | Per section as dispatcher fans out (0-N) | `30` | `section_id`, `intent_mode`, `model`, `is_primary` |
| 2b | `section_done` | Per section as each completes (0-N) | `50` | `section_id`, `succeeded`, `latency_ms`, `model`, `attempts` |
| 3 | `generator_done` | LLM content generation done | `55` | `latency_ms`, `used_model`, `section_count` |
| 3a | `enrichment_started` | Before image enrichment begins | `58` | `total_artifacts` |
| 4 | `artifact_enriched` | Per image as each URL resolves (0-N events) | `60-90` | `artifact_id`, `label`, `category`, `image_url`, `image_credit`, `image_source`, `prompt_hash`, `is_regen` (false), `url_type` ("cdn") |
| 4a | `enrichment_progress` | Aggregate enrichment progress (after each artifact_enriched) | `60-90` | `resolved`, `total`, `pct` |
| 5 | `validator_done` | Output validation done | `95` | `latency_ms`, `pass` |
| 5a | `snapshot_saved` | Session snapshot auto-saved | `98` | — |
| 6 | `turn_complete` | Full result ready | `100` | `result` (same schema as JSON route) |
| 6' | `turn_error` | Unrecoverable error (instead of turn_complete) | — | `error`, `code` |
| — | `heartbeat` | Every ~5s during LLM work | absent | `seq` (monotonic counter) |

**Note:** `section_started`/`section_done` events fire between `analyzer_done` and `generator_done`. The FE uses these to populate `assistSections` state. They appear once per section in the SectionDispatcher plan (typically 1-4 sections). They carry `progress_pct` (30 for started, 50 for done) — the FE can optionally show intermediate progress during section dispatch.

**Every event payload includes `progress_pct` (integer).** The FE should read it directly instead of maintaining a lookup table. This is the authoritative progress value from the backend.

### FE progress calculation

```js
// PREFERRED: read progress_pct directly from the event payload
function getProgress(event) {
  const pct = event.payload?.progress_pct;
  if (pct != null) return pct;
  if (event.type === "heartbeat") return null;  // don't change progress
  return null;  // unknown event — no change
}
```

Legacy fallback (if `progress_pct` is absent for any reason):

```js
function getProgressFallback(eventType, artifactsSoFar, totalExpected) {
  switch (eventType) {
    case "turn_started":    return 5
    case "analyzer_done":   return 25
    case "generator_done":  return 55
    case "artifact_enriched":
      if (totalExpected <= 0) return 75
      return 60 + (30 * Math.min(artifactsSoFar / totalExpected, 1))
    case "validator_done":  return 95
    case "turn_complete":   return 100
    case "heartbeat":       return null
    default:                return null
  }
}
```

### Complete FE handler

```js
async function streamChatTurn(sessionId, prompt, handlers) {
  const { onProgress, onArtifact, onComplete, onError } = handlers
  let artifactCount = 0

  const res = await fetch("/v2/opal/assist/stream", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ session_id: sessionId, prompt }),
  })

  if (!res.ok) throw new Error(`HTTP ${res.status}`)

  const reader = res.body.getReader()
  const decoder = new TextDecoder()
  let buffer = ""

  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    buffer += decoder.decode(value, { stream: true })

    const frames = buffer.split("\n\n")
    buffer = frames.pop()  // keep incomplete frame

    for (const frame of frames) {
      if (!frame.trim()) continue
      let data = null
      for (const line of frame.split("\n")) {
        if (line.startsWith("data: ")) {
          try { data = JSON.parse(line.slice(6)) } catch {}
        }
      }
      if (!data) continue

      const type = data.type
      const payload = data.payload ?? {}

      // Read progress_pct directly from payload (authoritative from backend)
      const pct = payload.progress_pct

      switch (type) {
        case "turn_started":
          onProgress?.(pct ?? 5, "Analyzing your request...")
          break

        case "thinking_started":
          onProgress?.(pct ?? 10, "Thinking...")
          break

        case "analyzer_done":
          onProgress?.(pct ?? 25, "Understanding your vision...")
          break

        case "section_started":
        case "section_done":
          // Optional — intermediate progress during parallel generation
          if (pct != null) onProgress?.(pct, "Generating content...")
          break

        case "generator_done":
          onProgress?.(pct ?? 55, "Creating your moodboard...")
          break

        case "enrichment_started":
          onProgress?.(pct ?? 58, "Finding images...")
          break

        case "artifact_enriched":
          artifactCount++
          onArtifact?.(payload)  // render card immediately
          onProgress?.(pct ?? 75, `Loading images (${artifactCount})...`)
          break

        case "enrichment_progress":
          // Aggregate: payload.resolved / payload.total
          if (pct != null) onProgress?.(pct, `Images ${payload.resolved}/${payload.total}`)
          break

        case "validator_done":
          onProgress?.(pct ?? 95, "Finalizing...")
          break

        case "snapshot_saved":
          // Session auto-saved — no user-visible action needed
          break

        case "turn_complete":
          onProgress?.(pct ?? 100, "Done")
          onComplete?.(payload.result)
          break

        case "turn_error":
          onError?.(payload.error)
          break

        case "heartbeat":
          // Keep-alive — no action needed, don't update progress
          break
      }
    }
  }
}
```

---

## Stream 2: Regen + Out-of-Band (GET `/v2/opal/assist/stream/{session_id}`)

### How to connect

```js
const source = new EventSource(`/v2/opal/assist/stream/${sessionId}?timeout_seconds=90`)
source.onmessage = (e) => {
  const msg = JSON.parse(e.data)
  // route on msg.type
}
```

Use `SessionStreamManager` (see `session_stream_manager.ts`) for auto-reconnect + singleton management.

### Wire format

```
id: sess-abc:1
data: {"plan_id":"sess-abc","type":"image_ready","payload":{...},"ts":"2026-03-24T..."}

```

**No `event:` line** — always use `source.onmessage`, NEVER `source.addEventListener("image_ready", ...)`.

### Events on this stream

| `type` | Source | Key `payload` fields |
|--------|--------|---------------------|
| `regenerate_started` | Regen POST | `artifact_id`, `label`, `session_id` |
| `image_ready` | Regen safety patch OR R2 callback | `artifact_id`, `label`, `category`, `image_url`, `url_type` ("cdn"/"r2"), `is_regen` (true/false) |
| `artifact_enriched` | Regen safety patch OR R2 callback (regen only) | `artifact_id`, `label`, `category`, `image_url`, `image_source`, `prompt_hash`, `progress_pct` (100), `is_regen` (true), `url_type` |
| `heartbeat` | Idle timeout about to fire | `session_id` |

### Regen FE handler

```js
// Using SessionStreamManager:
const mgr = new SessionStreamManager(sessionId)

mgr.on("regenerate_started", (payload) => {
  showSkeleton(payload.artifact_id)
})

// Option A: listen for artifact_enriched (recommended — same event as first-gen)
mgr.on("artifact_enriched", (payload) => {
  if (!payload.is_regen) return  // first-gen on POST stream, ignore here
  updateImage(payload.artifact_id, payload.image_url)
  if (payload.url_type === "r2") {
    // Permanent URL — regen fully complete
    hideSkeleton(payload.artifact_id)
  }
})

// Option B: listen for image_ready (legacy, still emitted for backward compat)
mgr.on("image_ready", (payload) => {
  if (!payload.is_regen) return  // first-gen R2 upgrade, skip
  updateImage(payload.artifact_id, payload.image_url)
  if (payload.url_type === "r2") {
    hideSkeleton(payload.artifact_id)
  }
})
```

---

## Common Mistakes

### 1. Using EventSource for chat turns
**Wrong:** `new EventSource("/v2/opal/assist/stream")` — this is a POST endpoint, EventSource only does GET.
**Right:** `fetch("/v2/opal/assist/stream", { method: "POST", ... })` + ReadableStream.

### 2. Using fetch for regen events
**Wrong:** Polling or waiting for the regen POST response to contain the image.
**Right:** Open `EventSource` on GET stream BEFORE posting regen, listen for `image_ready`.

### 3. Using `addEventListener` on the GET stream
**Wrong:** `source.addEventListener("image_ready", handler)` — never fires because there's no `event:` line.
**Right:** `source.onmessage = (e) => { const msg = JSON.parse(e.data); if (msg.type === "image_ready") ... }`

### 4. Not reopening the GET stream before regen
**Wrong:** Assuming the GET stream from the previous chat turn is still alive.
**Right:** Call `mgr.ensureConnected()` before every regen POST.

### 5. Expecting progress % from the GET stream
**Wrong:** Waiting for progress events on the GET stream during chat turns.
**Right:** Progress events only come from the POST stream (chat turns). The GET stream never emits analyzer_done, generator_done, etc.

---

## Architecture Diagram

```
FE sends chat message
        │
        ▼
POST /v2/opal/assist/stream ──── fetch() + ReadableStream
        │
        ├── turn_started        ──▶ FE shows 5%
        ├── thinking_started    ──▶ FE shows 10%
        ├── analyzer_done       ──▶ FE shows 25%
        ├── section_started     ──▶ FE shows 30%  (0-N per section)
        ├── section_done        ──▶ FE shows 50%  (0-N per section)
        ├── generator_done      ──▶ FE shows 55%
        ├── enrichment_started  ──▶ FE shows 58%
        ├── artifact_enriched   ──▶ FE shows 60-90%, renders card
        ├── enrichment_progress ──▶ (aggregate: resolved/total)
        ├── validator_done      ──▶ FE shows 95%
        ├── snapshot_saved      ──▶ FE shows 98%  (session persisted)
        └── turn_complete       ──▶ FE shows 100%, full result
                │
                │  (images may also fire _after_r2_ready callbacks)
                ▼
        Redis channel: kyra:v2:turn:{session_id}
                │
                ▼
GET /v2/opal/assist/stream/{session_id} ──── EventSource
        │
        ├── image_ready (is_regen=false, url_type=r2)  ──▶ FE upgrades CDN→R2 URL
        └── (idle 60s) ──▶ stream closes, FE auto-reconnects


FE clicks "Regenerate"
        │
        ├── FE calls mgr.ensureConnected()  ──▶ EventSource opens/reopens
        │
        ▼
POST /v2/opal/sessions/{sessionId}/artifact-feedback
        │
        ├── 200 OK (immediate)
        │
        │  (background task runs)
        ▼
Redis channel: kyra:v2:turn:{session_id}
        │
        ▼
GET /v2/opal/assist/stream/{session_id} ──── EventSource
        │
        ├── regenerate_started               ──▶ FE shows skeleton
        ├── image_ready (is_regen=true, url_type=cdn)  ──▶ FE shows CDN image
        └── image_ready (is_regen=true, url_type=r2)   ──▶ FE upgrades to R2 URL
```
