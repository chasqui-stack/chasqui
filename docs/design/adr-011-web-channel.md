# ADR-011 — Web channel (embeddable chat widget): anonymous identity + live outbound over SSE

> **Status:** Accepted — 2026-06-17
> **Sprint:** 13+ (Web channel epic — MVP)
> **Related:** ARCHITECTURE §5 (canonical contract — `channel`, `external_id`, media as `data:` URI), §5.1 (`POST /send`, ADR-004), §5.2 (deferred dispatch / coalescing, ADR-008), §10 (identity — BSUID-first, "channel-scoped id otherwise"), §12 (roadmap — web channel), ADR-003 (media storage / re-hydration), ADR-010 (STT fallback for inbound audio), `chasqui-create-channel` skill (the gateway pattern this channel **departs from** — Decision 1), admin `AGENTS.md` (the static-SPA charter — why the web channel is a separate service, Decision 1)

## Context

The stack has two channel gateways — WhatsApp and Telegram — and the canonical
contract (§5) was designed so *"a future web channel is just another adapter."*
This ADR opens that channel: an **embeddable chat widget** that a company drops
on whatever page it wants — a floating bubble that opens a chatbox between an
anonymous site visitor and the agent.

A web widget is the **hardest** channel to fit the existing seam, and that's the
point — it stress-tests the channel-agnostic thesis where it's most exposed.
Two things make it different from a messaging platform:

1. **No platform-provided identity.** WhatsApp/Telegram hand the gateway a stable
   user id (BSUID, `chat_id`). The browser hands us nothing — there is no
   account, no auth, no phone number.
2. **The reply has no inbound webhook to ride home on.** WhatsApp/Telegram are
   *push* platforms: the gateway calls their API to deliver a reply. A browser
   is not addressable from the server — the gateway must hold a **live
   connection** and push down it. And because the core defaults to **deferred
   dispatch** (§5.2: debounce > 0 → the reply arrives via `POST /send` seconds
   later, not in the `/ingest` response), that live connection has to **outlive
   the turn**.

Everything else the contract already covers: inbound is an ordinary `POST
/ingest`, media inlines as a `data:` URI (ADR-003), coalescing (ADR-008) works
unchanged, and audio gets the STT fallback (ADR-010) for free. So the design
work is concentrated in exactly two seams — **identity** and **live outbound** —
plus the trust boundary a public, anonymous endpoint introduces.

The non-goal that bounds this MVP: **token-by-token streaming**. It's tempting
(a web chat *looks* like it wants a typewriter effect), but it's a presentation
concern that only a subset of channels can use, and baking it into the core
contract would tax every push channel for one channel's benefit. See Decision 3
and Alternatives.

## Decision

### 1. A new service `chasqui-stack/web` — a **Node monolith** (Express + Vite + Preact), not a FastAPI gateway and not folded into the admin

The web channel is **another channel adapter** — it speaks only the canonical
contract, resolves its reply URL from `CHANNEL_WEB_SEND_URL` like any channel
(§5.1), and adds **no** web-specific business logic to the core (the one
exception is the generic internal read of Decision 3). But it **deliberately
departs from the FastAPI-gateway archetype** of WhatsApp/Telegram, for one
reason: it is the **only channel that ships its own client**. WhatsApp/Telegram
get their client from a third party; here the client is a **browser widget we
build** — an unavoidable JS bundle with a build step. Pairing a Python gateway
with a separate JS build means *two toolchains for one channel*; a single Node
monolith is the simpler shape. The canonical contract is **language-agnostic by
design** (the core speaks HTTP + JSON and never specified the gateway's
language), so this is fully compatible. One repo, one `npm` toolchain:

- **`src/server`** — an **Express** server: relays inbound → `POST /ingest`,
  holds the live **SSE** stream, receives the core's `POST /send` and pushes it
  to the browser, and **serves the built widget**. Holds the `INTERNAL_API_KEY`,
  the origin allowlist and the rate-limit (Decision 5).
- **`src/widget`** — the embeddable UI, **Vite + Preact via `preact/compat`**: a
  `<script>` → floating bubble → chatbox (Decision 6), **Shadow-DOM isolated**.
  `preact/compat` keeps a **React mental model at ~4KB** (vs React's ~45KB) — and
  lets the widget **borrow the admin's design system** (Tailwind + shadcn tokens,
  `DESIGN.md`) without React's embed weight.

