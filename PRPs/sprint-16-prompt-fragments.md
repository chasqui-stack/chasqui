# PRP: Sprint 16 — Module system-prompt fragments + FAQ question index

> **Version:** 1.0
> **Created:** 2026-09-18
> **Status:** Completed
> **Executor:** Willy + Claude · **Issue:** [chasqui#30](https://github.com/chasqui-stack/chasqui/issues/30) · **ADR:** [ADR-012](../docs/design/adr-012-module-prompt-fragments.md)

---

## Goal

The agent on `chasqui.chat` answered questions from priors instead of calling
`faq_search` (chasqui#30). Fix the *decision to call the tool* at the stack
level: (1) a new optional module hook `system_prompt_fragment(context, query)`
so any module can contribute a per-turn prompt block, (2) the memory facts
block moves out of the core into the `memory` module (proves the hook
generalises), (3) `faq` publishes its question index behind an opt-in knob,
and (4) `faq_search`'s docstring stops describing a shop.

**Non-goals:** semantic pre-filtering of the index, fragment ordering,
automated routing evals — ADR-012 follow-ups.

## Key decisions (details and rationale in ADR-012)

1. Hook signature `async def system_prompt_fragment(self, context, query) -> str | None`
   — `context` is the turn's `TurnContext`, `query` the inbound text
   (concatenated batch on a coalesced turn). Collected by
   `registry.get_prompt_fragments()` — **failure-isolated** (a raising hook
   logs and skips, never breaks the turn), discovery order, appended between
   the capability line and the date.
2. `_invoke()` now receives the `TurnContext` built early by both turn
   variants (it was built twice conceptually before).
3. FAQ knobs (flat, SchemaForm-safe): `inject_question_index: bool = False`,
   `question_index_max: int = 40`. Over the cap → inject **nothing** + warn
   once (module-global flag, resets when back under). Suppressed while
   `faq_search` is disabled. Questions only, framed as look-up-able.
4. `_tool_config()` now takes `AgentConfig` (not `TurnContext`) so the
   fragment and the tool share it.
5. Docstring rewrite: intent-based ("ANY question about this business,
   product or project… you do NOT know these details from training"),
   business-agnostic.

## Touched

```yaml
core (branch feat/prompt-fragments, PR closes chasqui#30):
  - app/modules/registry.py        # protocol comment + get_prompt_fragments()
  - app/services/orchestrator.py   # fragments compose the prompt; memory block gone
  - app/modules/memory/__init__.py # facts block as a fragment
  - app/modules/faq/__init__.py    # knobs, fragment, docstring, _tool_config(AgentConfig)
  - app/modules/faq/service.py     # list_questions()
  - tests: test_orchestrator.py (fragment lands / broken isolated / None omitted),
           test_faq.py (off by default, questions-never-answers + framing,
           over-cap warns once, empty silent, disabled-tool silent),
           test_admin_config.py (schema now has 4 knobs)
  - AGENTS.md                      # contract + turn bullets
parent (docs, direct to main):
  - docs/design/adr-012-module-prompt-fragments.md
  - docs/MODULES.md                # hook in the sketch + "System-prompt fragments" section
  - docs/ARCHITECTURE.md §8.2      # hook in the Protocol sketch
  - PRPs/sprint-15-document-rag-4-docs-adr.md  # Document-RAG's ADR renumbered → ADR-013
```

## Validation

```bash
cd core && make test   # 165 passed
# Live check (dogfood): enable inject_question_index on chasqui.chat's config,
# re-ask "Tiene un skill chasqui? cómo se instala?" → faq_search fires.
```

## Final checklist

- [x] Hook on the contract + registry collector, failure-isolated
- [x] Memory facts block owned by the memory module
- [x] FAQ index opt-in, capped, warn-once, disabled-tool-aware, questions only
- [x] faq_search docstring business-agnostic
- [x] Tests green (165), including framing assertions
- [x] ADR-012 + MODULES.md + ARCHITECTURE §8 + core AGENTS.md
- [ ] Dogfood verification on chasqui.chat (post-merge, Willy)
