# ADR-006: Telegram channel — library + webhook integration

> **Status:** Accepted — 2026-06-13
> **Context:** Sprint 9 adds Telegram as the second channel
> (`chasqui-stack/telegram`, sibling of `whatsapp/`). Two non-obvious choices
> shape the whole gateway and aren't dictated by the canonical contract: (1)
> which Telegram library, and (2) how it integrates with FastAPI. PRP:
> [`PRPs/sprint-09-telegram-channel.md`](../../PRPs/sprint-09-telegram-channel.md).

## Decision

1. **Library: [python-telegram-bot](https://docs.python-telegram-bot.org/)
   (PTB) v21+**, used as a **Bot API client only** — `Bot.send_*`,
   `Bot.get_file` / file download, and `Update.de_json` for parsing webhook
   payloads.
2. **FastAPI owns the webhook route.** The gateway exposes its own
   `POST /webhook`; we hand the raw JSON to `Update.de_json` and dispatch
   handlers ourselves. We do **not** run PTB's `Application` / `Updater` /
   job-queue machinery.

## Rationale

1. **One HTTP server, one mental model.** The WhatsApp gateway is a FastAPI
   app where PyWa registers itself on the existing `app`; the gateway stays a
   plain FastAPI service (`/webhook`, `/send`, `/health`). Mirroring that with
   Telegram keeps both gateways structurally identical — `main.py`, the
   ack-fast background-task dispatch, the `POST /send` contract guard, Sentry
   init, the `Makefile`/`Dockerfile` — so ~40% is copied verbatim and a
   contributor who knows one knows the other.
2. **Stateless adapter, by charter.** The gateway has no database and no
   business logic (ARCHITECTURE §3.1; `whatsapp/AGENTS.md`). PTB's
   `Application` brings an updater, a job queue, persistence hooks and its own
   event loop — infrastructure a stateless translator doesn't want and would
   have to fight to keep quiet.
3. **Ack-fast is preserved cleanly.** Telegram retries a slow webhook and can
   disable it; the gateway must return 200 immediately and process the core
   round-trip asynchronously. Owning the route lets us reuse the exact
   `_dispatch` / `_background_tasks` pattern from `whatsapp/main.py` instead of
   bending PTB's handler lifecycle to it.
4. **PTB earns its place as a client.** It's the most mature Python Bot API
   wrapper: typed `send_message`/`send_photo`/`send_voice`/`send_document`,
   `get_file` + download, and `Update.de_json` give us the inbound parsing and
   outbound rendering for free — the parts that would be tedious and
   error-prone to hand-roll against the raw HTTP API.

## Alternatives considered

- **aiogram** — also async, also typed, equally capable for this job. Rejected
  only on *closest mental model*: PTB's `Bot`-as-client maps most directly onto
  how PyWa is used in the sibling gateway. Not a quality judgment; revisiting is
  cheap because the library is confined to `services/` (sender + media) behind
  the canonical contract.
- **Raw HTTP to `api.telegram.org` (httpx only)** — zero dependency, but we'd
  re-implement multipart media upload, `getFile` download, and Update typing.
  Not worth it.
- **PTB's built-in webhook server (`Application.run_webhook`)** — would mean two
  servers or surrendering the route to PTB, breaking the FastAPI-native, single
  -server symmetry with `whatsapp/`. Rejected per points 1–3.

## Consequences

- `pyproject.toml` pins `python-telegram-bot>=21.0`; the dependency lives only
  in `app/services/` (sender, media) and the webhook parse in `main.py`.
- Webhook authenticity is the `X-Telegram-Bot-Api-Secret-Token` header set via
  `setWebhook(secret_token=…)` (Telegram's analog of Meta's `app_secret` HMAC),
  verified against `TELEGRAM_WEBHOOK_SECRET` — see the gateway AGENTS.md.
- The core is untouched beyond one config var (`channel_telegram_send_url`);
  `channel_send.send_url_for()` already resolves it generically.

## Revisit trigger

If we ever want push-style features PTB's `Application` specializes in (polling
fallback, conversation handlers, the job queue), re-evaluate — but for a
stateless webhook adapter, owning the route is the simpler, lower-surprise path.
