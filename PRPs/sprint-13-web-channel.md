# PRP: Sprint 13 — Web channel (embeddable chat widget, MVP)

> **Version:** 1.0
> **Created:** 2026-06-17
> **Status:** Draft
> **Tracks:** chasqui#23 (epic — Web channel).
> **Decision:** [ADR-011](../docs/design/adr-011-web-channel.md) — Node monolith
> (Express + Vite + Preact via `preact/compat`), anonymous visitor UUID, live
> outbound over SSE (complete messages; chunked streaming deferred), one new
> generic internal read on the core, browser↔server the only public hop.

---

## Goal

Ship the **third channel**: an embeddable **chat bubble** a company drops on any
page via `<script>`. An **anonymous** visitor chats with the agent; the core
stays channel-agnostic. New service `chasqui-stack/web` (submodule, sibling of
`whatsapp/`/`telegram/`) — a **Node monolith** that is *both* the gateway and the
client it ships.

End state: open a page with the embed → bubble → chatbox → type → the agent
replies **in real time** (SSE). Image + voice note work (voice answered via STT,
ADR-010, on an audio-less LLM). Close the tab mid-turn, reopen → the missed reply
is there (rehydration). The core learns the word "web" in exactly **one config
var** + **one generic** internal history endpoint — nothing else.

## Why

- **Proves channel-agnostic where it's hardest** — real-time + anonymous identity,
  the one shape messaging platforms don't exercise (epic #23, ADR-011 Context).
- **Most demo-able channel** — a *"Habla con Chasqui"* bubble on `chasqui-website`
  is dogfooding + a live landing in one.
- **Inherits the hard parts for free** — media (ADR-003), STT (ADR-010),
  coalescing/deferred dispatch (ADR-008). The real work is two seams: **identity**
  and **live outbound**.

## What

A Node monolith + one tiny core change, gated by one env var per channel:

1. **Core (the only core change):** add `channel_web_send_url` (per-channel
   pattern) **and** a **generic internal history read** —
   `GET /conversations/{channel}/{external_id}/messages` under `INTERNAL_API_KEY`
   — so a gateway can rehydrate a thread without an admin JWT. + ARCHITECTURE §5.
2. **`chasqui-stack/web` — `src/server` (Express):** `POST /chat` (inbound relay →
   `/ingest`), `GET /stream` (SSE, `visitor→stream` map), `POST /send` (core's
   deferred reply → push down the SSE), `GET /history` (proxy the core read),
   serves `widget.js` + the **demo harness**, `GET /health`. Holds
   `INTERNAL_API_KEY` + origin allowlist + rate-limit.
3. **`chasqui-stack/web` — `src/widget` (Vite + Preact/compat):** `<script>` →
   Shadow-DOM bubble → chatbox; visitor UUID in `localStorage`; `EventSource` for
   replies; media (image upload + mic record); rehydrate-on-open.
4. **A generic `demo.html`** the server serves (e.g. `/demo`) with the widget
   embedded — the **visual integration gate** (and a manual QA fixture forever).

**Escape hatch / default:** the channel is **opt-in** — not scaffolded unless the
operator enables it in the wizard. The core change is inert until
`CHANNEL_WEB_SEND_URL` is set (exactly like Telegram).

### Success Criteria
- [ ] Visitor opens the bubble → types → agent reply appears in the chatbox in
      real time (SSE), with the deferred-dispatch default (ADR-008).
- [ ] Anonymous identity: a `localStorage` UUID = `contact.external_id`; same
      visitor → same conversation across reloads; the operator sees it in the
      admin inbox like any channel.
- [ ] Image + voice note → multimodal turn; the voice note is answered via STT on
      an audio-less LLM (ADR-010), zero web-specific audio code.
- [ ] Close the tab during the deferred turn, reopen → the missed reply is present
      (history rehydration via the new core read).
- [ ] A request from a **non-allowlisted origin** is refused; rate-limit caps a
      flood.
- [ ] `INTERNAL_API_KEY` never reaches the browser (only the server holds it).
- [ ] **Demo gate:** `demo.html` served locally renders the bubble and a full
      round-trip works visually (text + media + reconnect).
