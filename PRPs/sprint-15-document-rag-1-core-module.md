# PRP: Sprint 15.1 — Document-RAG: `knowledge` core module (upload, chunking, processing)

> **Version:** 1.0
> **Created:** 2026-07-12
> **Status:** Ready
> **Executor:** Bryan (@BryanDev2023) · **Reviewer:** Willy (@willywg)
> **Series:** 1/4 — execute in order: 15.1 → 15.2 → 15.3 → 15.4

---

## Goal

A new built-in Tool Module `knowledge` in `core/app/modules/knowledge/`: admins upload
documents (**pdf, docx, txt, md, html**), the core extracts their text, splits it into
**overlapping chunks**, embeds each chunk into pgvector, and tracks per-document
processing state (`pending → processing → ready | error`). CRUD + search-preview
admin endpoints mirror the FAQ module. **No agent tool yet** — that's PRP 15.3.

**Non-goals (this PRP):** the admin UI (15.2), the `search_documents` agent tool
(15.3), OCR for scanned PDFs, storing the original file in S3, xlsx/pptx support.

## Why

- FAQ-RAG answers curated Q&A; real businesses have manuals, catalogs, price lists
  and policies as **files**. Document-RAG is the #1 post-MVP feature (backlog since
  Sprint 4).
- Second full exercise of the module contract (models + admin routes + config) —
  validates that `app/modules/faq/` really is a copyable blueprint.

## What

1. `app/modules/knowledge/` — self-contained module: `models.py`, `service.py`
   (extract/chunk/embed/process), `admin.py` (upload/list/delete/reprocess/search),
   `__init__.py` (module singleton; tools list **empty for now**).
2. Migration `008_knowledge_documents`: `documents` + `document_chunks` tables +
   HNSW index on `document_chunks.embedding` (dim-aware, same one-shot as 005).
3. Async processing via FastAPI `BackgroundTasks` + status column (decision below).

### Success Criteria

- [ ] `POST /admin/modules/knowledge/documents` (multipart) with a PDF → `202`,
      row `pending`; seconds later `GET …/documents` shows `ready` + `chunk_count > 0`.
- [ ] All five types extract: pdf, docx, txt, md, html. A `.doc` (legacy) upload →
      `415` with a clear error. A >10 MB file → `413`.
- [ ] Chunks are overlapping (verify: consecutive chunks share text) and each has an
      embedding (or the document lands in `error`, never half-indexed as `ready`).
- [ ] Re-uploading the same content (same sha256) → `409`.
- [ ] `DELETE …/documents/{id}` removes the document AND its chunks (FK cascade).
- [ ] `POST …/documents/{id}/reprocess` re-chunks + re-embeds from stored text;
      works on `error` docs too.
- [ ] A scanned PDF (no extractable text) ends `error` with detail
      `"no extractable text"` — never `ready` with 0 chunks.
- [ ] An embeddings outage NEVER 500s the upload endpoint: the doc ends in `error`
      with detail; a later `reprocess` recovers it.
- [ ] `GET …/search?q=` preview returns chunks with similarity scores (FAQ analog).
- [ ] `cd core && make test` green — new `tests/test_knowledge.py` covers service,
      states, limits, dedupe and admin routes (embeddings stubbed, no network).
- [ ] Zero edits outside `app/modules/knowledge/`, the migration, `pyproject.toml`
      and tests. The module is self-contained (FAQ proved it; keep it true).

---

## All Needed Context

### Documentation & References

```yaml
- file: core/AGENTS.md                       # module contract summary (§ Tool Registry)
- file: docs/MODULES.md                      # module authoring guide
- file: docs/ARCHITECTURE.md §8              # module contract, tool registry
- file: docs/design/adr-001-embeddings-provider-dims.md  # dims/index strategy — READ FIRST
- file: core/app/modules/faq/                # THE blueprint — mirror every file
- file: core/app/modules/registry.py         # ToolModule protocol + discover()
- file: core/app/core/vector_search.py       # cosine_distance() + hnsw_index_ddl() — MUST use
- file: core/app/core/embeddings.py          # get_embeddings(); aembed_documents batching
- file: core/alembic/versions/004_faq_entries.py   # provision-time Vector(dim) pattern
- file: core/alembic/versions/005_vector_indexes.py # hnsw_index_ddl() in a migration
- file: core/tests/test_faq.py               # fake-embeddings + admin-auth test patterns
- file: core/tests/conftest.py               # registry.discover(); savepoint sessions
- url: https://python.langchain.com/docs/how_to/recursive_text_splitter/
  why: RecursiveCharacterTextSplitter — chunk_size/chunk_overlap semantics
```

### Key decisions (made in this PRP — don't relitigate, ADR lands in 15.4)