Two artifacts (server + widget), one repo — the natural shape for the one channel
that is both a **bridge** *and* a **client**.

**Why a separate service and not part of the admin.** Tempting (one less deploy),
but the admin is the **wrong shape on every axis**, and its own charter forbids
exactly what a gateway needs. It is a **static Vite SPA** — *"no BFF, no SSR, no
server functions"* (admin `AGENTS.md`), served by `serve -s dist`; it has **no
running server** to hold an SSE stream, receive the core's `/send`, or keep the
`INTERNAL_API_KEY` server-side. It is **operator-only and authenticated** —
*"end users never access the admin"* — whereas the web channel is **public,
anonymous, internet-facing**; co-locating them fuses the most-privileged and
most-public surfaces into one **blast radius**. And the admin deliberately
**avoids realtime** (*"polling ~5s, no websockets, omakase"*); the web channel is
realtime by definition. They differ on runtime, trust zone, traffic profile and
release cadence. We **reuse the admin's DX and design system, not its
deployable**: the web service is an *admin-sibling* in toolchain, an **independent
service** in deployment (its own Kamal config, opt-in).

### 2. Anonymous visitor identity — a UUID in `localStorage`

There is no BSUID and **no auth**. On first load the widget mints a UUID, stores
it in `localStorage`, and sends it as `contact.external_id` on every inbound —
the contract's *"channel-scoped id otherwise"* (§5, §10). One visitor → one
`contact` → one **persistent conversation**, exactly like WhatsApp/Telegram
(`external_id` → contact → conversation; the operator sees it in the admin inbox
identically).

Losing the UUID (new machine, cleared cache, incognito) means a **new visitor
and a fresh conversation** — the same trade-off as a WhatsApp user switching
phones. Explicitly **accepted**: a stronger identity is not worth auth friction
on an anonymous web chat.

Capturing a **name or email is *not* the channel's job.** A deployer who wants
it asks for it through the **system prompt** (or a lead-capture tool module) and
stores it as conversation data — the same lever every other channel uses. The
widget stays identity-light by design.

### 3. Live outbound — the gateway holds an **SSE** connection and relays `/send`; **complete messages**, streaming deferred

When the widget opens, it holds an **SSE** stream to the gateway
(`GET /stream?visitor=<uuid>`). The gateway keeps a map `visitor_uuid → open
stream(s)`. When the core dispatches a reply (`POST /send` →
`CHANNEL_WEB_SEND_URL`, deferred per ADR-008), the gateway looks up the visitor's
stream and **pushes the message down it**. The core stays channel-agnostic — it
POSTs a one-shot reply exactly as for WhatsApp; the gateway owns the browser
transport. **ADR-004 and ADR-008 are untouched.**

Pairing: **inbound = plain `POST /ingest`** (browser → gateway), **outbound =
server-push over SSE** (gateway → browser). SSE — not WebSocket — because the
traffic is one-way server-push over an already-solved HTTP path; WS bidirectional
is overkill for the MVP (revisit it for typing/presence, Follow-ups).

