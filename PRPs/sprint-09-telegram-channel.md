# PRP: Sprint 9 — Telegram Channel

> **Version:** 1.0
> **Created:** 2026-06-12
> **Status:** Completed — accepted by Willy 2026-06-13. Live e2e: Telegram
> inbound text + photo + voice note → agent reply (multimodal), MarkdownV2
> rendering, human-mode silence; WhatsApp markdown after the i18n fix. Shipped
> as **v0.2.0** across the stack + `chasqui` 0.2.0 on PyPI (verified
> `uvx chasqui@0.2.0 new --channels whatsapp,telegram`). Decisions: ADR-006
> (Telegram lib + webhook), ADR-007 (canonical Markdown rendering).

---

## Goal

Add **Telegram** as the second channel — a new `chasqui-stack/telegram`
gateway (submodule, sibling of `whatsapp/`) that speaks the exact same
canonical contract (`docs/ARCHITECTURE.md` §5) the core already exposes.

The end state: a user messages a Telegram bot, the agent answers from the
same core (same memory, same FAQ-RAG, same tools), an operator can take over
from the handoff inbox and reply over Telegram — and **the core has no idea
Telegram exists**. The only core change is one `.env` var.

This is the sprint that turns *"channel-agnostic"* from an architectural
claim into a running, demonstrable second channel. If it lands small, the
seam is proven.

## Why

- **Proves the architecture.** Sprints 0–8 built a core whose entire premise
  (`docs/ARCHITECTURE.md` §2.2, §5) is that channels are thin, swappable
  adapters. Until a second channel exists, that's untested theory. Telegram
  is the cheapest possible falsification test.
- **Lowers the barrier to try Chasqui.** WhatsApp onboarding needs a Meta
  developer app, a business, phone-number verification, and a 24h-window
  dance (`docs/WHATSAPP-SETUP.md`). A Telegram bot token comes from
  **@BotFather in two minutes**, no business account, no review. For someone
  evaluating the stack, "does it actually work end-to-end" becomes a
  five-minute test instead of a Meta-app afternoon.
- **It's on the public roadmap.** README + landing both list "Telegram
  channel — a second gateway speaking the same contract" as the next step.
- **Telegram has no 24h window.** Bots can message a user who has started the
  bot at any time — so the handoff/outbound path is *simpler* than WhatsApp
  (no `WINDOW_EXPIRED`), which makes it a good first second-channel.

## What

A stateless FastAPI gateway, structurally a sibling of `whatsapp/`, that:

1. Receives Telegram webhook updates (text, photo, voice/audio, document,
   callback-query buttons).
