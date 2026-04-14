# Vibe Atlas — Milestone Tracker

All 5 milestones complete. The Vibe Atlas DB layer is fully shipped.

---

## Summary

| Milestone | Name | Migration | Status |
|-----------|------|-----------|--------|
| M01 | Board Foundation | `0002_atlas_boards.sql` | ✅ Done |
| M01-BE | BE Contract Stabilization | — | ✅ Done |
| M02 | Asset Lifecycle Schema | `0003_atlas_assets.sql` | ✅ Done |
| M03 | Semantics + Vectors | `0004_atlas_semantics_variants.sql` | ✅ Done |
| M04 | Collaboration + RLS | `0005_atlas_collab_share.sql` | ✅ Done |
| M05 | Board-Plan-Vendor Integration | `0006_atlas_board_plan_vendor_links.sql` | ✅ Done |

---

## M01 — Board Foundation

**Branch:** `opal-vibe-atlas/m01-db-foundation`
**Migration:** `0002_atlas_boards.sql`

Establishes the core canvas structure for visual boards.

**Tables delivered:**

`boards` — top-level moodboard container per workspace. Fields: `id`, `workspace_id`, `title`, `description`, `owner_user_id`, `status` (`active` | `archived`), `current_version`, `metadata`, `created_at`, `updated_at`.

`board_versions` — immutable snapshot per board save. Each save increments `version_num`. Fields: `id`, `board_id`, `version_num`, `snapshot` (jsonb), `change_summary`, `created_by`, `created_at`.

`board_cards` — individual canvas elements. Fields: `id`, `board_id`, `card_type` (`image` | `text` | `swatch` | `link` | `note` | `shape`), spatial layout (`pos_x`, `pos_y`, `width`, `height`, `z_index`, `rotation_deg`), `content` (jsonb), `metadata`, timestamps.

**Indexes:** workspace lookup, owner lookup, updated_at sort, z-index sort for layering, GIN on metadata.

**Triggers:** `set_updated_at` on boards and board_cards.

---

## M01-BE — BE Contract Stabilization

**Branch:** `opal-vibe-atlas/m01-be-contract-stabilization`

Locked down the API surface for board operations before proceeding with deeper schema layers. Ensures Opal Server route contracts align with the M01 schema — no schema drift going into M02+.

---

## M02 — Asset Lifecycle Schema

**Branch:** `opal-vibe-atlas/m02-db-assets`
**Migration:** `0003_atlas_assets.sql`

Manages the full lifecycle of visual assets uploaded to or generated for boards.

**Tables delivered:**

`assets` — canonical asset record. Tracks `source_type` (`upload` | `url` | `generated`), `storage_provider` (`supabase` | `s3` | `gcs` | `external`), `storage_bucket`, `storage_path`, `mime_type`, `file_size_bytes`, dimensions (`width_px`, `height_px`), `sha256` dedup hash, `status` (`uploaded` → `processing` → `ready` → `failed` → `archived`).

`card_assets` — join table linking cards to assets. Supports role (`primary` | `reference` | `mask` | `texture`) and non-destructive transforms (`crop` jsonb, `transform` jsonb).

`asset_processing_events` — processing audit trail. Records each pipeline step: `event_type` (`ingest` | `transcode` | `analyze` | `tag_extract` | `thumbnail`), `processor`, `status`, `attempt`, `duration_ms`, `error_code`.

**Key constraints:**
- `(workspace_id, storage_bucket, storage_path)` unique — prevents duplicate uploads.
- Failed events require `error_code` (enforced by check constraint).
- SHA-256 must be exactly 64 chars if provided.

---

## M03 — Semantics + Vectors

**Branch:** `opal-vibe-atlas/m03-db-semantics`
**Migration:** `0004_atlas_semantics_variants.sql`

Enables aesthetic similarity search and AI-driven board variants.

**Tables delivered:**

`asset_tags` — taxonomy-based tagging of assets. `taxonomy` values: `auto` | `manual` | `style` | `material` | `space` | `color` | `brand`. `confidence` 0–1. `source`: `model` | `user` | `system`. Trigram index on `tag` for fuzzy search.