- [ ] `grep` proves the core never learned "web" beyond `channel_web_send_url` +
      the **generic** (channel-param) history endpoint.
- [ ] ADR-011 (done) reflected; ARCHITECTURE §5 gains the internal read; new repo
      `AGENTS.md`/`README`; CLI wizard opt-in tracked (cross-repo).

---

## All Needed Context

### Documentation & References
```yaml
- file: docs/design/adr-011-web-channel.md
  why: THE decision — stack/archetype, identity, SSE seam, the one core read, trust boundary
- file: docs/ARCHITECTURE.md
  why: §5 inbound/outbound contract to mirror; §5.1 /send (ADR-004); §5.2 deferred dispatch (ADR-008); §10 identity
- file: core/app/controllers/ingest.py
  why: the INTERNAL_API_KEY router (verify_internal_key, line ~25; router deps ~36) — the new read mounts here
- file: core/app/services/ingest_service.py
  why: find-or-create by (channel, external_id) (~64-80) — the history read REUSES the find-by-external_id lookup (read-only, do NOT create); get_or_create_conversation (~176)
- file: core/app/controllers/admin/contacts.py
  why: GET /{contact_id}/messages (~369) — the serialization to mirror (has_media boolean, NEVER embeddings/media payloads); MessageListResponse schema
- file: core/app/services/channel_send.py
  why: send_url_for() = getattr(settings, f"channel_{channel}_send_url") (~31-37) — adding channel_web_send_url is the whole wiring; outbound payload shape (~66-90)
- file: core/app/core/config.py
  why: channel_whatsapp_send_url / channel_telegram_send_url (~107-108) — add channel_web_send_url next to them
- file: core/app/main.py
  why: lifespan warn loop hardcodes ("whatsapp","telegram") (~63) — add "web"
- file: whatsapp/ (and telegram/)  [submodules]
  why: gateway shape to ECHO conceptually (config, core_client→/ingest, /send, /health) — but in Node, not copied verbatim (ADR-011 §1: new archetype)
- file: admin/  [submodule]
  why: the DX + design system to BORROW (Vite, Tailwind, shadcn tokens, DESIGN.md) — NOT the deployable (static SPA charter, ADR-011 Alternatives)
- file: admin/DESIGN.md
  why: Chasqui brand tokens for the widget (amber #EA9B27, terracotta #C94B22, charcoal #1C1917; Rubik)
- url: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
  why: SSE (EventSource client, text/event-stream server) — the outbound transport
- url: https://preactjs.com/guide/v10/switching-to-preact  (preact/compat)
  why: React mental model at ~4KB; alias react→preact/compat in vite config
```

### The contract this channel speaks (from §5)
```jsonc
// browser → server → core   POST /ingest
{ "channel": "web",
  "contact": { "external_id": "<visitor-uuid>", "display_name": null, "metadata": {} },
  "message": { "type": "text|image|audio", "text": "...", "media_url": "data:<mime>;base64,..." },
  "received_at": "<iso8601>" }

// core → server   POST /send   (the deferred reply, ADR-004/008)
{ "contact": { "channel": "web", "external_id": "<visitor-uuid>" },
  "message": { "type": "text", "text": "...", "media_url": null } }
// server looks up visitor→stream and pushes an SSE event; returns {"status":"sent"}
```

### Live outbound — the seam (ADR-011 §3)
```
browser  --POST /chat-->  server  --POST /ingest (key)-->  core
browser  <==SSE /stream== server  <--POST /send (key)----  core   (deferred, seconds later)
                          server holds  visitor_uuid -> Set<sse connection>
```

