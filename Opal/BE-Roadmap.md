# Festimo — Backend Roadmap

> Implementation plan with live codebase type references.
> Existing types marked `[existing]`. New types marked `[new]`.
> `→ blocks` shows what cannot start until a feature ships.

---

## Dependency Chain

```
P0-A  Blueprint URL + publicSlug
  └→  P0-B  Blueprint Drop (publish + OG card)
  └→  P0-C  Guest View endpoint
  └→  P0-D  completionScore on HomeMoodboardViewModel
  └→  P1-D  viewCount + inspired attribution
  └→  P3-A  BlueprintIndex / Discovery Feed
  └→  P3-B  Fork endpoint

P1-A  UserTasteProfile model
  └→  P1-B  Signal extraction pipeline
  └→  P1-C  TasteContextBuilder → system prompt injection
  └→  P1-D  EventType sub-profiles
  └→  P2-B  Proactive Blueprint preparation

P2-A  CelebrationDate model
  └→  P2-B  Proactive Blueprint scheduled job
  └→  P2-C  Urgency tier injection into system prompt
  └→  P2-D  Celebration Recap

P3-C  AestheticMovement classification
  └→  P3-D  Trending movements table + endpoint
```

---

## Phase Status

| Phase | Name | Status |
|-------|------|--------|
| **P0** | Blueprint Foundation | ✅ Complete |
| **P1** | Taste Memory Engine | ✅ Complete |
| **P2** | Anticipatory AI | ✅ Complete |
| **P3** | Celebration Network | 🔴 Not started |
| **P4** | Vendor Intelligence | 🔴 Not started |
| **P5** | Memory OS | 🔴 Not started |

---

## ✅ Phase P0 — Blueprint Foundation

### P0-A · Blueprint URL — publicSlug + isPublic + viewCount

**Commits:** `11a6a05` — public slug, view tracking, sharing endpoints (#125)

**New fields on session:**

```typescript
publicSlug:   string;        // "golden-maple-7f3a"
isPublic:     boolean;       // default false
viewCount:    number;        // deduplicated unique views
publishedAt:  string | null; // ISO timestamp
```

**Endpoints:**
```
GET  /api/b/{slug}           → Public Blueprint (no auth)
POST /api/b/{slug}/view      → Record deduplicated view (no auth)
POST /api/b/{slug}/publish   → Toggle publish state (auth required)
```

---

### P0-B · OG Share Card Generation on Publish

**Commits:** `511cd8a` (#126)

Generated async on publish. `share_card_url` is null immediately after `POST /publish` — poll `GET /api/b/{slug}` until non-null (~2–4s).

**Dimensions:** 1200×630 PNG — standard OG image.

---

### P0-C · Guest View — Privacy-Safe Public Endpoint

**Commits:** `9758aa2` (#127)

`GET /api/b/{slug}` returns `BlueprintResponse` — deliberately excludes: budget, guestCount, vendorNotes, workspaceNotes, posterUrl, planSections detail, insights, entities, chat_turns.

---

### P0-D · completionScore on Session Save

**Commits:** `2b41473` (#128)

`plan_completion_pct` (0–100) auto-calculated on every session save. Counts sections with ≥1 pinned artifact as "complete".

---

## ✅ Phase P1 — Taste Memory Engine

### P1-A · UserTasteProfile Model + Store + API

**Commits:** `3bbed5d` (#129)

```
GET /api/user/taste-profile         → Full profile (auth)
GET /api/user/taste-profile/public  → Public profile, no budget (no auth)
```

Returns **204** if user has no finalized sessions yet.

---

### P1-B · Signal Extraction Pipeline

**Commits:** `1aafdc2` (#130)

Triggered automatically when a session is finalized. Extracts aesthetic signals from entities + artifact categories → merges into UserTasteProfile.

---

### P1-C · TasteContextBuilder — Prompt Injection

**Commits:** `f63218a` (#131)

For returning users, taste profile is injected into Kyra's system prompt at turn 0. FE doesn't need to pass it — it's automatic.

---

### P1-D · InspirationAttribution + View Milestone Notifications

**Commits:** `2fea4b2` (#132)

```
POST /api/b/{slug}/inspired                → Record attribution (no auth)
GET  /api/user/notifications               → Fetch unread (auth)
PATCH /api/user/notifications/{id}/read   → Mark read (auth)
```

Notifications fire at: 50, 100, 250, 500, 1000 views.

---

## ✅ Phase P2 — Anticipatory AI

### P2-A · CelebrationDate Model + CRUD + Auto-Seed

**Commits:** `97f1bb6`

```
POST   /api/calendar/dates          → Create
GET    /api/calendar/upcoming       → 180-day window
PATCH  /api/calendar/dates/{id}     → Update
DELETE /api/calendar/dates/{id}     → Soft-delete
```

Auto-seeded on Blueprint finalize when `date_or_time` entity exists.

---

### P2-B · Proactive Blueprint Scheduled Job

**Commits:** `e1986c3`

Daily scheduler checks CelebrationDates within 90 days → auto-generates DraftBlueprint.

```
POST /api/calendar/blueprints/{slug}/approve  → Promotes to workspace
POST /api/calendar/blueprints/{slug}/dismiss  → Soft-deletes + suppresses future
```

---

### P2-C · Urgency Intelligence — Time-Aware Prompting

**Commits:** `2f68eb8`

Injects urgency context into Kyra's prompt based on linked CelebrationDate proximity. Three tiers: `low` (30+ days) → `medium` (14–30 days) → `high` (<14 days).

---

### P2-D · Celebration Recap

**Commits:** `89c3621`

```
POST /api/b/{slug}/recap          → Submit recap (rating, notes, photos, vendors)
POST /api/b/{slug}/recap/photos   → Upload photos first, get CDN URLs back
```

Transitions `lifecycle_stage` → `'memory'`. Triggers taste profile update.

---

## 🔴 Phase P3 — Celebration Network (not started)

- P3-A: Public Blueprint Discovery Feed
- P3-B: Fork Mechanic — Start From Blueprint
- P3-C: Aesthetic Movement AI Classification
- P3-D: Trending Aesthetic Feed
- P3-E: Guest Contribution — Reactions & Voting

---

## 🔴 Phase P4 — Vendor Intelligence (not started)

- P4-A: AI Vendor Matching Engine
- P4-B: AI-Generated RFQ Templates
- P4-C: Festimo Verified — Quality Scoring
- P4-D: Vendor Booking Tracking in Plan Sections

---

## 🔴 Phase P5 — Memory OS (not started)

- P5-A: Celebration Memory Archive
- P5-B: Year in Celebrations — Annual Recap
- P5-C: Celebration Intelligence Dashboard
- P5-D: Day-of Mode — Execution Intelligence