1. **Extracted text lives in Postgres, original file is discarded.** `documents.content_text`
   (Text) keeps the full extracted text so `reprocess` re-chunks/re-embeds without the
   original file, and the module has **zero dependency on S3 storage** (works on every
   install). Storing originals in the Sprint-6 bucket = follow-up in the ADR.
2. **Async = FastAPI `BackgroundTasks` + status column.** The core has no generic job
   queue (the Sprint-10 worker is inbound-coalescing only, ADR-008). Upload returns
   `202` immediately; `_process_document(doc_id)` runs in background updating
   `status`. A doc stuck in `processing` after a crash is recovered manually via
   `reprocess`. A SKIP-LOCKED worker à la `coalesce_worker.py` = ADR follow-up.
3. **Chunking (omakase constants, not user knobs):** `RecursiveCharacterTextSplitter`,
   `CHUNK_SIZE = 1200` chars, `CHUNK_OVERLAP = 180` (15%). For `.md` use
   `RecursiveCharacterTextSplitter.from_language(Language.MARKDOWN, …)` (respects
   headings/fences). Changing constants later = `reprocess`. Retrieval knobs (top_k,
   min_similarity) are agent `tool_config` — that's 15.3, not here.
4. **Extractors — light deps only** (add to `core/pyproject.toml` [project] deps):
   `pypdf` (PDF), `python-docx` (docx), `beautifulsoup4` (html → `get_text()`),
   `langchain-text-splitters` (splitter). txt/md = plain `bytes.decode("utf-8",
   errors="replace")`. NO `unstructured` (heavyweight). Legacy `.doc` → `415`.
5. **Limits & dedupe:** max upload `10 MB` (`413` beyond), extension+MIME whitelist
   (`415` otherwise), sha256 over raw bytes stored in `documents.content_sha256`
   (unique) → duplicate upload = `409`.
6. **Embedding batching:** all chunks of a document embed in **one**
   `aembed_documents(texts)` call (the FAQ `reembed_all` pattern). If it raises →
   doc `status="error"`, `error_detail=str(exc)[:500]`, chunks rolled back.
7. **Tables** (`documents`, `document_chunks`) are module-owned via
   `register_models()` — exactly like `FaqEntry`. Embedding column
   `Vector(settings.embedding_dim)` captured at migration time (004 pattern).
   HNSW index via `hnsw_index_ddl("document_chunks")` in the same migration 008.

### Known Gotchas

```python
# 1. ALL vector queries via app/core/vector_search.cosine_distance() — a raw
#    .cosine_distance() at >2000 dims silently seq-scans (expression-index rule).
# 2. Module tables reach SQLModel.metadata via import: knowledge/__init__.py must
#    import models at module level (faq does it) — conftest/alembic discover() it.
# 3. BackgroundTasks run AFTER the response, in the SAME process, WITHOUT the
#    request's session. _process_document must open its OWN session
#    (async_session_factory pattern — see how coalesce_worker builds sessions).
# 4. UploadFile: `await file.read()` returns bytes — check size AFTER read (don't
#    trust Content-Length), hash those bytes for dedupe.
# 5. Embedding failures must never 500 a request NOR leave status="processing"
#    forever: wrap the whole background job in try/except → status="error".
# 6. pypdf extract_text() returns "" for scanned pages: strip+join all pages; if
#    the total is empty → error "no extractable text" (NOT ready-with-0-chunks).
# 7. `metadata` is a reserved attr in SQLModel — use `meta` if you need it.
# 8. session.refresh() doesn't autoflush pending changes — flush first.
# 9. Tests: stub get_embeddings with deterministic fake vectors (test_faq.py
#    pattern); BackgroundTasks in tests: call the service function directly, or use
#    TestClient which executes background tasks after the response.
# 10. English-only: models, errors, comments, docstrings. UI strings are 15.2's job.
# 11. FastAPI multipart needs python-multipart — already a core dep (OAuth2 login).
```

---

## Implementation Blueprint

### Data model

```python
class Document(SQLModel, table=True):
    __tablename__ = "documents"
    id: uuid.UUID                    # pk uuid4
    filename: str                    # original name, max 255
    mime_type: str
    size_bytes: int
    content_sha256: str              # unique — dedupe
    content_text: str | None         # extracted text (Text) — source for reprocess
    status: str                      # "pending" | "processing" | "ready" | "error"
    error_detail: str | None
    chunk_count: int                 # denormalized, set on success
    created_at / updated_at          # naive UTC (project convention)

class DocumentChunk(SQLModel, table=True):
    __tablename__ = "document_chunks"
    id: uuid.UUID
    document_id: uuid.UUID           # FK documents.id ON DELETE CASCADE, index
    seq: int                         # 0-based order within the document
    content: str                     # the chunk text (Text)
    embedding: Any | None            # Vector(settings.embedding_dim), nullable
    created_at
```

