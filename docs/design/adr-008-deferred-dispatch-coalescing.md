# ADR-008 — Inbound debounce + coalescing (deferred dispatch)

> **Status:** Accepted — 2026-06-14
> **Sprint:** 10 (inbound coalescing) — core#6 Etapa 2
> **Related:** ARCHITECTURE §5 (canonical contract), §6 (conversation model), ADR-002 (Postgres-only / omakase), ADR-004 (`POST /send` outbound seam, conversation mode)

## Context

Each inbound message is its own `POST /ingest` → its own synchronous agent
turn, and turns for the same conversation could run concurrently. On messaging
channels people **fragment a thought** across several quick sends — *"Hola"* /
*"una consulta"* / *"sobre mi pedido"*. That produces four problems:

1. **Races** on contact/conversation creation under a burst (the
   `UniqueViolationError` we hit on a Telegram webhook-retry storm, 2026-06-14).
2. **Out-of-order replies** — the turn for msg2 finishes before msg1.
3. **Incoherent answers** — the agent replies to each fragment in isolation,
   never seeing its siblings.
4. **Wasted cost** — N turns where one would do.

**Etapa 1** (a per-identity `pg_advisory_xact_lock`, shipped v0.2.3) serialized
turns per identity and fixed **1** and **2**. It did **not** fix **3** or **4**:
the agent still answers each fragment. This ADR is **Etapa 2** — the coherence
and cost win: collapse a rapid burst into **one** turn.

## Decision

### 1. Ingest becomes *fire-and-coalesce* (gated by `INBOUND_DEBOUNCE_SECONDS`)

When `INBOUND_DEBOUNCE_SECONDS > 0` (default **5**), `/ingest` no longer runs
the turn inline. It:

- takes the Etapa 1 advisory lock (the upsert race under a burst is still real),
- upserts the contact, gets-or-creates the conversation,
- persists the inbound as **pending** (`messages.processed_at IS NULL`),
- **(re)arms** `conversations.debounce_due_at = now() + window` — every fresh
  message pushes the deadline out, so the turn fires only after the burst
  settles (a debounce, not a fixed timer),
- returns an **empty** canonical response (`messages: []`) — an ack.

`INBOUND_DEBOUNCE_SECONDS = 0` keeps the **legacy synchronous path** unchanged
(reply in the `/ingest` body, no worker). It is the escape hatch for the
trivial single-message case and for deployments that don't want a worker.

Human mode (ADR-004) still short-circuits: the inbound is persisted **marked
processed** (a human owns the reply) and the window is **not** armed — the
worker never sees human-owned threads.

### 2. A Postgres-backed worker coalesces and dispatches

A single in-process asyncio loop (`app/services/coalesce_worker.py`) polls for
due conversations (`debounce_due_at <= now()`) and, **per conversation in one
transaction**:

- claims the row with `SELECT … FOR UPDATE SKIP LOCKED` — multiple replicas
  never grab the same conversation, with no broker,
- takes the Etapa 1 advisory lock (serializes against a straggler `/ingest`),
- gathers the pending inbound (`processed_at IS NULL, direction='in'`, ordered),
- runs **one** coalesced turn over the whole batch,
- persists the outbound, stamps the batch `processed_at`, clears the window,
- **commits**, then **dispatches** each reply via `channel_send.send_message()`.

**No Redis, no broker** — Postgres is the queue (ADR-002: "Postgres is
identity"). `FOR UPDATE SKIP LOCKED` is the standard, correct primitive for
this on the database we already run.

### 3. The reply path flips from synchronous to deferred — over the *existing* seam

This is the contract change: the agent reply no longer comes back in the
`/ingest` response; it goes **out** through `POST /send` — the **same** canonical
outbound seam the human-handoff already uses (ADR-004). So:

- The core already knows how to send (`channel_send` resolves
  `CHANNEL_<CH>_SEND_URL` per channel); deferred dispatch **reuses** it.
- Gateways already expose `/send`; they only need to treat an **empty** `/ingest`
  ack as success (the reply arrives moments later via `/send`).
- **Prerequisite:** `CHANNEL_<CH>_SEND_URL` must be set for every active channel
  when debounce > 0 (that's where replies go). The core logs a startup warning
  if it isn't. Generated projects already wire it for the handoff inbox.

### 4. Pending state is a per-row mark, not a clock watermark

`messages.processed_at IS NULL` means "pending". We deliberately do **not**
watermark on `created_at`: the gateway's `received_at` is not monotonic and can
collide, so a timestamp cursor would drop or double-count messages. Marking each
row is robust and indexable (partial index `WHERE processed_at IS NULL AND
direction='in'`).

### 5. Default window = 5s

Messaging agents typically debounce **3–10s**. The cost is **asymmetric**:
overshooting only adds a little latency (perceived ≈ window + turn time);
**undershooting fires mid-typing**, splits the burst into two turns, and defeats
the whole feature. **5s** tolerates a slow mobile typist while staying
responsive. Tunable per deployment via `INBOUND_DEBOUNCE_SECONDS`.

## Consequences

**Positive**
- One coherent reply per burst; N turns become 1 (cost down on the most common
  pattern).
- Multi-worker safe by construction (`SKIP LOCKED`), no new infrastructure.
- Reuses ADR-004's outbound seam — gateways barely change; the core gains no
  channel knowledge.
- Backwards compatible: `=0` restores the exact synchronous behavior.

**Negative / trade-offs**
- The reply is no longer synchronous — `/ingest` callers must not expect it in
  the body when debounce > 0. (All current gateways already ack-fast and can
  receive `/send`.)
- The worker holds a conversation row lock + a connection for the turn duration
  (seconds). Fine at Chasqui's scale; a future refactor (claim-then-process)
  would lift it if needed.
- Multimodal in deferred mode re-hydrates media from the bucket (ADR-003): media
  + coalescing **requires storage**. Without storage, media in a deferred burst
  degrades to a text fallback (the recommended production setup configures
  storage anyway).
- A crash between commit and dispatch leaves a reply persisted but unsent until
  a later trigger; a pending batch whose window was cleared waits for the next
  inbound. Acceptable for v1; a sweeper/redelivery is a follow-up.

## Alternatives considered

- **Redis / a message broker for the queue** — rejected: violates ADR-002's
  Postgres-only identity; `SKIP LOCKED` covers the need.
- **Keep replying synchronously, only add the advisory lock** — that's Etapa 1;
  it fixes integrity/ordering but not coherence/cost.
- **`created_at` watermark instead of `processed_at`** — rejected: the gateway
  clock is not monotonic (§4).
- **Fixed timer from the first message** (not a debounce) — rejected: it cuts
  off bursts that span longer than the window; debounce-on-silence matches how
  people actually type.

## Follow-ups (out of scope)

- **Typing-aware window:** extend the window while the channel reports the user
  is composing (`typing…`) — makes the fixed number matter less.
- **`LISTEN/NOTIFY`** to wake the worker instead of polling (lower latency, less
  idle DB chatter).
- **Redelivery** of a failed deferred dispatch; a sweeper for orphaned batches.
- **Per-conversation window override.**