**Delivery is one event = one complete message.** Chunked / token-by-token
streaming is **deliberately deferred** — the principle is **homologation across
channels** (the core emits one-shot replies for *all* channels; push platforms
can't consume a token stream anyway). Crucially, that principle is also what
keeps the door open: the browser↔gateway transport is **already event-based**, so
adding `chunk` events later is a **gateway-local** change that never touches the
core contract. A deployer who wants a typewriter effect extends the *gateway*,
not the core. (This is documented for them in the web gateway's `AGENTS.md`.)

**Replies are not lost when the tab is closed.** A deferred reply (ADR-008) can
land while the visitor has the tab shut. The core **already persists every
message**, so the widget **rehydrates conversation history on open** — the live
SSE is the fast path, not the durable one. But the core exposes **no internal
read** for this today: every message read is admin/JWT (`GET
/admin/contacts/{id}/messages`), unusable by a gateway. So this MVP adds **one
new internal endpoint** — a `INTERNAL_API_KEY`-protected, channel + `external_id`
scoped read of recent messages, the **read-mirror of `/ingest`**. It is
deliberately **generic** (any channel that wants rehydration reuses it), and it
is the **single point where the web channel touches the core**. Being a new seam
in the contract, it lands with a matching `docs/ARCHITECTURE.md` §5 update in the
same PR.

### 4. Full media parity over the canonical contract (text, image, audio, …)

The widget supports the same media as the other gateways. **Inbound**, the
browser uploads bytes and the gateway inlines them as a `data:` URI (ADR-003),
identical to WhatsApp/Telegram. **Outbound** media parts render in the chatbox.
The gateway **normalizes browser-captured formats** (a `MediaRecorder` voice note
is WebM/Opus or OGG/Opus) — and inbound audio gets the **STT fallback (ADR-010)
for free**: an audio-less LLM answers a web voice note exactly as it answers a
WhatsApp one, no web-specific code. The agent never learns the channel changed.

### 5. Trust boundary — the browser↔gateway hop is the only public surface

The browser talks **only** to the web gateway (widget + `/ingest` + `/stream`).
The `INTERNAL_API_KEY` stays **gateway-side**; the gateway↔core hop is unchanged
(§5.1). Because the browser-facing ingest is **public and unauthenticated**
(anonymous visitors), the gateway adds its own light protections at that hop:

- **Origin allowlist** — the operator configures which domains may embed the
  widget; requests from other origins are refused (CORS + server-side check).
- **Per-visitor / per-IP rate limiting** — bounds abuse of a public endpoint
  without introducing user auth.

User-facing literals that fire when the core is unreachable (`ERROR_REPLY`,
`UNSUPPORTED_REPLY`) are **gateway-local** `.env` strings, English by default —
per the stack's English-only convention.

### 6. The widget — a `<script>` snippet → floating bubble → chatbox

Distribution is a one-line embed the company places **wherever it chooses** (its
decision, not ours):

```html
<script src="https://chat.<domain>/widget.js" data-gateway="https://chat.<domain>"></script>
```

It renders a floating **bubble**; clicking opens a **chatbox** (visitor right,
agent left, media inline). The bundle is **self-contained and small** — Preact
via `preact/compat`, **Shadow-DOM isolated** so no styles leak either way (§1) —
built by Vite and **served by the Express server itself**, configured via
`data-*` attributes. First dogfood target: a *"Habla con Chasqui"* bubble on
`chasqui-website`.

## Consequences

**Positive**
- Proves the channel-agnostic thesis in the hardest case: a channel with neither
  platform identity nor a push API needs **one small generic** core addition (the
  internal history read) and nothing web-specific.
- The most **demo-able** channel — a live bubble on the landing is dogfooding +
  marketing in one.
- Media + STT + coalescing are inherited, not rebuilt.
- The streaming door is left open at **zero present cost**, and the escape hatch
  is honest (gateway-local, contract-stable).

**Negative / trade-offs**
- A **new long-lived connection type** (SSE) the stack didn't have — connection
  lifecycle, reconnect/backoff, and the `visitor → stream` map are real gateway
  state to get right (the gateway is still *stateless* re: business logic, but it
  holds transport state).
- A **public unauthenticated endpoint** — origin allowlist + rate limit are
  mandatory, not optional; abuse surface is wider than a platform webhook.
- Anonymous identity is **weak by construction** — cleared storage = lost thread.
- The core stops being strictly read-closed to gateways: it gains its **first
  internal read endpoint** (history rehydration). Small and generic, but it's a
  new seam in the contract to maintain.
- **A new gateway archetype** — the first **non-Python** gateway. It forgoes the
  ~40% verbatim reuse WhatsApp/Telegram share (`core_client.py`, media handling,
  `/send`), and the `chasqui-create-channel` skill + docs must now say *"gateways
  are usually FastAPI; the web channel is a Node monolith because it ships its own
  client."* A deliberate fork in the doctrine, not an accident.
- The Node service holds **SSE transport state** (the `visitor → stream` map) —
  still no *business* state, but it is no longer a stateless request/response box
  like the other gateways; connection lifecycle and reconnect are real concerns.
- No streaming in MVP — a complete-message chat feels slightly less "live" than a
  token stream. Accepted; recoverable later without a contract change.

## Alternatives considered

- **Core-side SSE / token streaming (the core streams to gateways)** — rejected.
  It leaks the orchestrator's turn model (LangGraph loops, tool calls,
  coalescing) into the contract and forces *every* gateway to speak a streaming
  protocol only one can use; push platforms gain nothing. The Telegram
  `editMessageText` "fake streaming" trick actually **reinforces** this: progressive
  output is presented **per-gateway** (Telegram edits, web SSE, WhatsApp
  buffers), so the core should stay one-shot and let gateways present. Revisit as
  an **opt-in capability negotiation**, never a default tax (Follow-ups).
- **WebSocket instead of SSE** — rejected for MVP. Inbound is fine as plain POST;
  outbound is one-way push. WS doubles the moving parts for bidirectional we
  don't need yet. Reconsider when typing/presence make a duplex channel pay.
- **Long-polling the core for replies** — rejected: higher latency and request
  overhead than SSE, and it would push channel-awareness toward the core.
- **Identity capture (name/email) in the channel** — rejected: it's a
  prompt/tool-module concern (Decision 2). Hard-coding it would make the widget
  opinionated about a flow that belongs to the operator.
- **Authenticated / known visitors as the MVP identity** — rejected as the
  baseline: most embeds are anonymous support chats. Passing a known identity is
  a clean **follow-up** (a signed `data-user` the host page provides).
- **Fold the web channel into the admin (avoid a new service entirely)** —
  rejected, despite the "one less deploy" appeal. The admin's own charter forbids
  what a gateway needs: it is a **static Vite SPA** (*"no BFF, no SSR, no server
  functions"*, served by `serve -s dist`) with **no running server** to hold SSE,
  receive the core's `/send`, or keep the `INTERNAL_API_KEY` server-side; it is
  **operator-only** (*"end users never access the admin"*) while the web channel
  is **public/anonymous** — fusing them merges the most-privileged and most-public
  surfaces into one blast radius; and it **avoids realtime by design** (*"polling,
  no websockets, omakase"*). Opposite shape on runtime, trust, traffic and
  cadence. We reuse the admin's **DX and design system**, not its deployable.
- **FastAPI gateway (like WhatsApp/Telegram) + a separate JS widget build** —
  rejected: this is the one channel that *must* ship a JS client with a build
  step, so a Python gateway means **two toolchains in one channel**. The canonical
  contract is language-agnostic, so a Node monolith is valid and simpler here.
  (This is the archetype decision — Decision 1.)
- **React (full) for the widget** — rejected for embed weight: ~45KB vs Preact's
  ~4KB, shipped into *every* host page. `preact/compat` keeps the React mental
  model and admin-pattern reuse at a fraction of the size. (The admin keeps full
  React — it's an internal panel where bundle size is irrelevant.)
- **Gateway-side transcript storage (avoid the new core read)** — rejected: it
  duplicates the core's source of truth and makes the "stateless gateway" hold
  business state. A generic internal read on the core is the right owner.
- **No rehydration in MVP (accept replies lost to a closed tab)** — considered, to
  keep the core untouched. Rejected: the deferred-dispatch default (ADR-008) makes
  the closed-tab window real, and a chat that silently drops answers is a poor
  first impression. The read endpoint is cheap enough to include now.
- **`iframe` embed instead of `<script>`** — viable and more isolated, but
  clumsier to theme/position and to pass host-page config. Kept as an option the
  bundle can offer; the `<script>` bubble is the default.

## Follow-ups (out of scope)

- **Chunked / token streaming** as an **opt-in capability** the gateway negotiates
  — `chunk` events on the existing SSE; core stays one-shot. (Documented as the
  deployer escape hatch in the gateway `AGENTS.md`.)
- **Typing / presence indicators** — cheap over SSE (emit a `typing` event when
  `/ingest` is accepted); deferred to keep the MVP lean, and a likely reason to
  later add WebSocket.
- **Richer outbound payloads** beyond what the contract already carries
  (quick-replies, buttons, Flows, sources/citations) — its **own ADR** when a
  channel needs it; explicitly **not** designed here.
- **Authenticated/known visitors** — host page supplies a signed identity.
- **Conversation reset / multi-thread UI** in the widget.
- **Proactive (agent-initiated) outbound** — push to a returning visitor.