### Known gotchas & conventions
```text
# 1. INTERNAL_API_KEY is SERVER-ONLY. It is read in Node env, used on the core
#    hops (/ingest, /history) and verified on inbound /send. It must NEVER be in
#    the widget bundle or any browser response. The browser talks ONLY to the
#    gateway's public routes.

# 2. Deferred dispatch is the DEFAULT (INBOUND_DEBOUNCE_SECONDS=5). So POST /chat
#    returns an EMPTY-messages ack; the real reply arrives later on POST /send.
#    The SSE stream MUST already be open (and outlive the debounce+turn) for the
#    reply to land. Open the stream on widget mount, before the first send.

# 3. visitor→stream map is IN-MEMORY → single replica (or sticky sessions) for
#    MVP. Horizontal scale needs shared pub/sub (Redis) or core-side fanout —
#    FOLLOW-UP (ADR-011 Consequences). Log a clear note; don't silently assume HA.

# 4. Origin allowlist on the PUBLIC routes (/chat, /stream, /history, widget.js):
#    check Origin/Referer against WEB_ALLOWED_ORIGINS + CORS. /send is core→server
#    (internal key), NOT origin-checked.

# 5. Media: the widget sends bytes as a base64 data: URI in media_url — the MIRROR
#    of inbound (ADR-003). MediaRecorder yields webm/opus or ogg; pass the real
#    mime; the core's STT (Groq) accepts both. The core size-caps per type — do a
#    light client guard too. Do NOT proxy media through a third store.

# 6. Rehydration returns text + has_media (mirror admin: NEVER ship embeddings or
#    media payloads). Media bytes on reload are a FOLLOW-UP (live path delivers
#    media fine; on reload, show the type badge like the admin fallback).

# 7. The history read is GENERIC: GET /conversations/{channel}/{external_id}/
#    messages — channel is a param, NOT hardcoded "web". It find-by-(channel,
#    external_id) READ-ONLY (no create); 404 if the contact doesn't exist yet.

# 8. Gateway-local literals (ERROR_REPLY/UNSUPPORTED_REPLY) are .env strings,
#    English default — they fire when the CORE is unreachable (English-only rule).

# 9. SSE hygiene: send a heartbeat/comment every ~15s (proxies kill idle streams);
#    set no-buffering headers; clean up the map entry on 'close'. Reconnect with
#    backoff on the client (EventSource auto-retries; cap it).

# 10. Preact: alias react/react-dom → preact/compat in vite.config so shadcn-style
#     components and the React mental model work at Preact size.
```

---

## Implementation Blueprint

### Core change A — config wiring (`core/app/core/config.py`, `main.py`)
```python
# config.py — next to the other channels (~108)
channel_web_send_url: str | None = None   # CHANNEL_WEB_SEND_URL — the web gateway /send
# main.py lifespan warn (~63): include web
for ch in ("whatsapp", "telegram", "web"): ...
# .env.example: document CHANNEL_WEB_SEND_URL (commented, opt-in)
```

### Core change B — generic internal history read (`core/app/controllers/`)
```python
# Mount on the INTERNAL_API_KEY router (the /ingest router, deps=[verify_internal_key]).
# A new controller (e.g. conversations.py) keeps it tidy; same router/dep.
@router.get("/conversations/{channel}/{external_id}/messages",
            response_model=MessageListResponse)
async def read_history(channel: str, external_id: str, limit: int = 50,
                       session: AsyncSession = Depends(get_session)):
    # READ-ONLY find by (channel, external_id) — reuse ingest_service's lookup,
    # but do NOT create. 404 if absent.
    contact = await find_contact(session, channel, external_id)
    if contact is None:
        raise HTTPException(404, {"code": "NO_CONTACT"})
    # Reuse the SAME message serialization as admin GET /contacts/{id}/messages:
    #   role, type, text, has_media (bool), created_at, conversation_id
    #   NEVER serialize embeddings or media payloads.
    return await list_recent_messages(session, contact.id, limit)
# ARCHITECTURE §5: document this as the internal read-mirror of /ingest.
# Tests: requires the key; scopes by (channel, external_id); has_media only.
```