### Task order

```yaml
Task 1 - Deps:
  - MODIFY core/pyproject.toml: add pypdf, python-docx, beautifulsoup4,
    langchain-text-splitters → `uv sync`
Task 2 - Models + migration:
  - CREATE core/app/modules/knowledge/models.py (Document, DocumentChunk)
  - CREATE core/alembic/versions/008_knowledge_documents.py
    (both tables + FK CASCADE + unique content_sha256 + hnsw_index_ddl("document_chunks"))
  - RUN: cd core && make migrate
Task 3 - Extraction:
  - CREATE core/app/modules/knowledge/extract.py
    (extract_text(filename, mime, data: bytes) -> str; per-type helpers; whitelist
     ALLOWED = {".pdf", ".docx", ".txt", ".md", ".html"}; UnsupportedType/EmptyText errors)
Task 4 - Service:
  - CREATE core/app/modules/knowledge/service.py
    (create_document → pending row; process_document(doc_id) → own session,
     status transitions, split via RecursiveCharacterTextSplitter, ONE
     aembed_documents, bulk-insert chunks, chunk_count; delete_document;
     reprocess(doc_id); search(session, query, top_k, min_similarity) via
     cosine_distance — mirror faq/service.py search exactly)
Task 5 - Admin routes:
  - CREATE core/app/modules/knowledge/admin.py
    POST   /documents        (UploadFile → validate → create → BackgroundTasks.add_task → 202)
    GET    /documents        (list, newest first: id, filename, mime, size, status,
                              error_detail, chunk_count, timestamps)
    DELETE /documents/{id}   (204; cascade removes chunks)
    POST   /documents/{id}/reprocess  (202; from content_text; 409 if processing)
    GET    /search?q=&top_k=8&min_similarity=0   (preview, FAQ analog: content
                              snippet, similarity, document filename)
Task 6 - Module singleton:
  - CREATE core/app/modules/knowledge/__init__.py
    (KnowledgeModule: name="knowledge", register_tools() -> [] (15.3 fills it),
     register_models() -> [Document, DocumentChunk], register_admin_routes;
     NO config_schema yet — arrives with the tool in 15.3)
Task 7 - Tests:
  - CREATE core/tests/test_knowledge.py
    (extract per type w/ tiny fixture files, size/type/dup rejections, full
     process happy path w/ fake embeddings, scanned-pdf → error, embed-outage →
     error + reprocess recovery, cascade delete, search ranking, admin routes
     with auth token — mirror test_faq.py fixtures)
Task 8 - Sanity:
  - RUN: cd core && make test && make dev  → manual curl flow (below)
```

## Validation Loop

```bash
cd core && uv sync && make migrate && make test        # all green

# Manual (make dev in another terminal; get a token first):
TOKEN=$(curl -s -X POST localhost:8090/admin/auth/login \
  -d 'username=admin@chasqui.local&password=changeme123' \
  -H 'Content-Type: application/x-www-form-urlencoded' | jq -r .access_token)
curl -s -X POST localhost:8090/admin/modules/knowledge/documents \
  -H "Authorization: Bearer $TOKEN" -F "file=@some.pdf"          # → 202 pending
curl -s localhost:8090/admin/modules/knowledge/documents \
  -H "Authorization: Bearer $TOKEN" | jq                          # → ready + chunk_count
curl -s "localhost:8090/admin/modules/knowledge/search?q=refund+policy" \
  -H "Authorization: Bearer $TOKEN" | jq                          # → scored chunks
```

## Final Checklist

- [ ] Migration 008 applied; HNSW index exists (`\d document_chunks` shows it)
- [ ] Five file types extract; .doc → 415; >10MB → 413; duplicate → 409
- [ ] States: pending → processing → ready/error; reprocess recovers error docs
- [ ] One batched embed call per document; outage → error, never a 500
- [ ] Delete cascades chunks; search preview scored like FAQ's
- [ ] `make test` green; zero edits outside the module + migration + tests + deps
- [ ] PR from your fork → `chasqui-stack/core`, branch `feat/knowledge-module`,
      title `feat: knowledge module — document upload, chunking, embedding (Sprint 15.1)`,
      body `Closes #<core-issue-number>`

## Anti-Patterns to Avoid

- ❌ Knowledge logic outside `app/modules/knowledge/` (breaks the module contract)
- ❌ Raw `.cosine_distance()` instead of `vector_search.cosine_distance()`
- ❌ Reusing the request session inside the background task
- ❌ Per-chunk embed calls (batch with `aembed_documents`)
- ❌ `ready` with 0 chunks, or `processing` forever after a crash path you control
- ❌ Spanish (or any non-English) strings in code/errors — UI localizes, core doesn't
- ❌ New retrieval-knob settings here — top_k/min_similarity belong to 15.3's tool_config
