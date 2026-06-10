# ADR-001: Provider-swappable embeddings & 768 dims as the project constant

> **Status:** Accepted — 2026-06-10
> **Context:** Sprint 3 follow-up. The LLM was already provider-swappable
> (`init_chat_model`, `core/app/core/llm.py`); embeddings were hardcoded to
> Google. Google also retired `text-embedding-004` (404), forcing a model
> migration and raising the dimensions question (gemini-embedding-001
> defaults to 3072).

## Decision 1 — Embeddings are swappable via `init_embeddings()`

Same pattern as the chat model: LangChain's `init_embeddings()` behind a
factory (`core/app/core/embeddings.py`), configured by `.env` only:

```bash
EMBEDDING_PROVIDER=google  EMBEDDING_MODEL=gemini-embedding-001   # default
EMBEDDING_PROVIDER=openai  EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_PROVIDER=ollama  EMBEDDING_MODEL=nomic-embed-text       # local
```

**Caveat:** vectors from different models live in different spaces.
Switching providers requires **re-embedding every stored row** (memories,
FAQ entries). A "re-embed all" admin action ships with Sprint 4.

## Decision 2 — `EMBEDDING_DIM = 768` is the canonical dimension

The DB schema (`Vector(768)`, migration 002) is the contract; every provider
must produce 768:

| Provider | How it hits 768 |
|----------|-----------------|
| Google `gemini-embedding-001` | Matryoshka — `output_dimensionality=768` requested (native 3072) |
| OpenAI `text-embedding-3-*` | Matryoshka — `dimensions=768` requested |
| Ollama `nomic-embed-text` | 768 native |

Why not 3072: MRL retains ~all retrieval quality at 768 for our scale
(per-contact memories, FAQ collections), storage is 4× cheaper, search is
faster, and **768 is the sweet spot every major provider supports**.

## Decision 3 — Dimension limits are a pgvector concern, NOT a Postgres-version concern

Recorded explicitly because it came up as "should we upgrade PG 16 → 17 to
allow more dims?" — **no**:

| Limit | Set by |
|-------|--------|
| `vector` indexable (HNSW/IVFFlat) ≤ **2,000 dims** | pgvector extension, any PG version |
| `halfvec` indexable ≤ **4,000 dims** (3072 fits) | pgvector ≥ 0.7 |
| Exact (non-indexed) search | no practical limit |

If we ever need 3072+: migrate the column to `halfvec` (pgvector ≥ 0.7) — a
Postgres major upgrade changes nothing here. PG 17 in our images is for
general currency only.

## Consequences / changes made

- `core/app/core/embeddings.py`: `init_embeddings()` factory + provider map
  + per-provider dimension kwarg (`tests/test_embeddings.py`).
- `core/app/core/config.py`: `EMBEDDING_PROVIDER` setting.
- `docker-compose.yml`: `pgvector/pgvector:pg16` → `:pg17`.
- `core/config/deploy.yml` (Kamal): **bug fix** — was plain `postgres:17`,
  which doesn't ship the `vector` extension (`init.sql` would fail) →
  `pgvector/pgvector:pg17`; also de-templated leftover `saas-template-*`
  names and aligned secrets (INTERNAL_API_KEY, GOOGLE_API_KEY).
- Local dev note: Postgres.app 16.13 + pgvector 0.8.1 already supports
  `halfvec`; upgrading local PG is optional and unrelated to dims.