### Server (`chasqui-stack/web/src/server`, Express + TS)
```text
config.ts        CORE_URL, INTERNAL_API_KEY, WEB_ALLOWED_ORIGINS, PORT,
                 RATE_LIMIT_*, ERROR_REPLY, UNSUPPORTED_REPLY, WEB_PUBLIC_URL
core-client.ts   ingest(payload)  -> POST {CORE_URL}/ingest   (X-Internal-API-Key)
                 history(channel, externalId) -> GET .../conversations/...  (key)
streams.ts       Map<visitorId, Set<res>>; add/remove; push(visitorId, event); heartbeat
routes:
  POST /chat     body {visitor,type,text,media?} -> build canonical inbound -> ingest()
                 -> 200 ack (empty messages = deferred). Origin-checked + rate-limited.
  GET  /stream   ?visitor=<uuid> -> text/event-stream; register res in map; heartbeat;
                 cleanup on close. Origin-checked.
  POST /send     core->server (verify key): {contact:{channel:"web",external_id},message}
                 -> streams.push(external_id, message) -> {status:"sent"} (or 200 even if
                 no open stream — it's persisted; rehydration will show it).
  GET  /history  ?visitor=<uuid> -> history() proxy (browser never holds the key). Origin-checked.
  GET  /widget.js (+assets)  -> the Vite build
  GET  /demo     -> the demo harness HTML (below)
  GET  /health
middleware:      originAllowlist (Origin/Referer vs WEB_ALLOWED_ORIGINS) + CORS;
                 rateLimit per visitor/IP; verifyInternalKey on /send.
```

### Widget (`chasqui-stack/web/src/widget`, Vite + Preact/compat + TS)
```text
- mounts a Shadow DOM root; injects scoped styles (Tailwind build → shadow);
  reads data-* (data-gateway) off its <script>.
- visitor.ts: getOrCreate UUID in localStorage (key "chasqui_web_visitor").
- on open: GET /history -> render past turns; new EventSource(`${gw}/stream?visitor=`).
- send: optimistic-render the user bubble; POST /chat; reply arrives via SSE 'message'.
- media: <input type=file> (image) + MediaRecorder (mic→audio) -> base64 data: URI.
- reconnect: EventSource auto-retries; cap/backoff; show a subtle "reconnecting".
- UI: floating bubble -> chatbox (visitor right / agent left, markdown, media inline),
  Chasqui tokens from admin/DESIGN.md (amber accent, charcoal chrome, Rubik).
```

### The demo harness (`chasqui-stack/web/public/demo.html`, served at /demo)
```html
<!doctype html><html><head><meta charset="utf-8"><title>Chasqui web widget — demo</title></head>
<body style="font-family:system-ui;max-width:680px;margin:3rem auto">
  <h1>Demo landing</h1>
  <p>Filler content so the bubble floats over a realistic page. Open the bubble
     (bottom-right), send a message, an image, and a voice note.</p>
  <!-- the same embed a real customer would paste -->
  <script src="/widget.js" data-gateway="http://localhost:PORT"></script>
</body></html>
```
> Dev-only fixture (not shipped to customers). It is the **visual integration
> gate** (Validation Level 4) and a permanent manual-QA page. `localhost` is in
> `WEB_ALLOWED_ORIGINS` for dev.

