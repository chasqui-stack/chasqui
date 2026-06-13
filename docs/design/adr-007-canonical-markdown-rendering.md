# ADR-007: Canonical text is Markdown; gateways render per platform

> **Status:** Accepted — 2026-06-13
> **Context:** First live Telegram test (Sprint 9) showed the agent's
> `**bold**` and bullet lists rendered as literal asterisks — the gateway sent
> the LLM's Markdown verbatim with no platform formatting. WhatsApp had the
> same latent bug (its syntax is single-`*` bold, `_italic_`). The question:
> where does channel-specific text formatting belong?

## Decision

**The canonical `message.text` (ARCHITECTURE §5, both directions) is standard
Markdown. Each gateway renders it to its platform's formatting; the core never
formats for a channel.**

- The LLM/agent emits standard Markdown (`**bold**`, `*italic*`, `` `code` ``,
  lists, `[label](url)`).
- Telegram gateway → MarkdownV2 (`parse_mode="MarkdownV2"`) via
  `telegramify-markdown`, with a plain-text fallback if Telegram rejects the
  entities.
- WhatsApp gateway → WhatsApp syntax (`*bold*`, `_italic_`, `~strike~`) via a
  small converter (`app/services/formatting.py`).

## Rationale

1. **Formatting is presentation — it lives in the adapter.** This is the exact
   same principle as media transcoding (ADR-003): the core ships one canonical
   representation, the gateway adapts it to the channel. Markdown→platform is
   formatting transcoding.
2. **Keeps the core channel-agnostic** (the system's core invariant: "the core
   never knows a channel exists"). The rejected alternative — passing the
   channel into the core so the agent formats per-platform — would put
   `if telegram / if whatsapp` branches and N markup dialects into the core,
   the precise coupling the architecture exists to avoid.
3. **One markup in, N renderers out** — the same fan-out shape as the rest of
   the contract. A new channel implements one renderer; the core is untouched.
4. **Robustness at the edge.** MarkdownV2 escaping is famously fiddly; the
   gateway owns a battle-tested converter and a plain-text fallback so a
   malformed entity never drops a reply — a concern that has no place in the
   core.

## Consequences

- New channels MUST render canonical Markdown to their platform (documented in
  each gateway's `AGENTS.md` "Don't" list and ARCHITECTURE §5).
- The default system prompt's "no heavy Markdown" hint is now a soft
  preference, not a correctness mechanism — formatting is handled regardless.
- Code spans / fenced blocks are passed through (both platforms render them
  natively); per-language syntax highlighting is out of scope.

## Revisit trigger

If a future channel has no rich-text concept at all (e.g. SMS), its renderer
strips Markdown to plain text — still a gateway concern, no core change.
