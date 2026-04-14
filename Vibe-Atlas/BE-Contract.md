# Vibe Atlas — BE Contract

Data types and API contracts for Vibe Atlas. Vibe Atlas is accessed via Supabase's PostgREST auto-API and direct Postgres queries from Opal Server. There is no separate Vibe Atlas HTTP service — Opal Server owns all business logic and talks to the Supabase DB directly via the Supabase client.

---

## Access Pattern

```
Festimo Frontend
      │
      ▼
 Opal Server (FastAPI)
      │  ← all Vibe Atlas reads/writes go through here
      ▼
 Supabase (PostgREST + Postgres)
      ├── boards
      ├── board_cards / board_versions
      ├── assets / card_assets / asset_processing_events
      ├── asset_tags / board_semantics / mood_vectors / board_variants
      ├── board_members / board_comments / board_shares / activity_log
      └── board_plan_links / vendor_briefs / vendor_matches
```

Frontend never calls Supabase directly for Vibe Atlas data. All access is via Opal Server endpoints.

---

## Core Types

### Board

```typescript
interface Board {
  id: string;                    // UUID
  workspace_id: string;          // maps to Opal session workspace
  title: string;                 // 1–200 chars
  description: string | null;
  owner_user_id: string;
  status: 'active' | 'archived';
  current_version: number;       // increments on each save
  metadata: Record<string, unknown>;
  created_at: string;            // ISO-8601 UTC
  updated_at: string;
}
```

### BoardCard

```typescript
interface BoardCard {
  id: string;
  board_id: string;
  card_type: 'image' | 'text' | 'swatch' | 'link' | 'note' | 'shape';
  pos_x: number;
  pos_y: number;
  width: number;                 // > 0
  height: number;                // > 0
  z_index: number;
  rotation_deg: number;
  content: Record<string, unknown>;   // card-type-specific payload
  metadata: Record<string, unknown>;
  created_by: string | null;
  updated_by: string | null;
  created_at: string;
  updated_at: string;
}
```

**`content` shape by card_type:**

| card_type | Required content keys |
|-----------|----------------------|
| `image`   | `asset_id`, `alt_text?` |
| `text`    | `body`, `font_size?`, `align?` |
| `swatch`  | `hex`, `label?` |
| `link`    | `url`, `preview_title?` |
| `note`    | `body` |
| `shape`   | `shape_kind`, `fill_color?`, `stroke_color?` |

### Asset

```typescript
interface Asset {
  id: string;
  workspace_id: string;
  board_id: string | null;
  source_type: 'upload' | 'url' | 'generated';
  storage_provider: 'supabase' | 's3' | 'gcs' | 'external';
  storage_bucket: string;
  storage_path: string;
  original_filename: string | null;
  mime_type: string;
  file_size_bytes: number;
  width_px: number | null;
  height_px: number | null;
  sha256: string | null;        // 64-char hex
  status: 'uploaded' | 'processing' | 'ready' | 'failed' | 'archived';
  metadata: Record<string, unknown>;
  created_by: string;
  created_at: string;
  updated_at: string;
}
```

### MoodVector

```typescript
interface MoodVector {
  id: string;
  workspace_id: string;
  entity_type: 'asset' | 'board' | 'variant';
  entity_id: string;                 // UUID of the asset/board/variant
  embedding_model: string;           // default: 'text-embedding-3-small'
  embedding: number[];               // 1536-dim float array
  confidence: number;                // 0–1
  metadata: Record<string, unknown>;
  created_by: string | null;
  created_at: string;
}
```

### BoardMember

```typescript
interface BoardMember {
  id: string;
  board_id: string;
  user_id: string;
  role: 'owner' | 'editor' | 'commenter' | 'viewer';
  membership_status: 'invited' | 'active' | 'revoked';
  invited_by: string | null;
  invited_at: string | null;
  joined_at: string | null;
  metadata: Record<string, unknown>;
  created_at: string;
  updated_at: string;
}
```

### BoardShare

```typescript
interface BoardShare {
  id: string;
  board_id: string;
  created_by: string;
  share_token: string;          // ≥ 16 chars, unique
  access_role: 'viewer' | 'commenter' | 'editor';
  scope: 'link' | 'invite';
  is_active: boolean;
  expires_at: string | null;
  revoked_at: string | null;
  max_uses: number | null;
  use_count: number;
  metadata: Record<string, unknown>;
  created_at: string;
  updated_at: string;
}
```

### BoardPlanLink

```typescript
interface BoardPlanLink {
  id: string;
  board_id: string;             // references boards.id
  plan_id: string;              // references plans.id (Opal Server)
  link_type: 'primary' | 'alternative' | 'historical';
  link_status: 'draft' | 'submitted' | 'accepted' | 'rejected' | 'archived';
  created_by: string | null;
  metadata: Record<string, unknown>;
  created_at: string;
  updated_at: string;
}
```

### VendorBrief

```typescript
interface VendorBrief {
  id: string;
  board_plan_link_id: string;
  vendor_type: 'designer' | 'contractor' | 'supplier' | 'stylist' | 'photographer' | 'other';
  brief_status: 'draft' | 'sent' | 'accepted' | 'declined' | 'closed';
  title: string;                // 1–200 chars
  brief_payload: Record<string, unknown>;
  requirements: Record<string, unknown>;
  due_at: string | null;
  created_by: string | null;
  metadata: Record<string, unknown>;
  created_at: string;
  updated_at: string;
}
```

### VendorMatch