### Tasks (in execution order)
```yaml
Task 0 (GATE — do first): SSE transport spike
  - Minimal Express: GET /stream (text/event-stream) + a static page with
    EventSource that prints pushed events; POST /push to fan out. Confirm the
    browser receives server-pushed events locally. De-risks the whole seam before
    building the widget. (Throwaway or the seed of streams.ts.)

Task 1: Core change A — channel_web_send_url
  - MODIFY: core/app/core/config.py (+ field), main.py (warn loop +"web"),
    core/.env.example (CHANNEL_WEB_SEND_URL, commented)

Task 2: Core change B — generic internal history read
  - ADD: read-only find_contact(channel, external_id) (reuse ingest_service lookup)
  - ADD: GET /conversations/{channel}/{external_id}/messages on the internal router
  - REUSE: the admin message serialization (has_media; no embeddings/media)
  - TEST: key required; scoped; 404 when absent; has_media only
  - MODIFY: docs/ARCHITECTURE.md §5 (the internal read-mirror)

Task 3: New repo + submodule
  - CREATE: chasqui-stack/web (Node, TS, Express, Vite, Preact + preact/compat,
    Tailwind, vitest, Dockerfile, config/deploy.yml, AGENTS.md + CLAUDE.md symlink)
  - ADD as submodule web/ in the parent; commit the pointer

Task 4: Server — config + core client
  - src/server/config.ts, core-client.ts (ingest + history, INTERNAL_API_KEY)

Task 5: Server — inbound relay
  - POST /chat: canonical builder (text/image/audio; media→data: URI) -> ingest()

Task 6: Server — live outbound
  - streams.ts (visitor→stream map, heartbeat, cleanup)
  - GET /stream (SSE) + POST /send (verify key -> push) + GET /history (proxy)

Task 7: Server — security
  - origin allowlist + CORS + rate-limit on public routes; verifyInternalKey /send;
    ERROR_REPLY/UNSUPPORTED_REPLY when the core is unreachable

Task 8: Widget — core UX
  - Shadow DOM mount, visitor UUID, bubble+chatbox, send (optimistic), SSE replies,
    rehydrate-on-open via /history

Task 9: Widget — media + resilience
  - image upload + mic record (MediaRecorder) -> data: URI; reconnect/backoff

Task 10: Demo harness
  - public/demo.html served at /demo (the visual gate); localhost allowlisted in dev

Task 11 (cross-repo): CLI wizard opt-in  -> file cli#N
  - "Add the web chat widget?" -> scaffold web/, write web/.env, set
    CHANNEL_WEB_SEND_URL in core/.env; provision runs `npm` (like admin).
  - stack.py: add "web" to CHANNEL_SERVICES; envfiles: web/.env block.

Task 12: Dogfood + docs
  - "Habla con Chasqui" bubble on chasqui-website
  - docs/WEB-SETUP.md (embed snippet, allowlist, deploy); parent README/
    ARCHITECTURE channel list + landing; new repo README/AGENTS
```

---

## Validation Loop

### Level 0: SSE transport spike (manual, once) — the gate
```bash
# Task 0: run the minimal Express SSE echo, open the static page, POST /push,
# see the event appear in the browser. Confirms browser↔server push works.
```

### Level 1: Core (`cd core && uv run pytest -q`)
```python
# test_internal_history.py
- requires_internal_key:        no/!wrong X-Internal-API-Key -> 401
- scoped_by_channel_external:   returns only that contact's recent messages
- not_found_when_absent:        unknown (channel, external_id) -> 404 NO_CONTACT
- never_serializes_media:       payload has has_media (bool), no embeddings/bytes
- read_only:                    calling it does NOT create a contact/conversation
- send_url_resolves_web:        send_url_for("web") reads channel_web_send_url
```

### Level 2: Server (`cd web && npm test` — vitest, core mocked)
```text
- chat_relays_canonical:    POST /chat -> ingest() called with channel:"web",
                            external_id, media as data: URI; returns the ack
- send_pushes_to_stream:    register a fake SSE conn -> POST /send -> conn receives it
- send_without_stream_ok:   POST /send with no open stream -> 200 (persisted; no throw)
- origin_allowlist_refuses: POST /chat from a non-allowlisted Origin -> 403
- rate_limit_caps_flood:    N+1 rapid /chat from one visitor -> 429
- internal_key_required_on_send: /send without the key -> 401
- key_never_leaked:         no public response/asset contains INTERNAL_API_KEY
```

### Level 3: Widget (`cd web && npm test` — vitest + @testing-library/preact)
```text
- visitor_uuid_persists:    mount twice -> same localStorage UUID
- renders_history_on_open:  mocked /history -> past turns rendered
- send_optimistic + sse_reply: POST /chat mocked; dispatch an SSE 'message' ->
                            agent bubble appears
- media_builds_data_uri:    selecting an image yields a data: URI in the payload
```

