# Festimo — Frontend Integration Guide

> **Single source of truth** for all FE integration work on the Festimo backend.
> Covers P0 · P1 · P2 + LLM Intelligence milestone.
> Branch: `opal/dev-stage` · April 2026

---

## Table of Contents

1. [Session Lifecycle](#1-session-lifecycle)
2. [Core Assist — Chat & SSE Stream](#2-core-assist--chat--sse-stream)
3. [Moodboard](#3-moodboard)
4. [Plan Tab](#4-plan-tab)
5. [Blueprint Sharing (P0)](#5-blueprint-sharing-p0)
6. [Fork Flow](#6-fork-flow)
7. [Inspiration Feed](#7-inspiration-feed)
8. [Taste Profile (P1)](#8-taste-profile-p1)
9. [Notifications (P1)](#9-notifications-p1)
10. [Celebration Calendar (P2-A)](#10-celebration-calendar-p2-a)
11. [Proactive Blueprints (P2-B)](#11-proactive-blueprints-p2-b)
12. [Celebration Recap (P2-D)](#12-celebration-recap-p2-d)
13. [LLM Intelligence — FE Surface Changes](#13-llm-intelligence--fe-surface-changes-149154)
14. [TypeScript Types Reference](#14-typescript-types-reference)
15. [Migration Checklist](#15-migration-checklist)

---

## 1. Session Lifecycle

Every session has a `lifecycle_stage` field returned in all API responses. FE must read it to restore correct tab state.

| Stage | When Set | UI State |
|-------|----------|----------|
| `moodboard` | Default on create | Exploring + pinning artifacts |
| `planning` | Auto-set when first plan builds (turn ≥ 3, artifacts ≥ 3) | Reviewing plan sections |
| `celebrate` | Explicit via `POST /finalize` | Shopping + vendor sourcing |
| `memory` | After recap is submitted | Read-only archive (**new**) |

### Tab Enable Logic

```typescript
const tabs = {
  moodboard: true,                                          // always enabled
  plan:      plan_ready || lifecycle_stage !== 'moodboard',
  celebrate: lifecycle_stage === 'celebrate',
  memory:    lifecycle_stage === 'memory',
};
```

---

## 2. Core Assist — Chat & SSE Stream

### `POST /v2/opal/assist`

The primary endpoint. Streams 12 events over SSE. All intelligence (questions, artifacts, plan, next-action) arrives here.

### SSE Event Chain

| Event | Progress % | Key Payload Fields |
|-------|------------|--------------------|
| `turn_started` | 5% | `session_id`, `request_id` |
| `thinking_started` | 10% | `intent_mode: string \| null` |
| `analyzer_done` | 25% | `intent_mode`, `conversation_stage`, `injection_detected` |
| `section_started` | 30% | `section_id`, `section_title` (0–N per section) |
| `section_done` | 50% | `section_id` |
| `generator_done` | 55% | `has_plan_update: bool` |
| `enrichment_started` | 58% | `total_artifacts: int` |
| `artifact_enriched` | 60–90% | `artifact_id`, `label`, `category`, `image_url`, `is_regen`, `url_type` |
| `enrichment_progress` | 60–90% | `resolved`, `total`, `pct` |
| `validator_done` | 95% | `validator_pass: bool` |
| `snapshot_saved` | 98% | _(no visible action needed)_ |
| `turn_complete` | 100% | `lifecycle_stage`, `plan_ready`, `party_plan` |

> 🆕 **New (#149/#151):** `analyzer_done` now emits `conversation_stage` (`'discovery' | 'narrowing' | 'execution'`) and `injection_detected: bool`.

### SSE Envelope

Every event includes:

```typescript
{ type, session_id, request_id, payload, progress_pct, ts }
```

`progress_pct` mirrors `payload.progress_pct`. `ts` is ISO-8601 UTC.

### Injection Fast-Exit (#151)

If the BE detects a prompt-injection attack (`injection_detected: true`), it skips the LLM entirely and returns a safety response immediately. Handle this:

```typescript
if (event.type === 'analyzer_done' && event.payload.injection_detected) {
  showSafetyBanner('I cannot process that request.');
  cancelStream();
}
```

### `artifact_enriched` — Unified Payload

```typescript
interface ArtifactEnrichedPayload {
  artifact_id: string;
  label: string;
  category: string;
  image_url: string;
  image_credit: string;
  prompt_hash: string;
  progress_pct: number;
  is_regen: boolean;    // false on chat turn, true on regen
  url_type: 'cdn' | 'r2';
}
```

---

## 3. Moodboard

### MoodboardItem Type

```typescript
interface MoodboardItem {
  artifact_id: string;           // guaranteed non-empty (UUID fallback)
  label: string;
  category: string;
  image_url: string | null;       // active display URL
  image_source: 'generated' | 'search' | 'none' | 'regenerate_queued';
  r2_url: string | null;          // permanent; null until R2 upload done
  cdn_url: string | null;         // short TTL ~7 days
  visual_query: string | null;    // image subject description
  prompt_hash: string | null;     // SHA-256 content-address
  turn_number: number;
  added_at: string;

  // Interaction signals — feed the AI feedback loop
  thumbs: -1 | 0 | 1;            // +1 liked, -1 disliked, 0 no vote
  regen_count: number;
  downloaded: boolean;
  shared: boolean;
  saved: boolean;
  is_pinned: boolean;

  // Provenance
  generation_model: string | null;
  generation_provider: string;    // 'runware' | 'unsplash' | 'none'
  enrichment_latency_ms: number | null;
  position_in_turn: number;
  confidence: number;             // 0.0–1.0
  preferred_type: 'search' | 'generation' | 'any';
}
```

> 🆕 **Feedback loop (#149):** `thumbs_up` / `thumbs_down` directly steers AI question generation. Pin/like → AI asks refining questions toward that aesthetic. Delete/dislike → AI avoids that style. **Wire all artifact interaction events** to the artifact-feedback endpoint.

### BoardTextCard

Crisp tip/insight cards shown alongside images in the moodboard grid (Pinterest-style text pins).

```typescript
interface BoardTextCard {
  card_id: string;           // stable 12-char hex key
  card_type: 'tip' | 'insight' | 'checklist';
  title: string;             // ≤ 60 chars
  body: string;              // 1–2 sentences
  bullets: string[];         // ≤ 4 items
  tags: string[];            // e.g. ['budget', 'save']
  section: string;           // 'decor' | 'food' | 'venue' | 'budget' | 'general'
  turn_number: number;
  added_at: string;
}
```

| `card_type` | Icon | Content Pattern |
|-------------|------|-----------------|
| `tip` | Lightbulb | Single actionable suggestion |
| `insight` | Sparkle | Context-aware observation |
| `checklist` | Checkbox | `bullets[]` is primary content, `body` is empty |

### Composite Poster

Auto-generated 4:3 collage (1600×1200 WEBP) from session's best artifacts. Triggers when ≥ 4 images exist.

| Field | Type | Notes |
|-------|------|-------|
| `poster_url` | `string \| null` | R2 URL; null until first generation |
| `poster_version` | `string \| null` | Use as `<img key>` for cache-busting |
| `poster_artifact_count` | `number` | How many artifacts are in current poster |

- Listen for SSE `moodboard_poster_ready` to update in real-time
- Display at `aspect-ratio: 4/3; object-fit: contain`
- `POST /v2/opal/sessions/{id}/poster` to force regeneration on existing sessions

### Artifact Feedback

**`POST /v2/opal/sessions/{session_id}/artifact-feedback`**

```typescript
{ artifact_label: string; action: FeedbackAction }
```

| Action | Effect |
|--------|--------|
| `thumbs_up` | Sets `thumbs = +1` — steers AI toward this aesthetic |
| `thumbs_down` | Sets `thumbs = -1` — steers AI away from this style |
| `pin` | `is_pinned = true` — style anchor |
| `unpin` | Removes pin |
| `delete` | Rejects artifact |
| `regenerate` | Re-generates image (`image_source → 'regenerate_queued'`) |
| `download` | Sets `downloaded = true` |
| `share` | Sets `shared = true` |
| `save` | Sets `saved = true` |

---

## 4. Plan Tab

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/v2/opal/sessions/{id}/progress` | Section-level plan completion | Optional |
| `POST` | `/v2/opal/sessions/{id}/finalize` | Lock plan → celebrate stage | Yes |
| `POST` | `/v2/opal/sessions/{id}/unfinalize` | Unlock plan → planning stage | Yes |
| `GET` | `/v2/opal/sessions/{id}/shopping-list` | Extract shopping list | Optional |

### Progress Response

```json
{
  "session_id": "...",
  "lifecycle_stage": "planning",
  "overall_percent": 43,
  "sections": [{ "section_id": "bar_drinks", "title": "Bar & Drinks", "status": "complete", "item_count": 2, "has_pinned": true }],
  "completed_count": 3,
  "in_progress_count": 2,
  "empty_count": 2,
  "total_sections": 7,
  "next_suggestion": "Consider adding: Cake & Desserts, Venue."
}
```

| Status | UI | Condition | Guidance |
|--------|-----|-----------|---------|
| `empty` | Grey / dashed | No items | "Add items" CTA |
| `in_progress` | Yellow | Items exist, none pinned | "Pin to confirm" hint |
| `complete` | Green + check | ≥ 1 pinned | Confirmed |

### Section IDs Reference

| `section_id` | Title | Categories Routed Here |
|--------------|-------|----------------------|
| `bar_drinks` | Bar & Drinks | `cocktail` |
| `food_catering` | Food & Catering | `food` |
| `cake_desserts` | Cake & Desserts | `cake` |
| `flowers_decor` | Flowers & Decor | `floral`, `decor`, `table_setting` |
| `venue` | Venue | `venue` |
| `theme_styling` | Theme & Styling | `theme`, `other` |
| `attire` | Attire | `outfit` |

---

## 5. Blueprint Sharing (P0)

All Blueprint endpoints live under `/api/b/`. Public reads require no auth.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/b/{slug}` | Fetch published Blueprint | No |
| `POST` | `/api/b/{slug}/view` | Record deduplicated view | No |
| `POST` | `/api/b/{slug}/publish` | Publish or unpublish (toggle) | Yes |
| `POST` | `/api/b/{slug}/inspired` | Record inspiration attribution | No |

### `BlueprintResponse` (GET `/api/b/{slug}`)

| Field | Type | Description |
|-------|------|-------------|
| `public_slug` | `string` | URL slug e.g. `'golden-maple-7f3a'` |
| `event_type` | `string \| null` | e.g. `'birthday'`, `'wedding'` |
| `title` | `string \| null` | Blueprint title |
| `aesthetic_tags` | `string[]` | Derived from moodboard categories |
| `moodboard_images` | `PublicArtifactImage[]` | First 6 displayable images |
| `plan_completion_pct` | `int (0–100)` | Completion score (#128) |
| `view_count` | `int` | Deduplicated unique views |
| `share_card_url` | `string \| null` | OG share card 1200×630 PNG |
| `color_palette` | `object \| null` | `{primary: {hex, name}, ...}` |
| `thumbnail_url` | `string \| null` | Cover image |
| `published_at` | `string \| null` | ISO timestamp |

### OG Share Card — Important

`share_card_url` is **null immediately after publish** — the card generates async (~2–4s). Poll `GET /api/b/{slug}` until non-null before showing share sheet.

### Publish Toggle

```typescript
// POST /api/b/{slug}/publish
{ publish: true }   // publish
{ publish: false }  // unpublish
// Response: { is_public, published_at, public_slug, share_card_url }
```

### Inspiration Attribution

Call `POST /api/b/{slug}/inspired` when a guest starts a new session inspired by a Blueprint. Pass `{ new_session_id: string }` in the request body.

---

## 6. Fork Flow — Start My Own Version

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/v2/opal/sessions/explore` | Public Blueprint grid (paginated) | No |
| `GET` | `/v2/opal/sessions/{id}` | Full session snapshot | No |
| `POST` | `/v2/opal/sessions/{id}/fork` | Fork a Blueprint | Optional |

### Owner vs. Guest Detection

```typescript
const isOwner = snapshot.user_id === currentUser?.uid;
if (isOwner) renderEditableSession(snapshot);
else { renderReadOnlySession(snapshot); showForkCTA(); }
```

### Fork Response (201)

```json
{ "session_id": "new-uuid", "parent_session_id": "source-id" }
```

### Post-Fork State on `GET /sessions/{new_id}`

| Field | Value | Notes |
|-------|-------|-------|
| `lifecycle_stage` | `'moodboard'` | Always starts here |
| `party_plan` | `null` | Auto-builds after interaction |
| `chat_turns` | `[]` | Fresh conversation |
| `moodboard` | Copied from source | All artifacts present |
| `entities` | Copied from source | occasion, theme, etc. |
| `title` | `'Fork: {source title}'` | Editable via `PATCH` |
| `version_label` | `'v2'` (or v3, v4…) | Increments per fork in chain |
| `parent_session_id` | Source session_id | Immediate parent reference |

---

## 7. Inspiration Feed

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/v2/opal/inspiration` | Browse feed (paginated) | No |
| `GET` | `/v2/opal/inspiration/categories` | Category list with counts | No |
| `GET` | `/v2/opal/inspiration/{artifact_id}` | Artifact detail + view++ | No |
| `POST` | `/v2/opal/inspiration/{artifact_id}/save` | Save / bookmark | No |
| `DELETE` | `/v2/opal/inspiration/{artifact_id}/save` | Unsave | No |
| `POST` | `/v2/opal/inspiration/{artifact_id}/add-to-board` | Add to moodboard (10× trending) | No |

### Feed Query Params

| Param | Type | Description |
|-------|------|-------------|
| `category` | string | `decor` `food` `floral` `cake` `cocktail` `venue` `outfit` `favor` `lighting` `table` `invitation` `photo` |
| `tags` | string | Comma-separated style tags (max 10): `bohemian,tropical` |
| `sort` | string | `trending` (default) \| `newest` \| `popular` |
| `limit` | int | 1–50 per page (default 20) |
| `cursor` | string | Opaque cursor from previous response |

### Image Loading — 3-Tier Strategy

| Layer | Source | When |
|-------|--------|------|
| 1 | `dominant_color` | CSS background-color on card — instant (0ms) |
| 2 | `lqip` (base64 WebP) | Blurred `<img>` layer — ~50ms |
| 3 | `thumb` (300px) | Lazy-loaded WebP — 200–500ms |
| 4 | `medium` (800px) | Load on detail open only |
| 5 | `full` (1600px) | Only from detail endpoint — **never in grid** |

```tsx
<div style={{ backgroundColor: dominant_color, aspectRatio: aspect_ratio }}>
  <img src={lqip} style={{ filter: 'blur(20px)', position: 'absolute', inset: 0 }} />
  <img src={thumb} loading="lazy" />
</div>
```

Use `aspect_ratio` to pre-set card height — eliminates CLS entirely.

### Trending Weights

- views: **1×**
- saves: **5×**
- add-to-board: **10×** ← always call this when user adds to moodboard

---

## 8. Taste Profile (P1)

Accumulated aesthetic preferences across all finalized sessions. Extracted automatically on `finalize`. Injected into Kyra's prompt at turn 0 for returning users — FE doesn't pass it, just displays it.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/user/taste-profile` | Full profile (includes budget) | Yes |
| `GET` | `/api/user/taste-profile/public` | Public profile (no budget) | No |

Returns **204 No Content** if the user has never finalized a session.

### Key Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `profile_version` | `int` | Monotonically increasing — use as SWR cache key |
| `total_events_planned` | `int` | Finalized sessions that fed this profile |
| `global_profile` | `object` | Aggregated taste across all event types |
| `event_type_profiles` | `object` | Per-event-type: `{ birthday: TasteSubProfile }` |
| `budget_pattern` | `object` | `avg`, `min`, `max`, `distribution` across sessions |
| `snapshots` | `array` | Per-session extraction history (max 20, newest last) |

---

## 9. Notifications (P1)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/user/notifications` | Fetch unread notifications | Yes |
| `PATCH` | `/api/user/notifications/{id}/read` | Mark as read | Yes |

```typescript
interface NotificationResponse {
  id: string;
  type: 'view_milestone' | 'inspiration_attribution';
  blueprint_slug: string;
  data: Record<string, unknown>;  // milestone: view_count | attribution: new_session_id
  created_at: string;
  is_read: boolean;
}
```

Two notification types:
- **`view_milestone`** — Blueprint hit 50, 100, 250, 500, or 1000 views
- **`inspiration_attribution`** — A guest started a session inspired by your Blueprint

---

## 10. Celebration Calendar (P2-A)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `POST` | `/api/calendar/dates` | Create a celebration date | Yes |
| `GET` | `/api/calendar/upcoming` | Upcoming dates (180-day window) | Yes |
| `PATCH` | `/api/calendar/dates/{id}` | Update a date | Yes |
| `DELETE` | `/api/calendar/dates/{id}` | Soft-delete a date | Yes |

### CelebrationDate Fields

| Field | Type | Description |
|-------|------|-------------|
| `label` | `string` | e.g. `"Mum's birthday"` |
| `event_type` | `string` | `birthday` \| `anniversary` \| `wedding` \| `graduation` \| `other` |
| `month` | `int (1–12)` | Month of occurrence |
| `day` | `int (1–31)` | Day of occurrence |
| `year` | `int \| null` | `null` = recurring annually |
| `is_recurring` | `bool` | `true` = auto-rolls to next year when past |
| `next_occurrence` | `string (YYYY-MM-DD)` | Calculated by BE — do not compute client-side |
| `source` | `string` | `'manual'` \| `'blueprint_extract'` |
| `proactive_dismissed` | `bool` | User dismissed proactive Blueprint suggestion for this date |

Dates are **auto-seeded** when a Blueprint is finalized with a `date_or_time` entity (`source: 'blueprint_extract'`).

---

## 11. Proactive Blueprints (P2-B) 🆕

The backend scheduler checks upcoming CelebrationDates (90-day window) and auto-generates draft Blueprints. FE shows these as suggestions.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `POST` | `/api/calendar/blueprints/{slug}/approve` | Approve draft — appears in workspace | Yes |
| `POST` | `/api/calendar/blueprints/{slug}/dismiss` | Dismiss draft + suppress future ones | Yes |

### `DraftBlueprintResponse`

```typescript
{
  id, user_id, session_id, blueprint_slug,
  celebration_date_id, is_draft, is_proactive,
  completion_score, created_at, updated_at
}
```

### UX Pattern

- Show draft Blueprints as a notification card or inbox item
- **Approve** → `POST .../approve` → `is_draft` becomes `false` → Blueprint appears in workspace
- **Dismiss** → `POST .../dismiss` → soft-deletes draft + marks `CelebrationDate.proactive_dismissed = true` (suppresses future auto-drafts for this date)

---

## 12. Celebration Recap (P2-D) 🆕

After an event, the user submits a recap. This transitions to `lifecycle_stage: 'memory'` and feeds the taste profile.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `POST` | `/api/b/{slug}/recap` | Submit recap | Yes |
| `POST` | `/api/b/{slug}/recap/photos` | Upload recap photos | Yes |

### RecapRequest Body

| Field | Type | Notes |
|-------|------|-------|
| `rating` | `int (1–5)` | Required — overall event rating |
| `notes` | `string \| null` | Free-text (max 5,000 chars) |
| `photo_urls` | `string[]` | CDN URLs from photo upload endpoint (max 10) |
| `actual_budget` | `float \| null` | Actual total spend |
| `vendor_ratings` | `array` | `[{ vendor_category, vendor_name?, rating 1-5, note? }]` |

**Upload photos first** via `POST .../recap/photos`, collect returned CDN URLs, include in recap submission.

### RecapResponse

```json
{ "recap_id": "...", "blueprint_slug": "...", "lifecycle_stage": "memory" }
```

After a successful recap: session enters `'memory'` stage (read-only archive). Taste extraction triggers automatically in the background.

---

## 13. LLM Intelligence — FE Surface Changes (#149–154)

### 13.1 Conversation Stage (#149)

`analyzer_done` SSE now includes `conversation_stage`:

| `conversation_stage` | AI Question Tone | Approx When |
|----------------------|-----------------|-------------|
| `discovery` | Broad / exploratory | Turn 0–4 |
| `narrowing` | Force a choice between 2–3 options | Turn 5–9 |
| `execution` | Logistics and timelines only | Turn 10+ |

Optionally use this to show a stage indicator: `"Exploring your vision"` → `"Narrowing options"` → `"Nailing the details"`.

### 13.2 Feedback Loop — Thumbs Drive Question Generation (#149)

**Most important FE-facing change.** Every thumbs_up, thumbs_down, pin, and reject action now directly influences what questions the AI asks next.

| User Action | AI Response |
|-------------|-------------|
| Pin / thumbs_up | AI steers ≥1 next question toward refining this aesthetic |
| Delete / thumbs_down | AI avoids this style direction entirely in future questions |

Wire **all** artifact interaction events to artifact-feedback — not just pin/delete.

### 13.3 Next Questions (#149 + #151)

`next_questions` now comes from a 56-candidate pool, filtered by:
- Conversation stage (discovery/narrowing/execution)
- Already-asked history (cross-turn deduplication)
- Known entities (won't ask for info already captured)

**Always render `next_questions` from the snapshot — never hardcode follow-up suggestions.**

### 13.4 Injection Safety Fast-Exit (#151/#152)

When the BE detects a prompt-injection attempt, `turn_complete` still fires but with a safety message. Check `injection_detected` from `analyzer_done` to show a user-friendly blocked-message UI rather than waiting for enrichment.

### 13.5 Sticky Facts (#149)

Critical entities (`occasion`, `guest_count`, `budget`, `venue`, `theme`) are now injected as `sticky_facts` into session context JSON. The AI will not forget confirmed details even across long sessions. No FE changes needed — FE benefit: fewer repetitive AI questions.

---

## 14. TypeScript Types Reference

> Place in `src/types/opal.ts`

```typescript
export type LifecycleStage = 'moodboard' | 'planning' | 'celebrate' | 'memory';
export type ConversationStage = 'discovery' | 'narrowing' | 'execution';
export type FeedbackAction =
  | 'thumbs_up' | 'thumbs_down' | 'pin' | 'unpin'
  | 'delete' | 'regenerate' | 'download' | 'share' | 'save';

export interface SessionSnapshot {
  session_id: string;
  version_id: string;
  turn_count: number;
  title: string;
  context_summary: string;
  lifecycle_stage: LifecycleStage;
  plan_ready: boolean;
  moodboard: MoodboardItem[];
  text_cards: BoardTextCard[];
  next_questions: string[];
  poster_url: string | null;
  poster_version: string | null;
  poster_artifact_count: number;
  chat_turns: ChatTurn[];
  entities: Record<string, unknown>;
  party_plan: PartyPlan | null;
  user_id: string | null;
  // Sharing (P0)
  public_slug: string;
  is_public: boolean;
  view_count: number;
  share_card_url: string | null;
  published_at: string | null;
  // Fork
  fork_count: number;
  parent_session_id: string | null;
  version_label: string;
  updated_at: string;
  created_at: string;
}

export interface MoodboardItem {
  artifact_id: string;
  label: string;
  category: string;
  image_url: string | null;
  image_source: 'generated' | 'search' | 'none' | 'regenerate_queued';
  r2_url: string | null;
  cdn_url: string | null;
  visual_query: string | null;
  prompt_hash: string | null;
  turn_number: number;
  added_at: string;
  thumbs: -1 | 0 | 1;
  regen_count: number;
  downloaded: boolean;
  shared: boolean;
  saved: boolean;
  is_pinned: boolean;
  generation_model: string | null;
  generation_provider: string;
  enrichment_latency_ms: number | null;
  position_in_turn: number;
  confidence: number;
  preferred_type: 'search' | 'generation' | 'any';
}

export interface BoardTextCard {
  card_id: string;
  card_type: 'tip' | 'insight' | 'checklist';
  title: string;
  body: string;
  bullets: string[];
  tags: string[];
  section: string;
  turn_number: number;
  added_at: string;
}

export interface NotificationResponse {
  id: string;
  type: 'view_milestone' | 'inspiration_attribution';
  blueprint_slug: string;
  data: Record<string, unknown>;
  created_at: string;
  is_read: boolean;
}
```

---

## 15. Migration Checklist

### Core Session & Assist

- [ ] Add `'memory'` to `LifecycleStage` type — new post-recap stage
- [ ] Read `conversation_stage` from `analyzer_done` SSE event
- [ ] Handle `injection_detected: true` in `analyzer_done` — show safety UI, cancel stream
- [ ] Wire **all** artifact interactions to artifact-feedback endpoint (thumbs, download, share, save — not just pin/delete)
- [ ] Always render `next_questions` from snapshot — never hardcode suggestions
- [ ] Handle `image_source: 'regenerate_queued'` — show skeleton

### Blueprint Sharing (P0)

- [ ] Add `public_slug`, `is_public`, `share_card_url`, `view_count` to `SessionSnapshot` type
- [ ] Implement `/api/b/{slug}` public page (no auth)
- [ ] Poll `share_card_url` after publish — async, null initially
- [ ] Call `POST /api/b/{slug}/view` on Blueprint page load

### P1 — Taste & Notifications

- [ ] Add taste profile display on profile page (handle 204 = first-timer)
- [ ] Add notification bell: `GET /api/user/notifications` + `PATCH .../{id}/read`

### P2 — Calendar, Proactive, Recap

- [ ] Add Calendar feature: CelebrationDate CRUD at `/api/calendar/dates`
- [ ] Show proactive Blueprint drafts as inbox cards with approve/dismiss CTAs
- [ ] Add Recap flow: photo upload → recap submission → `'memory'` stage
- [ ] Memory stage: all edit controls hidden — read-only archive

---

*Festimo · Confidential · April 2026 · branch: `opal/dev-stage`*