```typescript
interface VendorMatch {
  id: string;
  vendor_brief_id: string;
  vendor_external_id: string;
  score: number;                // 0–1 (5 decimal precision)
  rank: number;                 // 1–100
  match_status: 'suggested' | 'shortlisted' | 'accepted' | 'rejected';
  rationale: Record<string, unknown>;
  contact_payload: Record<string, unknown>;
  decided_by: string | null;
  decided_at: string | null;
  metadata: Record<string, unknown>;
  created_at: string;
  updated_at: string;
}
```

---

## Opal Server → Vibe Atlas Operations

These are the Supabase DB operations Opal Server performs. All go through the Supabase Python client.

### Boards

```
INSERT  boards                         → create board for a workspace session
SELECT  boards WHERE workspace_id = ?  → list boards for workspace
SELECT  boards WHERE id = ?            → get single board
UPDATE  boards SET ...                 → update title/description/status
INSERT  board_versions                 → snapshot on save (auto-increment version_num)
SELECT  board_versions WHERE board_id = ? ORDER BY version_num DESC LIMIT 1  → latest version
```

### Cards

```
INSERT  board_cards                    → add card to canvas
UPDATE  board_cards SET pos_x, pos_y, width, height, z_index, rotation_deg  → move/resize
UPDATE  board_cards SET content        → update card content
DELETE  board_cards WHERE id = ?       → remove card
SELECT  board_cards WHERE board_id = ? ORDER BY z_index  → fetch all cards for render
```

### Assets

```
INSERT  assets                         → register uploaded/generated asset
UPDATE  assets SET status = 'ready'    → after processing completes
SELECT  assets WHERE sha256 = ?        → deduplication check before upload
INSERT  card_assets                    → link asset to a card
INSERT  asset_processing_events        → log each processing step
SELECT  asset_processing_events WHERE asset_id = ?  → check processing status
```

### Semantic Search (M03)

```
INSERT  asset_tags                     → tag asset post-processing
INSERT  board_semantics                → store derived semantic profile
INSERT  mood_vectors (embedding = ?)   → store vector embedding
SELECT  mood_vectors WHERE entity_type = 'board'
  ORDER BY embedding <=> $query_vector  LIMIT 10
  -- cosine distance search via pgvector; filter to workspace_id
```

Similarity threshold: reject results where cosine distance > 0.15 (equivalent to cosine similarity < 0.85).

### Variants (M03)

```
INSERT  board_variants                 → create AI-generated alternative
INSERT  board_variant_cards            → store per-card diffs
UPDATE  board_variants SET status = 'accepted'  → apply variant
SELECT  board_variants WHERE board_id = ? AND status = 'draft'  → fetch pending variants
```

### Collaboration (M04)

```
INSERT  board_members (role='owner')   → on board creation
INSERT  board_members (role=?)         → invite collaborator
UPDATE  board_members SET membership_status = 'active'   → accept invite
UPDATE  board_members SET membership_status = 'revoked'  → remove access
SELECT  board_members WHERE user_id = ?  → boards I have access to
INSERT  board_comments                 → post comment
INSERT  board_shares                   → generate share link
SELECT  board_shares WHERE share_token = ?  → validate share link on access
INSERT  activity_log                   → log every mutation
SELECT  activity_log WHERE board_id = ? ORDER BY occurred_at DESC  → board activity feed
```

### Board-Plan Integration (M05)

```
INSERT  board_plan_links               → link board to Opal session plan
UPDATE  board_plan_links SET link_status = 'submitted'  → submit for review
INSERT  vendor_briefs                  → auto-generate vendor brief per vendor_type
UPDATE  vendor_briefs SET brief_status = 'sent'         → after sending to vendor
INSERT  vendor_matches                 → store AI vendor match results
UPDATE  vendor_matches SET match_status = 'shortlisted' → user shortlists a vendor
UPDATE  vendor_matches SET match_status = 'accepted'    → vendor selected
SELECT  vw_board_plan_vendor_pipeline WHERE board_id = ?  → full pipeline status
SELECT  vw_vendor_match_leaderboard WHERE vendor_brief_id = ?  → ranked vendors
```

---

## RLS Summary

| Table | Auth scope |
|-------|-----------|
| `boards` | No RLS — workspace_id filtering handled in application layer |
| `board_versions` | Inherits via board_id |
| `board_cards` | Inherits via board_id |
| `assets` | No RLS — workspace_id filtering in application layer |
| `asset_tags` | Inherits via asset_id |
| `board_semantics` | Inherits via board_id |
| `mood_vectors` | No RLS — workspace_id filtering in application layer |
| `board_variants` | Inherits via board_id |
| `board_members` | ✅ RLS — owners can mutate; members see own row |
| `board_comments` | ✅ RLS — active members read; owner/editor/commenter write |
| `board_shares` | ✅ RLS — owner/editor manage; active members read |
| `activity_log` | ✅ RLS — active members insert/select |
| `board_plan_links` | No RLS — Opal Server service role |
| `vendor_briefs` | No RLS — Opal Server service role |
| `vendor_matches` | No RLS — Opal Server service role |

Opal Server connects with the **service role key** for internal operations (bypasses RLS). Frontend-initiated operations use the **anon/user JWT** (RLS enforced).

---

## Error Handling

| Scenario | Behaviour |
|----------|-----------|
| Asset `sha256` already exists in workspace | Return existing asset ID — skip upload |
| Board `status = 'archived'` | Reject card mutations with 409 |
| `mood_vectors` query returns 0 results | Fall back to keyword-based tag search on `asset_tags` |
| `board_plan_links` with `plan_id` not in `plans` | FK violation → 409 from Opal Server before insert |
| Variant `status = 'accepted'` | Apply diff ops in `board_variant_cards` to produce new `board_versions` entry |
| Share link `is_active = false` or `expires_at` past | Return 403 with `share_expired` error code |
