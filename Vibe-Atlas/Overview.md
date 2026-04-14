# Vibe Atlas — Overview

Vibe Atlas is the intelligence and database layer powering Festimo's visual and aesthetic discovery engine. It sits alongside Opal Server and handles:

- **Board & asset management** — visual boards, moodboard items, artifact lifecycle
- **Semantic search** — vector embeddings for aesthetic similarity
- **Collaboration** — multi-user board access with Row-Level Security
- **Vendor & plan integration** — linking aesthetic choices to vendor recommendations and plan sections

---

## How It Fits

```
Festimo Frontend
       │
       ▼
  Opal Server  ←────────────────────────┐
  (FastAPI)                              │
       │                                │
       ├── LLM Pipeline (Kyra)          │
       ├── Session + Plan management    │
       └── Vibe Atlas API ─────────────▶│
                                         │
                              Vibe Atlas DB
                              (Supabase / Postgres)
                              ├── boards
                              ├── board_items (assets)
                              ├── embeddings (vectors)
                              ├── variants
                              ├── collaborators + RLS
                              └── vendor_links
```

---

## Milestone Map

| Milestone | Name | Branch | Status |
|-----------|------|--------|--------|
| M01 | Board Foundation | `opal-vibe-atlas/m01-db-foundation` | ✅ Done |
| M01-BE | BE Contract Stabilization | `opal-vibe-atlas/m01-be-contract-stabilization` | ✅ Done |
| M02 | Asset Lifecycle Schema | `opal-vibe-atlas/m02-db-assets` | ✅ Done |
| M03 | Semantics + Vectors | `opal-vibe-atlas/m03-db-semantics` | ✅ Done |
| M04 | Collaboration + RLS | `opal-vibe-atlas/m04-db-collaboration` | ✅ Done |
| M05 | Board-Plan-Vendor Integration | `opal-vibe-atlas/m05-db-integration` | ✅ Done |

All 5 milestones complete. Vibe Atlas DB layer is fully shipped.

---

## Tech Stack

| Layer | Stack |
|-------|-------|
| Database | Supabase (Postgres) |
| Vectors | pgvector extension |
| Auth / RLS | Supabase Auth + Row-Level Security policies |
| API | FastAPI (Opal Server routes to Vibe Atlas) |
| Migrations | Supabase CLI |
