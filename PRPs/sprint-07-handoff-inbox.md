# PRP: Sprint 7 — Human Handoff Inbox & Leads

> **Version:** 1.1
> **Created:** 2026-06-10
> **Status:** Completed — accepted by Willy 2026-06-11 (full e2e: handoff →
> silence → panel replies with text/emoji/image/document/voice over WhatsApp
> → delivery-status badges → resume bot → leads)

---

## Goal

Close the human-in-the-loop. Today `human_handoff` / `lead_capture` only write
flags into `conversation_state` JSONB **that nothing reads**: the bot never
goes quiet, nobody is notified, leads live invisible. After this sprint:

1. **Handoff silences the agent** — `conversations.mode: "agent" | "human"`,
   checked FIRST by ingest (human → persist inbound, no turn, no reply).
2. **Operators reply from the panel through the channel** — the canonical
   outbound contract `POST /send` (the mirror of `/ingest`) implemented by
   the WhatsApp gateway, resolved per channel from `.env`.
3. **Someone finds out** — webhook (`NOTIFY_WEBHOOK_URL`) **and/or SMTP
   email** (Brevo or any relay — Willy's addition, 2026-06-10) fired on
   handoff, best-effort.
4. **Leads are first-class records** — module-owned `leads` table with
   config-driven required/extra fields, visible in a `/leads` page.

Last product sprint of the first release. Design locked in
[ADR-004](../docs/design/adr-004-conversation-mode-outbound-send.md).

## Why

- The Sprint 3 tools were decorative by design ("flags nobody reads") — this
  is the sprint that makes them real.
- It's **channel architecture, not a feature patch** (Willy's framing):
  future channels must know when a conversation is in human mode (they
  don't need to — silence is an empty canonical response) and must accept
  admin-initiated outbound (`/send`). One seam, N channels.
- Lead capture with configurable fields is the Sprint 5 auto-form contract
  proving itself again — operators tune `require_email` / `extra_fields`
  without anyone touching code.

## What

### Part A — Core (`chasqui-stack/core`)

| Piece | Behavior |
|---|---|
| Migration 006 | `conversations.mode` varchar(8) NOT NULL default `'agent'` + index; `leads` table (module-owned: id, contact_id FK, name, email, phone, interest, notes, extra JSONB, created_at). One migration, both changes. |
| Ingest short-circuit | After get-or-create conversation: `mode == 'human'` → persist inbound (media included, ADR-003 path unchanged), bump `updated_at`, return `IngestResponse(messages=[])`. No turn, no memory extraction. |
| `human_handoff` v2 | Sets `conversation.mode = 'human'` + audit JSONB (`reason`, `at`) + dispatches notifications (background task). |
| `notify_service` | `dispatch_handoff(...)`: fire-and-forget; webhook POST (httpx, 10s timeout) + SMTP email (stdlib `smtplib` via `asyncio.to_thread`; 587 STARTTLS / 465 SSL). Each independent, each best-effort (log on failure). |
| `channel_send` service | `send_text(contact, text)` → resolves `CHANNEL_<channel>_SEND_URL` from settings, POST canonical send payload with `X-Internal-API-Key`. Raises `ChannelSendError(code, message)` mapped from the gateway's error contract. |
| `PUT /admin/contacts/{id}/mode` | Body `{mode}`. Get-or-create conversation, flip mode. Resuming agent mode stamps `conversation_state.handoff.resolved_at` (audit kept, `requested` → false). Returns the new mode. |
| `POST /admin/contacts/{id}/messages` | Body `{text}`. 409 if `mode != 'human'` (the agent owns agent-mode replies). **Send-then-persist**: gateway confirms → persist `direction='out'`, `meta.sent_by`/`sent_by_email`. Gateway failure → 502 `{code}` passthrough (`WINDOW_EXPIRED` stays meaningful), nothing persisted. |
| `GET /admin/contacts` | Grows `mode`, `handoff_reason`/`handoff_at`, `last_inbound_at` per item; `?mode=` filter; default sort: human-mode first, then `updated_at` desc. Detail gets the same fields. |
| `lead_capture` v2 | Inserts a `Lead` row. Config-driven (`config_key="lead_capture"`, flat schema): `require_email`, `require_phone` (known contact `wa_id` satisfies phone), `extra_fields` (comma-separated). Missing required data → tool returns "ask the user for X" instead of saving. |
| Leads admin API | Module-owned: `GET /admin/modules/handoff/leads` (`{items,total}` + limit/offset + `?contact_id=`). The panel's `/leads` page consumes it. |
| Settings | `channel_whatsapp_send_url`, `notify_webhook_url`, `smtp_host/port/user/password/from`, `notify_email_to`. |

### Part B — WhatsApp gateway (`chasqui-stack/whatsapp`)

- `POST /send` (`X-Internal-API-Key` auth, same secret as core's `/ingest`):
  canonical payload → PyWa `send_message(to=wa_id, text)`.
- Errors per ADR-004: 401 bad key · 400 `NO_WA_ID` · 422 `UNSUPPORTED_TYPE`
  (text-only this sprint) · 502 `WINDOW_EXPIRED` (Meta 131047 /
  re-engagement) or `SEND_FAILED`.
- Tests with PyWa stubbed (existing `tests/` pattern).

### Part C — Admin (`chasqui-stack/admin`)

- Conversations list: 🚨 human-mode badge, "needs human" filter tab
  (`?mode=human`), attention-first order comes from the API.
- Conversation detail: "Take over" / "Resume bot" button; in human mode a
  composer POSTs through the new endpoint; messages poll
  (`refetchInterval` ~5s) while open.
- **24h-window indicator** on the composer: countdown from
  `last_inbound_at` ("window closes in 3h"), disabled + explanation when
  expired; `WINDOW_EXPIRED` send errors get their own message.
- `/leads` page: table (name, contact, email/phone, interest, extra, date)
  + sidebar entry.
- Dashboard: "waiting for human" count card (reuses
  `GET /admin/contacts?mode=human&limit=1` → `total`).
- All strings via `t()` (en/es), key-parity test stays green.

### Part D — Parent

- ADR-004 (done) · ARCHITECTURE §5 grows the outbound contract · compose
  wires `CHANNEL_WHATSAPP_SEND_URL` for the gateway profile · README/AGENTS
  touch-ups · sprint doc ticked · submodule bumps.

### Success criteria

- [x] "quiero hablar con una persona" → bot confirms and **goes silent**;
      conversation badged in the admin; webhook/email fired (when configured).
- [x] Operator reply from the panel arrives on the user's WhatsApp; thread
      shows it as outbound with the admin's identity.
- [x] "Resume bot" → the agent answers the next message again.
- [x] Lead with configurable required/extra fields → row in `/leads`.
- [x] A second gateway could implement `/send` + ride the mode semantics with
      zero core changes (contract documented).
- [x] `make test` (core, 124) + gateway pytest (23) + admin build/lint/test green.

---

## All Needed Context

```yaml
- file: docs/design/adr-004-conversation-mode-outbound-send.md  # the decision — read first
- file: docs/sprints/sprint-07-handoff-inbox.md                 # task list (internal)
- file: core/app/services/ingest_service.py                     # the short-circuit point
- file: core/app/modules/handoff/__init__.py                    # v1 tools to upgrade
- file: core/app/modules/faq/                                   # full module contract reference (models/admin/config)
- file: core/app/controllers/admin/contacts.py                  # list queries to extend (no N+1)
- file: core/alembic/versions/004_faq_entries.py                # module-table migration pattern
- file: whatsapp/app/main.py                                    # where /send mounts (wa client lives here)
- file: whatsapp/tests/test_handlers.py                         # gateway test pattern
- file: admin/src/pages/ConversationDetailPage.tsx              # composer + polling land here
```

### Key decisions (on top of ADR-004)

1. **Send-then-persist** for operator messages (deviation from the sprint
   draft's "persist anyway"): a failed send persists nothing — the text
   stays in the composer, the thread never lies about delivery.
2. **Notifications are fire-and-forget background tasks** (kept-referenced
   `asyncio.Task` set, the gateway's `_dispatch` pattern) — the LLM turn
   never waits on Slack or an SMTP handshake.
3. **SMTP via stdlib `smtplib`** — zero new dependencies; "where is your
   relay" (Brevo/Mailgun/SES/Gmail), never "which provider" (ADR-002 shape).
4. **Leads endpoint is module-owned** (`/admin/modules/handoff/leads`, not a
   core `/admin/leads`): the table is the module's, the routes are the
   module's — same as faq. The panel page is named "Leads" regardless.
5. **`require_phone` is satisfied by a known `wa_id`** — on WhatsApp the
   number is usually implicit; the agent only asks when the contact has none
   (e.g. BSUID-only or another channel).
6. **`extra` lead answers travel as a dict tool arg** — the configured
   `extra_fields` names are dynamic, so they can't be static tool params;
   the docstring instructs the model, the tool validates the keys.
7. **Mode flip endpoints use get-or-create** for the conversation — taking
   over a contact that never got an agent reply must not 500.
8. **Window state is computed client-side** from `last_inbound_at` (24h,
   WhatsApp-only advisory); the server never blocks a send for it — Meta is
   the authority (`WINDOW_EXPIRED` mapping is the honest fallback).

### Known gotchas

1. **JSONB mutation**: always reassign `conversation.conversation_state`
   (never mutate in place) — SQLAlchemy change detection (v1 lesson).
2. **Meta's 131047 can be asynchronous** (status webhook after a 200): the
   synchronous `WINDOW_EXPIRED` mapping is best-effort; the composer
   countdown is the real prevention. Templates → backlog.
3. **`session.refresh()` does not autoflush** — flush before refresh
   (proven in Sprint 5 tests).
4. **Background notify tasks in tests**: test `_send_webhook` / `_send_email`
   directly + the dispatcher with monkeypatched senders; never sleep-poll.
5. **PyWa send addressing**: `wa_id` only (no BSUID send endpoint — Sprint 2
   gotcha). `NO_WA_ID` is a real case for non-WhatsApp-originated contacts.
6. **Polling + JWT expiry**: the api-client's existing refresh flow covers
   it; keep `refetchInterval` on the query, not a manual setInterval.
7. **i18n key parity** (en/es) is vitest-enforced — add both languages or
   the suite fails.
8. **httpx test stubbing**: stub `channel_send` at the service seam in core
   tests (no real gateway); gateway tests stub the PyWa client method.

## Tasks

- [x] ADR-004 + this PRP.
- [x] Core: migration 006 (`mode` + `leads`), `Lead` model via
      `register_models()`.
- [x] Core: ingest short-circuit; `human_handoff` sets mode + notifies;
      `notify_service` (webhook + SMTP).
- [x] Core: `channel_send` + inbox endpoints (messages POST / mode PUT /
      list fields+filter+sort) + `lead_capture` v2 + leads listing.
- [x] Core tests: short-circuit, 409, send seam (stubbed), WINDOW_EXPIRED
      passthrough, lead requirements loop, leads listing, notify senders.
- [x] Gateway: `POST /send` + error mapping + tests.
- [x] Admin: list badge/filter, take-over + composer + polling, 24h
      indicator, `/leads`, dashboard card, i18n.
- [x] Docs: ARCHITECTURE outbound contract, READMEs/AGENTS, `.env.example`s,
      compose; submodule bumps; sprint doc ticked.
- [x] Manual e2e with Willy (handoff → silence → panel reply → WhatsApp →
      resume bot; lead visible in /leads; **accepted 2026-06-11**).

## Mid-sprint additions (Willy's e2e feedback, 2026-06-11)

- **Fix:** WA_PHONE_ID now passed to the PyWa client — inbound replies infer
  the sender from the update, client-initiated sends can't (SEND_FAILED bug).
- **Fix:** the open chat follows new messages when pinned near the bottom
  (own sends + polled inbound); reading older history is never yanked down.
- **Outbound media** (the ADR-004 "revisit" trigger arrived same-sprint):
  `/send` grows `type: image|document|audio` + `media_url` as base64 data URI
  (mirror of inbound) + `filename`; gateway maps PyWa send_image/document/
  audio; core validates per-type size caps (5/25/16 MB), uploads outbound
  media to the bucket (log-and-NULL — never undo a delivered message) so the
  timeline renders it. Composer: dependency-free emoji picker, file attach
  (jpg/png/webp/pdf/doc/docx/xls/xlsx), voice notes via MediaRecorder
  (`audio/mp4` preferred — Meta rejects webm; webm fallback may SEND_FAILED
  on old browsers).
