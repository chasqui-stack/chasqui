# PRP: Sprint 3 — LangGraph Orchestrator & Tool Registry

> **Version:** 1.0
> **Created:** 2026-06-09
> **Status:** In Progress

---

## Goal

Replace the `Echo:` stub in `core/app/services/orchestrator.py` with a real LangGraph agent (LangChain v1 `create_agent`) that:

1. Answers using a **DB-editable system prompt** + conversation **history** + retrieved **memories**.
2. Uses a **pluggable LLM** via `init_chat_model()` (Gemini / Claude / OpenAI / Ollama / OpenRouter) gated by the per-model **capability registry** (vision/audio).
3. Executes tools contributed by **self-contained modules** under `app/modules/` — discovered at startup, filterable at runtime (enable/disable per project), with tool errors converted to `ToolMessage` instead of crashing the graph.
4. Handles **multimodal input** (the user's image-with-caption and voice notes) end-to-end: gateway downloads media → base64 data URI → content blocks for capable models, graceful text fallback otherwise.

## Why

- This is the moment Chasqui becomes an *agent* instead of an echo pipe — everything before (canonical contract, identity, persistence) existed to feed this turn.
- The Tool Registry is **the differentiator** (ARCHITECTURE §8): companies extend the agent by dropping modules, never editing core.
- The editable prompt + enable/config columns are what Sprint 5's admin panel will surface — the data model must exist now.

## What

A WhatsApp message arrives → gateway normalizes (now including media bytes) → core `/ingest` → orchestrator builds the prompt (DB system prompt + memories + history + current possibly-multimodal message) → `create_agent` runs router→tools→respond → reply persists and returns canonically.

### Success Criteria

- [ ] A WhatsApp question produces an LLM answer respecting the editable system prompt + history + memory.
- [ ] Adding a new module under `app/modules/` exposes its tool **without touching core code**.
- [ ] Toggling a tool off in `agent_config.enabled_tools` removes it from the model's tool set at runtime.
- [ ] A tool exception becomes a `ToolMessage` (graph survives; agent apologizes/recovers).
- [ ] Image + caption and voice notes reach Gemini as native content blocks; a text-only model degrades politely with a logged warning.
- [ ] `make test` green (existing 15 tests adapted + new orchestrator/registry tests).

---

## All Needed Context

### Documentation & References

```yaml
- file: docs/ARCHITECTURE.md §6 §8        # memory + Tool Registry design (source of truth)
- file: core/app/services/orchestrator.py # the stub being replaced — run_turn() is the seam
- file: core/app/services/ingest_service.py
- file: core/app/modules/registry.py      # ToolModule protocol + in-memory registry (Sprint 0)
- file: core/app/core/llm_capabilities.py # vision/audio resolution (Sprint 2)
- ref: psicolab/backend/app/services/ai_chat_service.py  # multimodal content blocks pattern
- versions: langchain 1.3.4 · langchain-core 1.4.2 · langgraph 1.2.4 · langchain-google-genai 4.2.4
```

### Verified LangChain v1 APIs (inspected in core/.venv)

- `create_agent(model, tools, system_prompt, middleware, context_schema, ...)` — prebuilt router→ToolNode→respond loop.
- `AgentMiddleware` hooks: `awrap_model_call(request, handler)` / `awrap_tool_call(request, handler)`.
- `ModelRequest` fields incl. `tools`, `runtime`; has `.override(tools=...)` for immutable-style replacement.
- `ToolRuntime` injected param → `runtime.context` (our per-turn dataclass), `runtime.tool_call_id`.
- `init_chat_model("google_genai:gemini-3-flash-preview", api_key=...)`.

### Key decisions (made in this PRP)

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **`create_agent` + middleware** instead of hand-rolled `StateGraph` | psicolab predates LangChain v1; the prebuilt agent IS router→ToolNode→respond, and middleware gives us `wrap_model_call`/`wrap_tool_call` exactly as ARCHITECTURE §8 prescribes. Custom nodes can still be added later (the result is a compiled LangGraph graph). |
| 2 | **`agent_config` singleton table** (system_prompt, enabled_tools JSONB, tool_config JSONB) | Chasqui = one project per deployment (no multi-tenant). Migration 003 seeds the default row; Sprint 5 admin edits it. |
| 3 | **Media as base64 `data:` URI in canonical `media_url`** | Meta media URLs expire (~5 min) and require the WA token — the core must stay channel-agnostic, so the **gateway** downloads bytes (PyWa `download()` → tempfile → base64) and ships a self-contained data URI. Contract unchanged (`media_url` is still a URI). Only the **current** inbound message is sent multimodal; history stays text-only (token control). |
| 4 | **Memory extraction via `save_memory` tool** (memory module) instead of a post-turn LLM pass | One LLM call per turn instead of two; the model saves silently when relevant (proven pattern in psicolab). `extract_after_turn()` seam stays as a documented no-op for batch extraction later. `retrieve_relevant()` becomes real: Google embeddings (text-embedding-004, 768d) + pgvector cosine. |
| 5 | **Turn runs BEFORE persisting the inbound row** | History query then naturally returns only *prior* messages (no self-exclusion hacks). Single transaction semantics unchanged (failure rolls back everything either way). Inbound keeps `received_at` as `created_at`, so ordering is preserved. |
| 6 | **Module discovery via `pkgutil` scan** of `app/modules/*` packages exposing a module-level `module` attribute | Drop a folder = get its tools. No entry-points/config needed. |

### Desired Structure

```bash
core/app/
├── core/
│   ├── llm.py                  # NEW get_chat_model() factory (init_chat_model)
│   └── config.py               # MODIFY: openai/openrouter/ollama keys, history_limit
├── models/agent_config.py      # NEW singleton config (prompt, enabled_tools, tool_config)
├── alembic/versions/003_agent_config.py  # NEW + seed default row
├── services/
│   ├── agent_config_service.py # NEW get_config() (cached-per-turn singleton fetch)
│   ├── orchestrator.py         # REWRITE: create_agent turn (same seam, + session param)
│   ├── memory_service.py       # MODIFY: real retrieve_relevant (pgvector)
│   └── ingest_service.py       # MODIFY: turn-before-persist order, pass session
├── core/embeddings.py          # NEW get_embeddings() (Google text-embedding-004)
└── modules/
    ├── registry.py             # MODIFY: discover() via pkgutil
    ├── faq/__init__.py         # NEW faq_search (stub — real RAG Sprint 4)
    ├── handoff/__init__.py     # NEW human_handoff + lead_capture (conversation_state)
    └── memory/__init__.py      # NEW save_memory tool

whatsapp/app/
├── services/media.py           # NEW download_to_data_uri(media, mime) helper
└── handlers/message_handlers.py # MODIFY: image/audio payloads carry data URI

docs/design/module-example-commercial-locations.md  # NEW (docs-only reference module)
```

### Known Gotchas

```python
# 1. SQLModel: `metadata` is reserved → Python attr `meta`, DB column "metadata"
# 2. asyncpg: TIMESTAMP WITHOUT TIME ZONE → naive UTC (_to_naive_utc)
# 3. Gemini returns content as block lists → use result.text / filter "text" blocks
# 4. init_chat_model provider string for Gemini is "google_genai", NOT "google"
#    (llm_capabilities registry keys keep using settings.llm_provider verbatim)
# 5. ToolRuntime param must be named `runtime` and excluded from args_schema
# 6. Tests must not require GOOGLE_API_KEY: orchestrator takes an injectable model
#    (tests pass a scripted FakeChatModel; prod path uses get_chat_model())
# 7. Existing test_ingest.py asserts the Echo stub → patch run_turn there
# 8. PyWa download() writes to disk only → tempfile → read → unlink
```

---

## Implementation Blueprint

### Task order

```yaml
Task 1: agent_config model + migration 003 (+ seed) + agent_config_service
Task 2: app/core/llm.py (get_chat_model) + config.py additions
Task 3: registry discovery + 3 modules (faq, handoff, memory) + middleware
        (ToolFilterMiddleware via awrap_model_call, ToolErrorMiddleware via awrap_tool_call)
Task 4: memory_service.retrieve_relevant real (embeddings + pgvector cosine)
Task 5: orchestrator rewrite — build_messages(system+memories, history, current
        multimodal block per llm_capabilities) → create_agent.ainvoke(context=...)
Task 6: ingest_service reorder + main.py startup discovery
Task 7: gateway media download → data URI (image, audio)
Task 8: tests (FakeChatModel: round-trip, disabled tool, tool error, prompt
        assembly, multimodal gating) + adapt test_ingest.py
Task 9: docs (Commercial Locations reference, READMEs/AGENTS) + e2e WhatsApp
```

### Per-turn context (ToolRuntime)

```python
@dataclass
class TurnContext:
    session: AsyncSession        # tools may persist (handoff, save_memory)
    contact_id: uuid.UUID
    conversation_id: uuid.UUID
    config: AgentConfig          # enabled_tools / tool_config visible to middleware
```

## Validation Loop

```bash
cd core && make test                  # all unit + DB tests
# live: core :8090 (gemini-3-flash-preview) → ngrok http 8000 → sed WA_CALLBACK_URL
# → gateway :8000 → user messages +1 555-658-0492:
#   text question (history+prompt respected), image+caption, voice note,
#   "necesito hablar con un humano" (human_handoff tool fires)
psql chasqui -c "select direction,type,left(text,60) from messages order by created_at desc limit 6;"
```

## Final Checklist

- [ ] Migration 003 applied; default config row seeded
- [ ] `get_chat_model()` builds gemini-3-flash-preview from .env
- [ ] Modules discovered at startup (log line lists tools)
- [ ] Tool filter middleware removes disabled tools at runtime
- [ ] Tool exception → ToolMessage (test proves it)
- [ ] Multimodal: image/audio content blocks for capable models; warn+fallback otherwise
- [ ] Memories: retrieve before turn; save_memory tool writes with embedding
- [ ] Gateway ships data URIs; e2e verified on real WhatsApp
- [ ] Conventional commits in core + whatsapp + parent submodule bump

## Anti-Patterns to Avoid

- ❌ Don't put module logic in core services (modules are self-contained)
- ❌ Don't send full base64 history to the LLM (current message only)
- ❌ Don't let a tool exception kill the turn
- ❌ Don't block /ingest on memory extraction beyond the single agent call
- ❌ Don't hardcode Gemini anywhere outside get_chat_model()
