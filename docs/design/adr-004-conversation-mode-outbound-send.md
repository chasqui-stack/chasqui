# ADR-004 — Conversation mode & the canonical outbound contract (`POST /send`)

> **Status:** Accepted — 2026-06-10
> **Sprint:** 7 (human handoff inbox & leads)
> **Related:** ARCHITECTURE §5 (canonical contract), §8 (tool modules), ADR-002 (omakase posture)

## Context

Since Sprint 3, `human_handoff` and `lead_capture` only write flags into
`conversations.conversation_state` (JSONB) **that nothing reads**: the agent
never goes quiet, no operator is notified, and an admin reply has no way to
reach the user. Closing the human-in-the-loop requires three architectural
decisions that will outlive this sprint:

1. Where does "a human owns this conversation" live, and who enforces it?
2. How does the core push an operator's message **out** through a channel it
   is, by design, not allowed to know about?
3. How does anyone find out a conversation needs attention?

All three must hold for future channels (Telegram, web widget) without
per-channel logic in the core — the same constraint that shaped the inbound
contract (§5).

## Decision

### 1. Conversation mode is a real column, enforced by the core

`conversations.mode: "agent" | "human"` — a plain indexed varchar (default
`'agent'`), **not** a JSONB flag. The ingest pipeline checks it **before**
running the agent turn: `mode = 'human'` → persist the inbound message,
run **no** turn, return an **empty** canonical response (`messages: []`).

- Gateways already render "0..N reply messages" — an empty list is silence.
  **Channels inherit human mode for free, with zero changes.**
- `human_handoff` flips the mode (and keeps `reason`/`at` in
  `conversation_state.handoff` as audit trail); the admin flips it back
  ("resume bot"). The mode column is the single source of truth — the JSONB
  is history, never behavior.
- A real column means the inbox query (`WHERE mode = 'human'`) is indexed and
  honest, not a JSONB containment scan.

### 2. The canonical **outbound** contract: every gateway exposes `POST /send`

The mirror of `/ingest`. Same auth (`X-Internal-API-Key`), same canonical
shapes:

```jsonc
// core → gateway
POST /send
{
  "contact": { "channel": "whatsapp", "external_id": "<BSUID>", "wa_id": "519..." },
  "message": { "type": "text", "text": "Hola, soy Ana del equipo…" }
}
// 200 → { "status": "sent", "message_id": "<channel id>" }
// 4xx/502 → { "detail": { "code": "NO_WA_ID" | "WINDOW_EXPIRED" | "SEND_FAILED" | "UNSUPPORTED_TYPE", "message": "…" } }
```

The core resolves the gateway per channel from `.env` — **one variable per
channel**, wizard-able (`uvx chasqui new` can ask for it):

```bash
CHANNEL_WHATSAPP_SEND_URL=http://localhost:8000/send
# a future channel is one more line, zero core code:
# CHANNEL_TELEGRAM_SEND_URL=http://localhost:8001/send
```

One seam, N channels — adding a channel means implementing `/ingest`'s
producer and `/send`'s consumer, nothing else. The WhatsApp gateway
implements `/send` with PyWa addressed by `wa_id` (Meta has no BSUID send
endpoint yet — the Sprint 2 gotcha; PyWa will flip when Meta does).

**Error codes are part of the contract** because the admin UX needs them:
WhatsApp's 24-hour customer-service window means a perfectly valid send can
be rejected by Meta (error 131047). The gateway maps that to
`WINDOW_EXPIRED`; the core passes the code through; the panel explains it to
the operator instead of showing a generic 502. Codes are channel-advisory —
a channel without the concept simply never emits it. (Caveat: Meta sometimes
accepts the message and fails it *asynchronously* via a status webhook — the
synchronous mapping is best-effort; the panel's countdown computed from
`last_inbound_at` is the real prevention.)

### 3. Handoff notifications: webhook **and/or** SMTP, both optional, both best-effort

When `human_handoff` fires, the core dispatches (fire-and-forget background
task — never blocks or breaks the turn, the embeddings/storage failure
posture):

- **`NOTIFY_WEBHOOK_URL`** — POST a JSON event
  (`{event: "handoff", reason, at, contact: {…}}`). Covers Slack, Discord,
  Zapier, n8n — any relay.
- **SMTP** — a plain-text email via Python's stdlib `smtplib`
  (`asyncio.to_thread`, **zero new dependencies**). Config is "where is your
  relay", never "which provider" (the ADR-002/003 shape): `SMTP_HOST`,
  `SMTP_PORT` (587 STARTTLS / 465 SSL), `SMTP_USER`, `SMTP_PASSWORD`,
  `SMTP_FROM`, `NOTIFY_EMAIL_TO` (comma-separated). Works verbatim with
  Brevo (`smtp-relay.brevo.com:587`), Mailgun, SES, Postmark, or a Gmail app
  password.

Unset = silent (the panel's badge/counter is the baseline signal). We
deliberately do **not** adopt a provider SDK or a transactional-email
abstraction: handoff alerts are one short text message; SMTP is the
universal seam.

## Considered and rejected

- **JSONB flag for mode** — nothing enforces it (proven: it shipped in
  Sprint 3 and was decorative), unindexable for the inbox, invisible to SQL.
- **Core talks to channel SDKs for outbound** (PyWa in the core) — breaks
  the prime directive (§5: the core never knows a channel exists); every
  new channel would mean core code.
- **A message-queue between core and gateways** — infrastructure tax for a
  request/response problem; HTTP + `INTERNAL_API_KEY` is already the proven
  inbound seam.
- **WebSockets for the inbox** — polling (~5s, TanStack `refetchInterval`)
  is omakase-simple, proxy-friendly and plenty for operator chat. Revisit if
  a real-time SLA ever appears.
- **WhatsApp template messages** (to message outside the 24h window) —
  requires Meta template management UI/approval flow; out of scope. We
  *surface* the window (countdown + `WINDOW_EXPIRED`), templates → backlog.
- **Provider email SDKs (Brevo/SendGrid API)** — a dependency per provider
  for what one stdlib SMTP call does; the webhook already covers rich
  integrations.

## Consequences

- `messages` grows operator-authored rows: `direction='out'` with
  `meta.sent_by = <admin id>` — the timeline distinguishes bot vs human.
- Sending from the panel is **send-then-persist**: the gateway confirms
  delivery, then the row is written. A failed send persists nothing — the
  operator's text stays in the composer and the error is explicit (a
  persisted-but-undelivered message would lie in the thread).
- `GET /admin/contacts` exposes `mode`, handoff metadata and
  `last_inbound_at` (the 24h-window anchor) and can filter/sort by
  attention.
- Leads graduate from JSONB to a module-owned table (`register_models()`,
  the §8 contract proving itself again) — see the Sprint 7 PRP.

## Revisit when

- Meta ships BSUID send endpoints → drop the `wa_id` requirement in the
  WhatsApp gateway (`NO_WA_ID` disappears).
- Template messages land (backlog) → `/send` grows `type: "template"`.
- ~~A channel needs non-text outbound (media) → extend `message` the same way
  the inbound contract does (`media_url`)~~ — **landed same sprint** (Willy's
  e2e feedback, 2026-06-11): `/send` carries `type: image|document|audio`
  with `media_url` as a base64 `data:` URI (the exact mirror of inbound —
  gateways can never fetch core-private URLs) + `filename` for documents.
  The core also uploads outbound media to the bucket (ADR-003 path) so the
  admin timeline renders what the operator sent. WhatsApp maps PyWa
  `send_image`/`send_document`/`send_audio`; the composer records voice as
  `audio/mp4` when the browser supports it (Meta rejects webm).
