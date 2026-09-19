# ADR-012 — Module system-prompt fragments (and the opt-in FAQ question index)

**Status:** Accepted — 2026-09-18 · decision 4's docstring wording amended by [ADR-013](./adr-013-document-rag.md) (decision 8)
**Sprint:** 16
**Related:** [ARCHITECTURE §8](../ARCHITECTURE.md) (module contract), [MODULES.md](../MODULES.md), [ADR-009](./adr-009-extension-skills.md) (the dogfood deployment that surfaced this), issue [chasqui#30](https://github.com/chasqui-stack/chasqui/issues/30)

## Context

Dogfooding on `chasqui.chat` (a real stack answering questions about Chasqui
itself), the agent was asked *"Tiene un skill chasqui? cómo se instala?"* and
confidently invented an answer — it never called `faq_search`, even though the
knowledge base had the entry. Retrieval was fine (cross-lingual embeddings
ranked the right entry #1); the failure was entirely in the **decision to call
the tool**. Two causes:

1. `faq_search`'s docstring described a shop ("products, services, prices,
   schedules…"), so questions about *skills* or *architecture* matched none of
   its nouns. In Chasqui the docstring IS the prompt — and every non-shop
   deployment inherited it.
2. The deployment's system prompt listed the topics the agent handles, which
   the model reads as *"you already know these"*.

Prompt-side fixes per deployment work on capable models, but **weaker models
under-call tools no matter how the operator words the prompt** — and those are
exactly the models people self-host.

There was also a layering smell: `_system_message()` in the orchestrator
already composes the prompt from parts, but hardcoded the memory module's
facts block — the core knew about one specific module at prompt-assembly time.

## Decision

1. **New optional module hook** (the fifth on the contract, mirroring the
   existing optional ones):

   ```python
   async def system_prompt_fragment(self, context, query) -> str | None: ...
   ```

   `context` is the turn's `TurnContext` (session, contact_id,
   conversation_id, config); `query` is the inbound text (concatenated batch
   text on a coalesced turn). `None`/empty = nothing to say this turn.
   `registry.get_prompt_fragments()` collects them **failure-isolated**: a
   raising hook is logged and skipped, never breaking the turn. The
   orchestrator appends fragments between the capability line and the date,
   in module-discovery order.

2. **The memory facts block moves into the `memory` module** as a fragment.
   The core no longer names any module in prompt assembly — proof the hook
   generalises (a commercial `locations` module could publish "you serve
   these 3 cities" the same way).

3. **`faq` publishes its question index, opt-in.** Two flat knobs in
   `tool_config["faq_search"]`: `inject_question_index` (default **false** —
   it costs tokens on every turn) and `question_index_max` (default 40).
   Questions only, never answers. Framing is load-bearing: the list reads as
   *"call `faq_search` for these — things you can LOOK UP, not things you
   know"*. Over the cap → **nothing** is injected and a warning logs once
   (a silently truncated list is worse than none: the model treats it as
   exhaustive). Suppressed while `faq_search` is disabled in `enabled_tools`
   (never advertise a tool the model can't call).

4. **`faq_search`'s docstring is rewritten business-agnostic** — it names the
   retrieval intent ("ANY question about this business, product or project…
   you do NOT know these details from training") instead of a shop's noun
   list. This is the cheaper fix that helps every deployment regardless of
   the index knob.

## Consequences

**Positive**

- Modules can shape the agent's prompt without core edits — the same
  "drop a folder in" extension story as tools, tables and admin routes.
- Grounding improves precisely where it's weakest: small self-hosted models
  that under-call tools. Default-off means zero cost until an operator opts in.
- The orchestrator is module-agnostic again at prompt-assembly time.

**Negative / trade-offs**

- The index costs tokens **every turn** while enabled (~350 tokens at 19
  entries) and scales linearly — hence opt-in + cap, but an operator can
  still enable it on a 40-entry FAQ and pay for it.
- **Anchoring risk:** a literal list can make the model snap a nuanced
  question onto the nearest listed one, or refuse a KB-answerable question
  phrased differently (retrieval is semantic; a list is not).
- Fragments run on every turn: each hook that hits the DB adds latency
  (memory retrieval already did; new modules must keep fragments cheap).
- Fragment order is fixed (discovery order); modules can't prioritise
  themselves. Acceptable until a real conflict shows up.

## Alternatives considered

- **Fix it in each deployment's system prompt** — worked on
  `gemini-3.5-flash-lite`, but the fix lives in one deployment and leans on
  model obedience; the stack ships the defaults.
- **Always-on index** — indefensible at 200 entries; the FAQ is otherwise
  retrieved only when relevant.
- **Truncate the index at the cap** — rejected: the model treats a listed
  subset as exhaustive and denies the rest. All-or-nothing is honest.
- **A generic prompt-pipeline/middleware API** (priorities, sections,
  templating) — premature for one producer; the minimal hook mirrors the
  contract's existing optional hooks and can grow later.

## Follow-ups (out of scope)

- Semantic pre-filter: inject only the questions relevant to the current
  turn (the query is embedded anyway for memory retrieval).
- Fragment ordering/priority if two modules ever fight over placement.
- A tool-routing eval harness (Sprint 15.3 introduces a manual 10-question
  eval; automate it and A/B the index on/off per model).
