# PRP: Sprint 15.3 — Document-RAG: `search_documents` agent tool (FAQ-safe retrieval)

> **Version:** 1.0
> **Created:** 2026-07-12
> **Status:** Ready
> **Executor:** Bryan (@BryanDev2023) · **Reviewer:** Willy (@willywg)
> **Series:** 3/4 — requires 15.1 merged (15.2 helps for e2e but isn't a blocker)

---

## Goal

The `knowledge` module grows its agent-facing half: a `search_documents` tool
(on-demand retrieval — the agent formulates the query when it decides to call it,
exactly like `faq_search`), with `tool_config` knobs (`top_k`, `min_similarity`)
editable from the Tools page via `config_schema()`, enable/disable for free through
the existing registry — and **proven non-overlap with FAQ**: two similar retrievers
must not trip over each other.

**Non-goals:** reranking, hybrid (keyword+vector) search, citations with page
numbers, cross-encoder scoring — ADR follow-ups.

## Why

- Chunks without a tool are dead weight; this is where uploaded documents start
  answering real user questions.
- FAQ (curated Q&A) and Knowledge (uploaded docs) are semantically close — the
  known risk of this sprint. The mitigation is cheap and testable: sharply distinct
  tool descriptions + an eval checklist, decided here.

## What

1. `search_documents` LangChain `@tool` in `knowledge/__init__.py` — mirror of
   `faq_search` (same `ToolRuntime[TurnContext]` access, same honest-empty answer).
2. `KnowledgeSearchConfig` (Pydantic, FLAT) + `config_key = "document_search"` +
   `config_schema()` → the Tools page auto-renders the knobs; `enabled_tools`
   toggle works with zero admin changes.
3. Tool-description discipline (the anti-overlap mechanism) + a 10-question eval
   run as acceptance.

### Success Criteria

- [ ] Agent e2e (web widget or Telegram): a question answerable ONLY from an
      uploaded document gets a grounded answer citing document content.
- [ ] A question answerable ONLY from FAQ still routes to `faq_search` (no
      regression) — and vice versa. Eval: 5 FAQ-only + 5 docs-only questions,
      ≥ 4/5 correct tool choice each side (inspect turns via admin conversations
      or core logs).
- [ ] Below `min_similarity` → tool returns the explicit NO_RESULTS instruction;
      the agent answers honestly (no hallucinated document content).
- [ ] Tools page: `search_documents` appears under the knowledge module, its
      Switch disables it live (agent stops calling it), knobs save and apply.
- [ ] `PUT /admin/config` rejects an invalid `tool_config["document_search"]`
      (422 — schema validation already generic, just verify).
- [ ] `make test` green: tool invocation, config parsing (invalid → defaults),
      NO_RESULTS path, snippet format, module-contract assertions extended.

---

## All Needed Context

### Documentation & References

```yaml
- file: core/app/modules/faq/__init__.py      # THE mirror: @tool + ToolRuntime + _tool_config + module class
- file: core/app/modules/knowledge/           # your 15.1 module — tool plugs into __init__.py
- file: core/app/services/agent_middleware.py # how enabled_tools filters (nothing to change; understand it)
- file: core/app/services/agent_config_service.py  # tool_enabled default-True semantics
- file: core/app/controllers/admin/config.py  # tool_config validation on write (generic)
- file: core/tests/test_faq.py                # tool test pattern: await tool.coroutine(query=…, runtime=…)
- file: docs/ARCHITECTURE.md §8               # tool registry contract
- file: admin/src/pages/ToolsPage.tsx         # verify-only: module appears, SchemaForm renders knobs
```

### Key decisions

1. **Tool descriptions are the router.** The LLM picks tools by docstring; make the
   boundary explicit and complementary — each description says what the OTHER tool
   is for:

   ```python
   # faq_search (MODIFY - one sentence appended):
   """Search the operator-curated FAQ: short official answers (prices, policies,
   hours, contact info). For content inside uploaded documents (manuals, catalogs,
   contracts), use search_documents instead. Args: query — the key concepts."""

   # search_documents (NEW):
   """Search the uploaded document knowledge base (manuals, catalogs, guides,
   contracts): returns relevant passages from those files. For short official
   answers curated by the operator (prices, policies, hours), use faq_search
   instead. Args: query — the key concepts, not the user's literal message."""
   ```

2. **Config:** `KnowledgeSearchConfig(top_k: int = 4, min_similarity: float = 0.35)`.
   Chunks score lower than curated Q&A pairs — 0.35 default vs FAQ's 0.5; FLAT
   schema (str/int/float/bool) so SchemaForm renders it. `config_key =
   "document_search"`.
3. **Tool return format** (strings only, English — the agent localizes):

   ```
   Relevant passages from the knowledge base:

   [catalog.pdf] <chunk content …>

   [manual.docx] <chunk content …>
   ```

   Empty → the FAQ NO_RESULTS pattern: an explicit instruction to say the
   information isn't available, never an empty string.
4. **Both-tools coexistence is prompt-free:** no system-prompt surgery to route
   between them — descriptions must carry it (that's the eval's point). If the eval
   fails, iterate the docstrings, not the prompt seed.

### Known Gotchas

```python
# 1. Tool docstrings and returns: English only (LLM-facing strings convention).
# 2. Snippet budget: cap each chunk to ~600 chars in the return (chunks are 1200)
#    and top_k defaults to 4 — keep the tool message small; whole-context dumps
#    degrade the turn.
# 3. tool_config may be missing/invalid → parse via the faq _tool_config pattern
#    (fallback to defaults with a warning), never raise inside the tool.
# 4. Tool exceptions become error ToolMessages via middleware — still, catch your
#    own expected failures (embed outage → NO_RESULTS honesty, not a stack trace).
# 5. enabled_tools: missing key = enabled (default-True) — new tool is live on
#    merge; mention it in the PR description so Willy knows.
# 6. The search service already exists (15.1) — the tool ADDS NO new SQL; it calls
#    knowledge.service.search(). If you're writing SQL here, stop.
# 7. Eval needs seeded data: 3+ FAQ entries AND 1+ real document whose content
#    doesn't duplicate the FAQs — disjoint topics make tool-choice measurable.
```

---

## Implementation Blueprint

### Task order

```yaml
Task 1 - Config schema:
  - MODIFY core/app/modules/knowledge/__init__.py: KnowledgeSearchConfig +
    config_key = "document_search" + config_schema() hook
Task 2 - The tool:
  - MODIFY core/app/modules/knowledge/__init__.py: @tool search_documents
    (ToolRuntime[TurnContext] → _tool_config → service.search → formatted
    passages w/ [filename] prefix, or NO_RESULTS instruction)
  - register_tools() returns [search_documents]
Task 3 - FAQ description touch-up:
  - MODIFY core/app/modules/faq/__init__.py: append the one-sentence boundary to
    faq_search's docstring (decision 1) — smallest possible diff
Task 4 - Tests:
  - MODIFY core/tests/test_knowledge.py: tool invocation w/ fake embeddings
    (ranking respected, filename prefixes, cap), NO_RESULTS path, invalid
    tool_config → defaults, module contract now exposes 1 tool + config schema
  - CHECK core/tests/test_orchestrator.py for tool-list assertions that may
    need the new tool added
Task 5 - Manual verification:
  - Tools page shows the knowledge module + Switch + knobs (SchemaForm) — no
    admin code changes should be needed; if something doesn't render, the bug
    is in config_schema() flatness
Task 6 - Eval (acceptance, manual):
  - Seed: 3 FAQs (hours, refund policy, contact) + 1 document (e.g., a product
    manual PDF) with disjoint content
  - Run the 10 questions over a real channel; record tool calls (core logs);
    ≥4/5 correct routing per side; iterate docstrings if under
```

## Validation Loop

```bash
cd core && make test                    # green, incl. extended test_knowledge.py

# Live: make dev (core) + a channel (web gateway is the fastest: cd web && npm run dev)
# 1. Ask a docs-only question → grounded answer from the document
# 2. Ask a FAQ-only question → faq_search fires (logs), same answer as before
# 3. Ask something in neither → honest "I don't have that information"
# 4. Admin → Tools → disable search_documents → docs question now comes up empty
#    (honest), re-enable → grounded again
# 5. Lower min_similarity to 0.1 via knobs → noisier passages retrieved (knob applies)
```

## Final Checklist

- [ ] search_documents registered, enabled by default, disable/enable live
- [ ] Knobs render in Tools page via SchemaForm and persist through /admin/config
- [ ] Passages capped + filename-prefixed; NO_RESULTS honest path
- [ ] faq_search docstring boundary added (and nothing else touched in faq)
- [ ] 10-question eval done and reported in the PR description (routing table)
- [ ] make test green
- [ ] PR from your fork → `chasqui-stack/core`, branch `feat/knowledge-tool`,
      title `feat: search_documents tool + FAQ boundary (Sprint 15.3)`,
      body `Closes #<core-issue-number>` + the eval results table

## Anti-Patterns to Avoid

- ❌ Routing via system-prompt edits — descriptions are the router here
- ❌ Identical/vague docstrings for faq_search vs search_documents (the overlap bug)
- ❌ Returning raw ORM objects or empty strings from the tool
- ❌ New SQL in the tool (service.search exists; reuse it)
- ❌ Unbounded passage dumps into the tool message
- ❌ Skipping the eval — "it compiles" is not "it routes"
