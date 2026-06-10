# PRP: Sprint 4 — FAQ Knowledge Base & RAG

> **Version:** 1.0
> **Created:** 2026-06-10
> **Status:** Completed

---

## Goal

The `faq` module stops being a stub: admins manage **Q&A pairs** (one table — question, answer, tags), every entry is embedded into pgvector, and the agent answers grounded from them via a real `faq_search` retriever tool. The module is the **proof-of-fire of the full module contract** — first to use `register_models()`, `register_admin_routes()` and `config_schema()` in one self-contained folder. Plus: vector indexes (memories + faq) auto-selected by `EMBEDDING_DIM`, and the Sprint-3 carry-over (memory dedup/update/forget).

**Non-goal:** document upload / chunking / file storage — that's the future document-RAG extension (backlog in `docs/sprints/README.md`).

## Why

- The #1 use case of a WhatsApp business agent: answer company questions **without hallucinating**. Grounded retrieval is the difference between a demo and a product.
- Proves the module contract end-to-end (tools + models + admin routes + config) — the blueprint every future module (and `chasqui generate module`, Sprint 6) copies.
- Memory contradictions ("Soltero" + "Casado" coexist, seen in Sprint 3 e2e) make long-term memory untrustworthy — fix before the admin panel exposes it.

## What

1. `app/modules/faq/` becomes a full module: `models.py` (FaqEntry), `service.py` (embed/search/reembed), `admin.py` (CRUD + re-embed action), `__init__.py` (module + real tool + config schema).
2. Registry gains the two pending hooks: module models reach `SQLModel.metadata` & Alembic, and module admin routers mount under `/admin/modules/<name>` (JWT-protected).
3. Vector ANN indexes on `memories.embedding` + `faq_entries.embedding`, strategy auto-selected by `EMBEDDING_DIM` (ADR-001).
4. Memory module learns to dedup/update/forget.

### Success Criteria

- [x] Admin creates an FAQ via API → the agent answers it grounded over WhatsApp (manual e2e).
- [x] `faq_search` returns relevant entries above a similarity threshold; below it → honest "no tengo ese dato" (no hallucination).
- [x] Editing an entry re-embeds it; `POST .../reembed` re-embeds all.
- [x] `app/modules/faq/` is self-contained — zero edits to `app/models/`, `app/controllers/` or the orchestrator.
- [x] Migration creates HNSW (`vector` ops) at 768 dims; halfvec path proven by unit test logic at 3072.
- [x] Saving a near-duplicate memory updates instead of appending; `update_memory`/`forget_memory` work.
- [x] `make test` green (core), no regressions.

---

## All Needed Context

### Documentation & References

```yaml
- file: core/CLAUDE.md                      # service conventions (no orgs, BSUID, modules)
- file: docs/ARCHITECTURE.md §8             # tool registry contract
- file: docs/design/adr-001-embeddings-provider-dims.md   # dims/index strategy
- file: core/app/modules/registry.py        # ToolModule protocol + discover()
- file: core/app/services/memory_service.py # existing pgvector cosine search pattern
- file: core/app/modules/memory/__init__.py # ToolRuntime[TurnContext] tool pattern
- file: core/app/controllers/admin/auth.py  # CurrentAdmin dependency pattern
- file: core/alembic/versions/002_domain.py # provision-time EMBEDDING_DIM migration pattern
- file: core/tests/conftest.py              # create_all from SQLModel.metadata (gotcha #1)
- ref:  psicolab/backend                    # video-RAG: top-k + similarity threshold
```

### Key decisions (made in this PRP)

1. **Module anatomy** (the blueprint `chasqui generate module` will scaffold):
   ```
   app/modules/faq/
   ├── __init__.py    # FaqModule (register_tools/register_models/register_admin_routes/config_schema) + faq_search tool
   ├── models.py      # FaqEntry (SQLModel, table=True)
   ├── service.py     # embed-on-save, search, reembed_all
   └── admin.py       # APIRouter: CRUD + POST /reembed
   ```