2. Normalizes each to the canonical inbound message (`channel: "telegram"`)
   and `POST`s the core's `/ingest`. Media is downloaded via the Bot API
   (`getFile` → file path → download) and inlined as a base64 `data:` URI in
   `media_url` — the same pattern WhatsApp uses, for the same reason (the
   channel-agnostic core can't fetch a channel-private file URL).
3. Renders the canonical response back to Telegram (`sendMessage` /
   `sendPhoto` / `sendVoice` / `sendDocument`). An empty `messages` list is
   silence (human-mode) — render nothing.
4. Exposes `POST /send` (ADR-004) — the canonical outbound contract, mirror
   of `/ingest`, same `INTERNAL_API_KEY` — so the handoff inbox can reply
   over Telegram. No 24h window → no `WINDOW_EXPIRED`.

Plus: the core wired to know the gateway's URL (one `.env` var), the CLI
wizard optionally asking for a Telegram token, a setup doc, and docs/ADRs
closing the sprint.

### Success Criteria

- [ ] New repo `chasqui-stack/telegram` exists, added here as a submodule at
      `telegram/`, with `AGENTS.md` + `CLAUDE.md` symlink, `README.md`,
      `.env.example` (no secrets), `Makefile` (`make dev`), `pyproject.toml`,
      `Dockerfile`, mirroring the `whatsapp/` layout.
- [ ] **Inbound:** a real text message to the bot produces an agent reply in
      the same Telegram chat, persisted under one canonical conversation.
- [ ] **Inbound media:** a photo and a voice note are inlined as base64
      `data:` URIs and the agent responds to their content (multimodal turn).
- [ ] **Outbound silence:** when the conversation is in `human` mode, the bot
      stays quiet (core returns empty `messages`).
- [ ] **Handoff:** an operator reply from the admin inbox is delivered over
      Telegram via `POST /send` (text + at least one media type).
- [ ] **Core is untouched except config:** the only core diff is adding
      `channel_telegram_send_url` to `config.py` + `.env.example`. No channel
      SDK, no `if channel == "telegram"` branch (verified by grep).
- [ ] Webhook is set in dev via ngrok and documented; `GET /health` returns
      ok; the gateway acks Telegram fast (returns 200 immediately, processes
      the core round-trip async).
- [ ] `docs/TELEGRAM-SETUP.md` exists (@BotFather → token → webhook), README
      roadmap/architecture updated, **ADR-006** records the library +
      webhook-integration decision.
- [ ] Gateway unit tests (pure payload builders + send mapping) pass; an e2e
      acceptance run is recorded in this PRP's status.

---

## All Needed Context

### Documentation & References

```yaml
- file: docs/ARCHITECTURE.md
  why: §3.1 (gateway is a stateless adapter), §5 (canonical inbound contract),
       §5.1 + ADR-004 (POST /send outbound seam), §9 (future channel repos),
       §10 (identity — Telegram's analog of BSUID is the chat/user id).

- file: docs/design/adr-004-conversation-mode-outbound-send.md
  why: The outbound contract this gateway must mirror; error-code semantics.

- file: whatsapp/app/main.py
  why: THE template. FastAPI app + lifespan CoreClient, ack-fast background
       dispatch (_dispatch / _background_tasks), POST /send with the
       INTERNAL_API_KEY guard, GET /health. Copy the shape, swap PyWa for
       the Telegram webhook + Bot API.

- file: whatsapp/app/handlers/message_handlers.py
  why: The pure-builder pattern to mirror — `payload_from_*` functions
       (unit-testable, no I/O) + one `process_update` that does the network
       round-trip + `_reply_canonical` that renders the core's response.
       Copy the structure verbatim; change only the field extraction.

- file: whatsapp/app/services/core_client.py
  why: Copy almost as-is — ingest() + notify_status() + close(). Telegram
       doesn't need notify_status for a 24h window, but late send failures
       still exist; keep it.

- file: whatsapp/app/services/sender.py
  why: The /send rendering to mirror — SendRequest/SendContact/SendMessage
       Pydantic models, _decode_data_uri (base64 data: → bytes), SendError
       with contract codes, type dispatch. Telegram version is SIMPLER:
       no ReEngagementMessage/WINDOW_EXPIRED, no wa_id requirement.

- file: whatsapp/app/services/media.py
  why: media_to_data_uri pattern + MAX_MEDIA_BYTES cap + never-raise posture.
       Telegram download is getFile → download path, but the data: URI output
       and size-cap discipline are identical.

- file: whatsapp/app/core/config.py
  why: Settings shape (pydantic-settings, .env). Telegram needs:
       telegram_bot_token, telegram_webhook_secret, core_url,
       internal_api_key, port, sentry_dsn, optional telegram_webhook_url (dev).

- file: core/app/services/channel_send.py
  why: PROOF the core needs no code: send_url_for() does
       getattr(settings, f"channel_{channel}_send_url"). Channel "telegram"
       resolves automatically once channel_telegram_send_url exists.

- file: core/app/core/config.py  (line ~92)
  why: The ONE core change — add `channel_telegram_send_url: str | None = None`
       next to `channel_whatsapp_send_url`.

- url: https://core.telegram.org/bots/api
  why: Bot API reference — Update object, getUpdates vs webhook, setWebhook
       (+ secret_token header X-Telegram-Bot-Api-Secret-Token), sendMessage,
       sendPhoto, sendVoice/sendAudio, sendDocument, getFile, callback_query.

- url: https://docs.python-telegram-bot.org/  (v21+)
  why: PTB's Bot class for typed send + file download, and Update.de_json for
       parsing webhook payloads. We use the Bot client, NOT PTB's Application
       server (FastAPI owns the route) — see ADR-006 below.
```

### Current vs. desired structure

```
chasqui-stack/chasqui          # parent (this repo)
├── core/      → submodule
├── admin/     → submodule
├── whatsapp/  → submodule   (the template)
└── telegram/  → submodule   ★ NEW — chasqui-stack/telegram
```

```bash
telegram/                     # mirror whatsapp/ layout
├── app/
│   ├── main.py               # FastAPI app, POST /webhook + POST /send + /health, ack-fast dispatch
│   ├── core/
│   │   └── config.py         # pydantic-settings: telegram_bot_token, telegram_webhook_secret, core_url, ...
│   ├── handlers/
│   │   └── message_handlers.py  # payload_from_text/photo/voice/document/callback + process_update + _reply_canonical
│   └── services/
│       ├── core_client.py    # ~copy of whatsapp's
│       ├── media.py          # getFile → download → data: URI (size-capped, never raises)
│       └── sender.py         # canonical /send → Bot API (sendMessage/Photo/Voice/Document)
├── tests/
│   ├── test_handlers.py      # pure payload builders
│   ├── test_send.py          # canonical → Bot API mapping (mock Bot)
│   └── test_media.py         # data: URI shaping + size cap
├── .env.example  .gitignore  AGENTS.md  CLAUDE.md→AGENTS.md
├── Dockerfile  Makefile  pyproject.toml  README.md  LICENSE
```

Core diff (the entire core footprint of this sprint):

```python
# core/app/core/config.py — next to channel_whatsapp_send_url
channel_telegram_send_url: str | None = None
```
```bash
# core/.env.example
# CHANNEL_TELEGRAM_SEND_URL=http://localhost:8001/send
```

### Known Gotchas & Project Conventions

```python
# 1. ACK FAST. Telegram retries a webhook that doesn't 200 quickly and can
#    disable it after repeated failures. Mirror whatsapp/main.py: parse the
#    update, dispatch the core round-trip as a fire-and-forget asyncio task
#    (the _dispatch / _background_tasks pattern), return 200 immediately.

# 2. IDENTITY (ARCHITECTURE §10 analog). Telegram has no BSUID. Canonical
#    contact.external_id = the Telegram **chat id** (message.chat.id) — that's
#    what you send replies to and it's stable per conversation. Keep user id /
#    username in metadata. wa_id is null for Telegram (it's WhatsApp-specific);
#    the /send contract's wa_id field is simply unused — DO NOT require it
#    (the NO_WA_ID error is WhatsApp-only).

# 3. NO 24h WINDOW. Telegram lets a bot message any user who has /start-ed it,
#    anytime. So the outbound path drops WhatsApp's whole ReEngagementMessage →
#    WINDOW_EXPIRED branch. Failures collapse to SEND_FAILED (e.g. 403 "bot was
#    blocked by the user"). Map 403 to a clear code if useful, else SEND_FAILED.

# 4. WEBHOOK SECRET, not signature. WhatsApp uses app_secret HMAC; Telegram
#    uses a shared secret you pass to setWebhook(secret_token=...) and it
#    echoes back in the `X-Telegram-Bot-Api-Secret-Token` header on every
#    call. Verify that header == settings.telegram_webhook_secret; 401 if not.
#    This is the gateway's authenticity check — keep it PRIVATE/gitignored.

# 5. MEDIA download is two-step. getFile(file_id) → file_path, then download
#    https://api.telegram.org/file/bot<TOKEN>/<file_path>. Reuse media.py's
#    data: URI shape + MAX_MEDIA_BYTES cap + never-raise (return None → the
#    turn proceeds text-only). Telegram is MORE permissive than Meta on audio
#    (OGG/Opus voice notes are native) — likely no ffmpeg transcode pain, but
#    confirm what the LLM accepts; if a format is rejected, the whatsapp
#    _transcode_to_mp3 helper is the proven fallback to port.

# 6. ENGLISH-ONLY codebase (parent CLAUDE.md). Code, comments, logs, API error
#    details are English. The only user-facing literals (error/unsupported
#    replies) live in the gateway like whatsapp/handlers — but prefer to keep
#    user-language localization in the core's system prompt; the gateway's
#    own literals (e.g. "technical hiccup") should be minimal and ideally
#    .env-configurable like the core's FALLBACK_REPLY if we add any.

# 7. STATELESS. No DB, no business logic in the gateway (parent CLAUDE.md +
#    whatsapp/AGENTS.md "Don't"). If it restarts, nothing is lost.

# 8. PORT. whatsapp/ defaults to 8000. Default telegram/ to **8001** so both
#    gateways run locally at once; core stays 8090, admin 5191.

# 9. SECRETS. telegram_bot_token + telegram_webhook_secret are PRIVATE —
#    gitignored .env, NEVER copied to .env.example with real values, never
#    printed. INTERNAL_API_KEY is the SAME shared value as core/whatsapp.
```

---

## Implementation Blueprint

### ADR-006 (write first — it's a real decision)

Record **which Telegram library + how it integrates with FastAPI**, because
it's non-obvious and shapes the whole gateway. Recommendation to ratify:

> Use **python-telegram-bot (PTB) v21+** for its `Bot` client (typed send
> methods, `getFile`/download, `Update.de_json` parsing) but **NOT** PTB's
> `Application`/built-in webhook server. FastAPI owns the `POST /webhook`
> route (consistent with the stateless, ack-fast, FastAPI-native posture of
> the WhatsApp gateway); we hand the raw JSON to `Update.de_json` and dispatch
> handlers ourselves. Rationale: keeps one HTTP server (FastAPI), keeps the
> ack-fast background-task pattern identical to whatsapp/, avoids running PTB's
> updater/job-queue machinery we don't need in a stateless adapter.
> Alternative considered: `aiogram` (also async/typed) — equally viable;
> PTB chosen for maturity + closest mental model to the existing gateway.

Decide token-vs-stateless and any media-transcode stance here too if they
turn out non-obvious during the build.

### Tasks (in execution order)

```yaml
Task 0: Create the repo + submodule
  - CREATE GitHub repo chasqui-stack/telegram (public, Apache-2.0).
  - Bootstrap by copying whatsapp/ layout (≈40% reusable verbatim:
    core_client.py, the main.py skeleton, config.py shape, Makefile,
    Dockerfile, .gitignore, AGENTS.md structure).
  - Add as submodule here: telegram/ → chasqui-stack/telegram. Commit the
    pointer in the parent (submodules pin commits — parent CLAUDE.md).

Task 1: config.py
  - telegram_bot_token (required), telegram_webhook_secret (required),
    core_url (default http://localhost:8090), internal_api_key (default ""),
    port (default 8001), sentry_dsn (default ""),
    telegram_webhook_url (optional — dev/ngrok, used to call setWebhook).
  - MIRROR: whatsapp/app/core/config.py.

Task 2: core_client.py
  - Copy whatsapp/app/services/core_client.py near-verbatim (ingest +
    notify_status + close). Keep notify_status for late send failures.

Task 3: media.py — inbound media → data: URI
  - getFile(file_id) via the Bot client → file_path → download bytes →
    data:<mime>;base64,<...>. Reuse MAX_MEDIA_BYTES cap + never-raise.
  - MIRROR: whatsapp/app/services/media.py.

Task 4: handlers/message_handlers.py
  - Pure builders (NO I/O, unit-testable), one per inbound type:
    payload_from_text, payload_from_photo (caption→text), payload_from_voice/
    audio, payload_from_document, payload_from_callback (callback_query →
    canonical "button"). channel="telegram", contact.external_id=chat.id.
  - process_update(): typing action (sendChatAction "typing") → core.ingest →
    _reply_canonical. _reply_canonical renders core messages back via the
    sender. Media-bearing inbound: after building the pure payload, fill
    media_url = await media_to_data_uri(file_id) (mirror whatsapp handlers).
  - MIRROR: whatsapp/app/handlers/message_handlers.py (same shape).

Task 5: sender.py — canonical POST /send → Bot API
  - SendRequest/SendContact/SendMessage Pydantic models (copy from whatsapp,
    drop wa_id requirement). _decode_data_uri (copy verbatim). SendError with
    contract codes: SEND_FAILED, UNSUPPORTED_TYPE, INVALID_MEDIA (NO blocked
    /WINDOW_EXPIRED unless we map Telegram 403 to a code). _dispatch maps
    type → Bot.send_message / send_photo / send_voice (ogg) or send_audio /
    send_document. Address = contact.external_id (chat id).
  - MIRROR: whatsapp/app/services/sender.py (simpler — no 24h window).

Task 6: main.py — FastAPI app
  - lifespan creates CoreClient + the Bot client; on startup, if
    telegram_webhook_url is set, call setWebhook(url, secret_token=...).
  - POST /webhook: verify X-Telegram-Bot-Api-Secret-Token header (401 else);
    Update.de_json(body); route by content type; _dispatch the handler as a
    background task; return {"ok": true} immediately (ACK FAST).
  - POST /send: INTERNAL_API_KEY header guard (copy whatsapp), call
    send_canonical, map SendError → HTTPException with {code,message}.
  - GET /health → {"status":"ok","service":"chasqui-telegram"}.
  - Sentry init when DSN set.
  - MIRROR: whatsapp/app/main.py.

Task 7: core wiring (the ONLY core change)
  - MODIFY core/app/core/config.py: add channel_telegram_send_url.
  - MODIFY core/.env.example: commented CHANNEL_TELEGRAM_SEND_URL example.
  - VERIFY: grep core/app for "telegram" → only the config field, no branch.

Task 8: CLI wizard (chasqui-stack/cli — separate repo, ADR-005)
  - Add optional Telegram step to `chasqui new`: ask for bot token (skippable
    like WhatsApp creds), write telegram/.env + set CHANNEL_TELEGRAM_SEND_URL
    in core/.env when provided. Pin the new telegram submodule tag in the
    stack the CLI fetches. (Coordinate via the cli repo's AGENTS.md release
    ceremony — do NOT tag out of order.)

Task 9: docs
  - CREATE docs/TELEGRAM-SETUP.md: @BotFather → token → (dev) ngrok + the
    gateway auto-setWebhook, or manual setWebhook curl; how to find your
    chat by messaging the bot; troubleshooting (webhook secret, getWebhookInfo).
  - MODIFY README.md: architecture mermaid (TG solid, not dashed), roadmap
    (Telegram → done / move web widget up), services table (add telegram row),
    Quickstart (telegram make dev on :8001).
  - MODIFY docs/ARCHITECTURE.md §3/§9 if wording implies WhatsApp-only.
  - CREATE docs/design/adr-006-telegram-channel.md (the decision above).
  - MODIFY landing (chasqui-website): flip Telegram from roadmap-dashed to live
    in the architecture diagram + roadmap (EN + ES i18n.ts). [separate repo]

Task 10: tests
  - test_handlers.py: each payload_from_* builds a correct canonical dict.
  - test_send.py: each canonical type → the right Bot method + args (mock Bot).
  - test_media.py: data: URI shape + size-cap returns None.
  - MIRROR whatsapp/tests/*; plain `uv sync` installs test deps.
```

### Validation gates

```bash
# Gateway (in telegram/)
uv sync && uv run pytest          # unit tests green
make dev                          # boots on :8001, GET /health ok

# Core unchanged behavior
cd core && make test              # still green; grep app for 'telegram' = 1 hit (config)

# E2E acceptance (record result in this PRP's Status)
# 1. ngrok http 8001 → set telegram_webhook_url → restart → getWebhookInfo ok
# 2. Text the bot → agent replies in the same chat.
# 3. Send a photo + a voice note → agent responds to their content.
# 4. Flip the conversation to human mode in the admin → bot goes silent.
# 5. Reply from the handoff inbox → message arrives over Telegram (text + media).
# 6. grep proves the core never learned the word "telegram" beyond one config var.
```

---

## Why this is a small sprint

The engine already exists. Sprints 0–7 built memory, RAG, the orchestrator,
multimodal turns, the handoff inbox and the outbound seam — all
channel-agnostic. This sprint is **translation + plumbing**: roughly 40% is
copied from `whatsapp/` verbatim (core_client, the main.py skeleton, config
shape, sender/media structure), the inbound/outbound mapping is mechanical,
and the outbound path is *simpler* than WhatsApp's (no 24h window). The core
footprint is a single `.env`-backed config field. Estimated materially
smaller than the core sprints (6/7).

---

## Out of scope (explicit)

- Inline keyboards / rich Telegram-native UI beyond what maps to the existing
  canonical `button` type. (The contract carries buttons; fancy Telegram
  layouts are a later polish.)
- Telegram groups / channels — Chasqui is 1:1 contact ↔ conversation (§6).
  Bot-in-a-group is a separate design question.
- Payments, web-app buttons, location/contact sharing — future, behind the
  contract.
- Instagram / web widget — their own sprints.
