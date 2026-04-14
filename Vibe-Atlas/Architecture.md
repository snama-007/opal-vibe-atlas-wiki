# Vibe Atlas — Architecture & Design Principles

---

## Design Principles

### 1. Deterministic-first control
Core transitions and safety gates are policy-driven, not free-form. LLM provides structured recommendations; the policy engine owns final action selection.

### 2. Strict contracts everywhere
All LLM/tool outputs use JSON schemas. Invalid outputs trigger repair/fallback paths.

### 3. Cost-aware by default
Cheap model first. Premium only when justified by confidence or risk. Router inputs: ambiguity score, stage criticality, historical failure rate, current budget state.

### 4. State continuity over prompt replay
Keep compact session state + summary. Avoid full transcript replay each turn.

### 5. Progressive disclosure
Ask one high-value question per turn. Avoid interrogation-style multi-question dumps.

### 6. Observable and reversible
Every intelligence decision is logged and explainable. Rollout behind feature flags with clear rollback.

---

## DB Schema Layers

### Layer 1 — Board Foundation (M01)

Core tables for board and item management.

```sql
boards
  id, user_id, title, description, event_type,
  is_public, slug, view_count, fork_count,
  created_at, updated_at

board_items
  id, board_id, artifact_id, label, category,
  image_url, r2_url, position, is_pinned,
  thumbs, regen_count, downloaded, shared,
  generation_model, confidence, created_at
```

RLS: owners can read/write their own boards. Public boards (`is_public = true`) are readable by anyone.

---

### Layer 2 — Asset Lifecycle (M02)

Tracks the full lifecycle of visual artifacts.

```sql
asset_versions
  id, artifact_id, version, image_url, prompt_hash,
  generation_provider, generation_model,
  enrichment_latency_ms, status, created_at

asset_regen_log
  id, artifact_id, trigger, old_url, new_url,
  model_used, latency_ms, created_at
```

---

### Layer 3 — Semantics + Vectors (M03)

Enables aesthetic similarity search.

```sql
artifact_embeddings
  artifact_id, embedding vector(1536),
  style_tags, dominant_colors, aesthetic_score,
  embedding_model, created_at

aesthetic_variants
  id, source_artifact_id, variant_type,
  prompt_delta, image_url, similarity_score,
  created_at
```

Vector search via `pgvector`. Similarity threshold: 0.85 cosine distance.

---

### Layer 4 — Collaboration + RLS (M04)

Multi-user board access.

```sql
board_collaborators
  board_id, user_id, role ('viewer' | 'editor' | 'owner'),
  invited_by, accepted_at, created_at

board_activity_log
  id, board_id, user_id, action_type,
  item_id, metadata jsonb, created_at
```

RLS policies:
- `viewer` — read-only access to board and items
- `editor` — can add/edit items, cannot delete board or change collaborators
- `owner` — full access

---

### Layer 5 — Board-Plan-Vendor Integration (M05)

Links aesthetic choices to plan sections and vendor recommendations.

```sql
board_plan_links
  board_id, session_id, plan_section_id,
  linked_at, link_type ('auto' | 'manual')

vendor_artifact_links
  artifact_id, vendor_id, vendor_category,
  match_score, match_type, created_at

plan_section_artifacts
  section_id, artifact_id, position,
  is_primary, added_at
```

---

## Core Intelligence Patterns

### Pattern A — Policy-Governed Workflow
```
User input
    ↓
Deterministic Analyzer (fast, no LLM)
    ↓
[if injection detected] → fast-exit safety response
[if LLM needed] → LLM Analyzer
    ↓
reconcile_decision_with_analyzer()
    ↓
Switchboard → route to intent handler
    ↓
Generator → LLM response
    ↓
Validator → check output
    ↓
Enrichment → fetch images
```

### Pattern B — Occasion Slot Packs
Each intent mode has required + optional slots with extraction rules and confidence thresholds. Missing required slots → `missing_info_questions`.

### Pattern C — Next-Best-Question Ranking
Questions ranked by: information gain × stage relevance × user friction × confidence risk. One primary question selected per turn (56-candidate pool, cross-turn deduplicated).

### Pattern D — Model Routing
```python
if injection_detected: return deterministic_result  # free
if not should_run_llm_analyzer(): return deterministic_result  # free
# else: call LLM analyzer, merge with deterministic result
```

Cheap model first, premium only on escalation.