2. **Models reach metadata via import**: `registry.discover()` imports each module package; `faq/__init__.py` imports `models.py` → table lands in `SQLModel.metadata`. Therefore `tests/conftest.py` and `alembic/env.py` call `registry.discover()` before using metadata. `register_models()` returns the classes (explicit contract + future tooling); the import is what does the work.
3. **Admin mounting**: registry gains `mount_admin_routes(router)`; `app/main.py` calls `registry.discover()` at import time (idempotent — lifespan call stays) and includes a parent router `/admin/modules` protected with `Depends(CurrentAdmin)`-equivalent at router level; each module gets prefix `/admin/modules/<module.name>`.
4. **Dim-aware search helper** `app/core/vector_search.py`: `cosine_distance(column, vector)` returns the SQLA expression — plain `column.cosine_distance(v)` when `EMBEDDING_DIM ≤ 2000`, halfvec-cast expression when `2000 < dim ≤ 4000` (MUST textually match the index expression or the index is ignored), plain exact when `> 4000`. `memory_service` and `faq/service` both use it.
5. **Two migrations**: `004_faq_entries` (table, `Vector(settings.embedding_dim)`, provision-time like 002) and `005_vector_indexes` (HNSW on memories + faq_entries, auto-selected; >4000 → no index). Both handwritten like 001–003.
6. **Tool config via `agent_config.tool_config`** (DB-editable, Sprint 5 UI): `faq_search` reads `tool_config["faq_search"]` → `{top_k: 4, min_similarity: 0.5}` defaults. `FaqModule.config_schema()` returns the Pydantic model describing those knobs (first use — feeds Sprint 5 auto-forms).
7. **Memory dedup (carry-over)**: hybrid —
   - `save_memory` dedups on save: nearest memory with cosine distance `< 0.1` → update that row (content + embedding) instead of insert.
   - New silent tools `update_memory(old_content_hint, new_content)` / `forget_memory(content_hint)`: embed the hint, nearest match within distance `< 0.45` → update/delete; no match → save as new / no-op. LLM-driven correction handles contradictions ("me casé") that pure similarity can't.
   - System prompt memory block gains one line instructing the model to correct outdated facts via `update_memory`.

### Known Gotchas

```python
# 1. conftest create_all + TRUNCATE iterate SQLModel.metadata — module tables
#    must be imported first: add registry.discover() to conftest (and alembic/env.py).
# 2. Expression indexes only fire when the query expression MATCHES:
#    halfvec index ⇒ ORDER BY (embedding::halfvec(D)) <=> (:q)::halfvec(D).
#    Centralize in app/core/vector_search.py so index and query can't drift.
# 3. halfvec requires pgvector >= 0.7 (local 0.8.1 OK; docker pgvector/pg17 OK).
# 4. Embedding failures must NEVER break a turn or a CRUD request:
#    save entry with embedding=None + warning; search returns [] + honest tool reply.
# 5. Bulk re-embed: use aembed_documents (one batched call), not N aembed_query.
# 6. session.refresh() does NOT autoflush pending changes — flush first (proven Sprint 3).
# 7. Tool docstrings are the model's manual — English, like every LLM-facing string (i18n posture, see AGENTS.md); the agent localizes via the system prompt.
# 8. `metadata` is reserved in SQLModel → attr `meta`, column "metadata" (if needed).
# 9. agent_config.tool_config may be {} — always .get() with defaults.
# 10. Tests must not hit the network: stub get_embeddings (pattern in test_embeddings.py);
#     deterministic fake vectors let us assert ranking.
```

---

## Implementation Blueprint

### Task order