### Level 4: VISUAL integration gate (the demo harness) — manual, REQUIRED
```bash
# Run core (LLM=anthropic → caps.audio=False, STT_PROVIDER=groq) + web locally,
# CHANNEL_WEB_SEND_URL=http://localhost:PORT/send, localhost allowlisted.
# Open http://localhost:PORT/demo  and verify VISUALLY:
#   1. bubble floats; opening shows the chatbox
#   2. type "hola" -> agent reply streams into the chat in real time
#   3. send an image -> multimodal turn answers it
#   4. record a voice note -> answered via STT (audio-less LLM), not "please type it"
#   5. close the tab DURING the deferred turn, reopen -> the missed reply is there
#   6. point a second demo page at a NON-allowlisted origin -> /chat refused
# (Optionally automate the happy path with Playwright per the webapp-testing skill;
#  the manual visual check is the acceptance gate.)
```

### Level 5: channel-agnostic proof
```bash
cd core && grep -rin '"web"\|web_send\|webwidget\|websocket' app | grep -v test
# Expect ONLY: channel_web_send_url (config) + the generic channel-param history
# route. No "if channel == 'web'", no web SDK, no business logic. The core never
# learned the widget exists.
```

---

## Final Checklist
- [ ] Level 0 SSE spike passed before building the widget
- [ ] Core: `channel_web_send_url` + generic internal history read (key-gated,
      scoped, has_media only, read-only) + §5 update; tests green
- [ ] Server: /chat relay · /stream SSE + visitor map · /send push · /history proxy
- [ ] Server: origin allowlist + rate-limit + key-only /send; key never in browser
- [ ] Widget: Shadow DOM, visitor UUID, bubble+chatbox, SSE replies, media,
      rehydrate-on-open, reconnect/backoff
- [ ] **Demo gate (Level 4) passed visually** — text + image + voice + reconnect +
      origin-refusal
- [ ] Level 5 grep clean (no web-specific core logic)
- [ ] New repo `AGENTS.md`/README; submodule pinned; CLI wizard opt-in (cli#N);
      WEB-SETUP.md; parent README/ARCHITECTURE/landing; chasqui-website bubble
- [ ] Multi-replica caveat + chunked-streaming + media-on-rehydration documented as
      follow-ups (not silently assumed)

---

## Anti-Patterns to Avoid
- ❌ Don't let the browser hold or ever receive `INTERNAL_API_KEY` — server-only.
- ❌ Don't put `if channel == "web"` or any business logic in the core — one config
  var + one GENERIC (channel-param) read endpoint, nothing else.
- ❌ Don't hardcode "web" in the history route — it's channel-scoped and generic.
- ❌ Don't open the SSE stream lazily after the first send — the deferred reply
  (ADR-008) would have nowhere to land. Open it on mount.
- ❌ Don't ship token/chunk streaming — MVP is complete messages (ADR-011 §3);
  chunks are a gateway-local follow-up.
- ❌ Don't assume horizontal scale — the visitor→stream map is in-memory; single
  replica / sticky for MVP, Redis fanout is a follow-up. Say so, don't hide it.
- ❌ Don't use full React for the embed (~45KB) — Preact/compat (~4KB). React stays
  in the admin only.
- ❌ Don't serialize embeddings or media payloads in the history read (mirror admin).
- ❌ Don't fold this into the admin — wrong shape on every axis (ADR-011 Alternatives).

---

## Notes

**Scope = MVP of an epic.** This PRP delivers a working anonymous web chat with
real-time complete-message replies, media, and rehydration. Deliberately **out of
scope** (ADR-011 Follow-ups): chunked/token streaming (capability negotiation),
typing/presence, richer outbound payloads (own ADR), authenticated/known visitors,
conversation reset, proactive outbound, media-bytes on rehydration, multi-replica
SSE fanout. Each is a clean follow-up; none blocks the MVP.

**The demo harness earns its keep.** `demo.html` is the cheapest honest test of an
embeddable widget — the bubble over a real page, a full round-trip you can *see*.
It is the acceptance gate (Level 4) and a permanent QA fixture; it is dev-only and
never shipped to customers.

**The new archetype is a conscious cost (ADR-011).** First non-Python gateway; no
verbatim reuse from whatsapp/telegram; the `chasqui-create-channel` skill/docs gain
a note that a channel may be a JS monolith when it ships its own client. Recorded,
not smuggled.