`board_semantics` — derived semantic profile per board version. Stores `mood_keywords[]`, `palette_keywords[]`, `semantic_profile` (jsonb), `confidence`, `derived_from` (`assets` | `user` | `hybrid`).

`mood_vectors` — pgvector embeddings. Supports `entity_type`: `asset` | `board` | `variant`. Default model: `text-embedding-3-small` (1536 dims). IVFFlat index with 100 lists for ANN search. Similarity threshold: cosine distance ≥ 0.85.

`board_variants` — AI-generated alternative board versions. Fields: `source_version_num`, `variant_rank` (1–20), `title`, `rationale`, `generator` (`model` | `user` | `hybrid`), `status` (`draft` | `accepted` | `rejected` | `archived`).

`board_variant_cards` — diff between variant and base board. Each row records an `operation` (`add` | `update` | `remove` | `reorder`) against a specific `board_card_id`.

**Extensions required:** `pgcrypto`, `pg_trgm`, `vector`.

---

## M04 — Collaboration + RLS

**Branch:** `opal-vibe-atlas/m04-db-collaboration`
**Migration:** `0005_atlas_collab_share.sql`

Multi-user board access with Row-Level Security policies.

**Tables delivered:**

`board_members` — per-board access roster. `role`: `owner` | `editor` | `commenter` | `viewer`. `membership_status`: `invited` | `active` | `revoked`. Tracks `invited_by`, `invited_at`, `joined_at`.

`board_comments` — threaded comments on boards or specific cards. `parent_comment_id` enables replies. `status`: `active` | `resolved` | `deleted`. Max 2000 chars per comment.

`board_shares` — tokenised share links. `share_token` (≥16 chars, unique), `access_role` (`viewer` | `commenter` | `editor`), `scope` (`link` | `invite`), `max_uses`, `use_count`, `expires_at`.

`activity_log` — append-only audit trail. `actor_type`: `user` | `system` | `integration`. `entity_type`: `board` | `card` | `asset` | `comment` | `share` | `member` | `variant` | `plan`. `visibility`: `workspace` | `board` | `private`.

**RLS policies:**

All four tables have RLS enabled. Access control rules:

`board_members` — owners can mutate; members can select their own row.

`board_comments` — active members (`owner` | `editor` | `commenter`) can write; any active member can read.

`board_shares` — `owner` and `editor` roles can create/update shares; active members can read.

`activity_log` — active members can insert (owner, editor, commenter roles); any active member can select.

---

## M05 — Board-Plan-Vendor Integration

**Branch:** `opal-vibe-atlas/m05-db-integration`
**Migration:** `0006_atlas_board_plan_vendor_links.sql`

Bridges aesthetic choices from Vibe Atlas boards into the Opal Server plan pipeline and vendor matching.

**Tables delivered:**

`board_plan_links` — links a Vibe Atlas board to an Opal Server plan. `link_type`: `primary` | `alternative` | `historical`. `link_status`: `draft` → `submitted` → `accepted` | `rejected` → `archived`. Unique on `(board_id, plan_id, link_type)`.

`vendor_briefs` — auto-generated vendor scoping docs derived from a board-plan link. `vendor_type`: `designer` | `contractor` | `supplier` | `stylist` | `photographer` | `other`. `brief_status`: `draft` → `sent` → `accepted` | `declined` → `closed`. Unique per `(board_plan_link_id, vendor_type)`.

`vendor_matches` — AI-ranked vendor suggestions per brief. `score` (0–1), `rank` (1–100), `match_status`: `suggested` → `shortlisted` → `accepted` | `rejected`. Unique on both `(vendor_brief_id, vendor_external_id)` and `(vendor_brief_id, rank)` to prevent duplicates and rank collisions.

**Views:**

`vw_board_plan_vendor_pipeline` — flat join of `board_plan_links → vendor_briefs → vendor_matches`. Useful for pipeline status dashboards.

`vw_vendor_match_leaderboard` — ranked vendor matches per brief using `ROW_NUMBER()` ordered by score descending. Use for "top N vendors" queries without application-side sorting.