```yaml
Task 1 - Registry hooks:
  - MODIFY core/app/modules/registry.py: get_models(), mount_admin_routes(router)
  - MODIFY core/app/main.py: discover() at import + include /admin/modules router (JWT)
  - MODIFY core/tests/conftest.py + core/alembic/env.py: registry.discover()
Task 2 - Dim-aware helper:
  - CREATE core/app/core/vector_search.py (cosine_distance expr + index DDL helper + startup warning >4000)
  - MODIFY core/app/services/memory_service.py to use it
Task 3 - FAQ models + migration:
  - CREATE core/app/modules/faq/models.py (FaqEntry: question, answer, tags JSONB, embedding nullable, timestamps)
  - CREATE core/alembic/versions/004_faq_entries.py
Task 4 - FAQ service:
  - CREATE core/app/modules/faq/service.py (create/update embed-on-save, search top-k + threshold, reembed_all batched)
Task 5 - Real tool + config:
  - MODIFY core/app/modules/faq/__init__.py (faq_search via ToolRuntime[TurnContext]; FaqConfig schema; hooks)
Task 6 - Admin CRUD:
  - CREATE core/app/modules/faq/admin.py (list/get/create/update/delete + POST /reembed)
Task 7 - Vector indexes:
  - CREATE core/alembic/versions/005_vector_indexes.py (memories + faq_entries, auto by EMBEDDING_DIM)
Task 8 - Memory dedup carry-over:
  - MODIFY core/app/services/memory_service.py (find_nearest helper)
  - MODIFY core/app/modules/memory/__init__.py (dedup-on-save + update_memory + forget_memory)
  - MODIFY orchestrator memory block hint (one line)
Task 9 - Tests:
  - CREATE core/tests/test_faq.py (service + tool + admin routes + reembed)
  - MODIFY core/tests/test_orchestrator.py if tool list assertions change
  - CREATE/extend tests for memory dedup + vector_search expr selection (768 vs 3072 vs 5000)
Task 10 - Docs (end-of-sprint rule):
  - core/README.md + core/AGENTS.md (module contract now fully exercised)
  - sprint-04 doc checkboxes; ADR-001 stays accurate (index strategy now real)
```

### Data model

```python
class FaqEntry(SQLModel, table=True):
    __tablename__ = "faq_entries"
    id: uuid.UUID            # pk, uuid4
    question: str            # Text, not null — what gets embedded is f"{question}\n{answer}"
    answer: str              # Text, not null
    tags: list[str]          # JSONB, server_default "[]"
    embedding: Any | None    # Vector(settings.embedding_dim), nullable (embedder may fail)
    created_at / updated_at  # naive UTC (project convention)
```

## Validation Loop

```bash
cd core && make test                  # all green, incl. new test_faq.py
# manual: make dev →
#   POST /admin/auth/login → token
#   POST /admin/modules/faq/entries {question, answer, tags}
#   GET  /admin/modules/faq/entries  → embedding present
# e2e WhatsApp (pause for Willy): create 2-3 FAQs, ask over WhatsApp →
#   grounded answer; ask something NOT in the KB → honest "no tengo ese dato";
#   memory correction flow ("ya no X, ahora Y") → update_memory fires.
# core :8090 → ngrok http 8000 (URL from :4040 API!) → sed WA_CALLBACK_URL → gateway :8000
```

## Final Checklist

- [x] Registry: models + admin hooks wired, discover() in conftest/env/main
- [x] FaqEntry + migration 004 applied (provision-time dim)
- [x] Service: embed-on-save, threshold search, batched reembed
- [x] faq_search real (grounded snippets, honest empty answer)
- [x] Admin CRUD JWT-protected under /admin/modules/faq
- [x] Migration 005: HNSW auto-select + startup warning >4000
- [x] Memory dedup + update/forget tools
- [x] Tests green; e2e WhatsApp verified with Willy
- [x] Docs updated (end-of-sprint rule)

## Anti-Patterns to Avoid

- ❌ FAQ logic outside `app/modules/faq/` (kills the self-contained proof)
- ❌ Query expression that doesn't match the index expression (silent seq-scan)
- ❌ Letting an embeddings outage 500 the CRUD or crash a turn
- ❌ Returning raw model objects from tools (must be strings)
- ❌ Hallucination path: tool returning "" instead of an explicit "no data" instruction
- ❌ Per-row embed calls in reembed_all (batch with aembed_documents)
