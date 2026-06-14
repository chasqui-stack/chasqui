# PRP: Sprint 10 — Inbound debounce + coalescing (deferred dispatch)

> **Version:** 1.0
> **Created:** 2026-06-14
> **Status:** Draft
> **Tracks:** core#6 (Etapa 2). Etapa 1 (per-identity advisory lock) already
> shipped in v0.2.3.
> **Decisions to record:** ADR-008 (ingest becomes fire-and-coalesce; reply
> moves to the deferred send seam; single-loop worker, multi-worker-safe via
> `FOR UPDATE SKIP LOCKED`).

---

## Goal

Collapse a **rapid burst of inbound messages** from the same identity into a
**single agent turn**. Today every `POST /ingest` is its own synchronous turn,
so when a user fragments a thought across *"Hola"* / *"una consulta"* / *"sobre
mi pedido"* the agent answers each fragment in isolation — incoherent, N× cost,
and (before Etapa 1) racy.

End state: inbound messages are persisted and **acked immediately**; a short
window of silence (`INBOUND_DEBOUNCE_SECONDS`, **default 5s**) later, a
background worker runs **one** coalesced turn over the whole batch and
dispatches the reply through the existing canonical send seam
(`CHANNEL_<CH>_SEND_URL`). The core's reply path flips from *synchronous*
(in the `/ingest` response) to *deferred* (via `POST /send`) — which is exactly
the seam the human-handoff outbound already uses (ADR-004), so gateways inherit
it with near-zero change.

## Why

- **Coherence (the real win).** Etapa 1 made bursts *correct and ordered*; it
  did **not** make them *coherent*. The agent still replies per fragment. One
  coalesced turn lets it answer the whole thought at once.
- **Cost.** N fragments → 1 turn instead of N. Direct LLM-spend reduction on
  the most common messaging pattern.
- **It's the documented Etapa 2.** core#6 was filed during live provider
  testing on 2026-06-14 (Telegram webhook-retry burst → `UniqueViolationError`)
  as the follow-up to the advisory lock.
- **No new infra.** Postgres is the queue (ADR-002: "Postgres is identity").
  No Redis, no broker. `FOR UPDATE SKIP LOCKED` makes the claim multi-worker
  safe on the DB we already run.

## What

Two behavior changes, gated by one env var:

1. **Ingest becomes fire-and-coalesce** (when `INBOUND_DEBOUNCE_SECONDS > 0`):
   `/ingest` persists the inbound, (re)arms `conversations.debounce_due_at =
   now() + window`, and returns an **empty** `IngestResponse` (an ack). No turn
   runs inline.
2. **A worker coalesces + dispatches.** A background loop claims conversations
   whose window has elapsed, gathers every still-pending inbound, runs **one**
   turn over the batch, persists the outbound, dispatches each reply via
   `channel_send.send_message()`, and marks the batch processed.

**Back-compat escape hatch:** `INBOUND_DEBOUNCE_SECONDS=0` keeps today's exact
synchronous path (reply in the `/ingest` body, no worker). This preserves the
trivial single-message case and keeps existing synchronous tests meaningful.

### Success Criteria
- [ ] Several inbound within the window → **exactly one** turn whose input is
      the ordered batch; one coherent reply dispatched via the send seam.
- [ ] A message arriving *during* a turn opens the **next** window (no overlap;
      Etapa 1 advisory lock still held by the coalesced turn).
- [ ] Window read from `INBOUND_DEBOUNCE_SECONDS` (default **5**); `0` =
      synchronous legacy path, byte-for-byte unchanged.
- [ ] Reply dispatched via `POST /send` (not the `/ingest` body) in deferred
      mode.
- [ ] Multi-worker safe: two workers never claim the same conversation
      (`FOR UPDATE SKIP LOCKED`).
- [ ] Human mode (ADR-004) under debounce: inbound persisted, **nothing
      scheduled**, **nothing dispatched** (silence, as today).
- [ ] ADR-008 written; ARCHITECTURE §5/§6 updated; `.env.example` + README +
      service AGENTS reflect deferred dispatch.

---

## All Needed Context

### Documentation & References
```yaml
- file: docs/ARCHITECTURE.md
  why: §5 canonical contract (ingest/send), §6 conversation model — both change
- file: docs/design/adr-002-postgres-only.md
  why: "Postgres is identity" — justifies no broker; this ADR-008 extends it
- file: docs/design/adr-004-conversation-mode-outbound-send.md
  why: the /send seam + human mode this feature reuses; ADR-008 builds on it
- file: core/app/services/ingest_service.py
  why: handle_ingest() splits into schedule-path vs synchronous-path here
- file: core/app/services/channel_send.py
  why: send_message() — the deferred dispatch target (already exists)
- file: core/app/services/orchestrator.py
  why: run_turn() — needs a coalesced variant over a list of inbound
- file: core/app/models/conversation.py
  why: add debounce_due_at column
- file: core/app/models/message.py
  why: add processed_at column (pending vs dispatched marker)
- file: core/app/main.py
  why: lifespan — start/stop the worker loop here
- file: core/app/core/config.py
  why: add inbound_debounce_seconds setting
- file: telegram/app/handlers/message_handlers.py
  why: process_update() renders the sync /ingest reply — must tolerate empty ack
```

### Current flow (synchronous, today)
```
gateway → POST /ingest ─┐
                        │ handle_ingest (one txn):
                        │   advisory_xact_lock(channel:external_id)   ← Etapa 1
                        │   upsert_contact → get_or_create_conversation
                        │   if mode==human: persist inbound, return []
                        │   replies = orchestrator.run_turn(inbound)   ← inline
                        │   persist inbound + outbound
                        └→ IngestResponse(messages=replies)            ← gateway renders
```

### Desired flow (deferred, when INBOUND_DEBOUNCE_SECONDS > 0)
```
gateway → POST /ingest ─┐
                        │ handle_ingest (schedule path, one txn):
                        │   advisory_xact_lock(channel:external_id)   ← still needed (upsert race)
                        │   upsert_contact → get_or_create_conversation
                        │   persist inbound (processed_at = NULL → "pending")
                        │   if mode != human: debounce_due_at = now() + window
                        └→ IngestResponse(messages=[])                ← ack only

worker loop (every ~1s, or LISTEN/NOTIFY):
   claim: SELECT ... FROM conversations
          WHERE debounce_due_at <= now()
          ORDER BY debounce_due_at
          FOR UPDATE SKIP LOCKED
   for each claimed conversation (own txn):
     advisory_xact_lock(channel:external_id)        ← Etapa 1, on the turn
     batch = pending inbound (processed_at IS NULL, direction='in') ORDER BY created_at
     replies = orchestrator.run_coalesced_turn(conversation, batch)
     persist outbound
     mark batch.processed_at = now()
     debounce_due_at = NULL
     commit
     for reply in replies: channel_send.send_message(contact, ...)  ← deferred dispatch
```

### Known gotchas & conventions
```python
# 1. Inbound is now persisted at INGEST time, not after the turn. So the
#    coalesced turn's HISTORY query must EXCLUDE the pending batch (they are the
#    "current input"). History = messages where (processed_at IS NOT NULL) OR
#    (direction == 'out'). Pending inbound is never history.

# 2. Dispatch happens AFTER commit, never inside the turn txn. channel_send can
#    fail (gateway down, WINDOW_EXPIRED) — a failure must NOT roll back the turn
#    (the reply is persisted; redelivery is a separate concern). Log on failure,
#    mirroring notify_service's best-effort posture.

# 3. Etapa 1 advisory lock stays in BOTH places: the ingest schedule path (the
#    upsert_contact/get_or_create race under a burst is unchanged) AND the
#    coalesced turn (serialize against a late straggler).

# 4. Human mode short-circuits BEFORE scheduling: persist inbound, do NOT set
#    debounce_due_at, return []. The worker never sees human-owned threads.

# 5. Multimodal batch: a burst can mix text + image + audio. Render the batch as
#    an ORDERED sequence of Human turns (one per inbound, caps-gated per
#    message), not a single concatenated string — preserves per-message media.

# 6. Clock: do NOT watermark on created_at (gateway received_at is not
#    monotonic and can collide). Use messages.processed_at IS NULL as the
#    pending marker — robust, indexable, no clock dependency.

# 7. Deferred mode REQUIRES CHANNEL_<CH>_SEND_URL for every active channel
#    (that's where the reply goes). Generated projects already wire it for the
#    handoff inbox (ADR-004), but add a startup check that WARNS if debounce>0
#    and a configured channel has no send URL.

# 8. Migrations are Alembic (`make migrate`), not create_all. Next rev: 007.
```

---

## Implementation Blueprint

### Data model (migration `007_inbound_coalescing.py`)
```python
# conversations: when the current debounce window elapses (NULL = nothing pending)
op.add_column("conversations",
    sa.Column("debounce_due_at", sa.DateTime(), nullable=True))
# Partial index: the worker only ever scans armed rows
op.create_index("ix_conversations_debounce_due", "conversations", ["debounce_due_at"],
    postgresql_where=sa.text("debounce_due_at IS NOT NULL"))

# messages: NULL = pending (not yet in a turn); set when coalesced/dispatched
op.add_column("messages",
    sa.Column("processed_at", sa.DateTime(), nullable=True))
# Partial index for the per-conversation pending-batch gather
op.create_index("ix_messages_pending_inbound", "messages",
    ["conversation_id", "created_at"],
    postgresql_where=sa.text("processed_at IS NULL AND direction = 'in'"))
```
Model edits mirror these: `Conversation.debounce_due_at: datetime | None`,
`Message.processed_at: datetime | None`. **Backfill:** existing inbound rows are
historical → set `processed_at = created_at` in the migration so they never look
pending to a freshly-started worker.

### Config (`app/core/config.py`)
```python
# Inbound coalescing (ADR-008). Wait this many seconds of SILENCE after the last
# inbound, then run ONE turn over the whole burst (debounce + coalesce). The
# reply is dispatched via the channel send seam (deferred), not the /ingest body.
# Default 5s: long enough to catch fragmented sends, short enough to feel
# responsive. 0 = disabled → legacy synchronous reply in the /ingest response.
inbound_debounce_seconds: int = 5
```

### Tasks (in execution order)
```yaml
Task 1: Schema
  - CREATE: core/alembic/versions/007_inbound_coalescing.py (columns + indexes + backfill)
  - MODIFY: core/app/models/conversation.py (debounce_due_at)
  - MODIFY: core/app/models/message.py (processed_at)

Task 2: Config
  - MODIFY: core/app/core/config.py (inbound_debounce_seconds)
  - MODIFY: core/.env.example (document INBOUND_DEBOUNCE_SECONDS + send-url note)

Task 3: Split the ingest pipeline
  - MODIFY: core/app/services/ingest_service.py
    * keep the advisory lock + upsert + get_or_create (shared)
    * if debounce==0: existing synchronous path UNCHANGED
    * if debounce>0: persist inbound (processed_at=NULL); if mode!=human set
      debounce_due_at = now()+window; return IngestResponse(messages=[])
    * human mode: persist, no schedule, return [] (both paths)

Task 4: Coalesced turn in the orchestrator
  - MODIFY: core/app/services/orchestrator.py
    * add run_coalesced_turn(session, conversation, batch: list[Message])
      → builds [system, *history(excluding pending), *current(batch as ordered
        caps-gated Human turns)]; reuses _build_agent/_get_agent
    * make _history_messages exclude pending inbound (processed_at IS NULL)

Task 5: The worker
  - CREATE: core/app/services/coalesce_worker.py
    * claim_due(session) → conversations WHERE debounce_due_at <= now()
      FOR UPDATE SKIP LOCKED (cap N per tick)
    * process_conversation(conv): own txn → advisory lock → gather pending batch
      → run_coalesced_turn → persist outbound → mark processed_at → clear
      debounce_due_at → commit → dispatch each reply via channel_send (post-commit,
      best-effort, log on ChannelSendError)
    * run_loop(stop_event): poll every ~1s (POLL_SECONDS), back off when idle;
      optional LISTEN/NOTIFY wake is a follow-up, not v1

Task 6: Wire the worker lifecycle
  - MODIFY: core/app/main.py (lifespan)
    * if settings.inbound_debounce_seconds > 0: start asyncio task on startup,
      signal stop + await on shutdown
    * startup check: warn if debounce>0 and any configured channel lacks a send URL

Task 7: Gateways tolerate the empty ack
  - MODIFY: telegram/app/handlers/message_handlers.py
    * process_update: empty messages list is SUCCESS (deferred), not error —
      don't send error_reply on an empty-but-present response
  - MIRROR: whatsapp gateway equivalent
  - NOTE: the reply later arrives via the gateway's existing POST /send — no new
    code there

Task 8: Tests (see Validation Loop)

Task 9: Docs + ADR
  - CREATE: docs/design/adr-008-deferred-dispatch-coalescing.md
  - MODIFY: docs/ARCHITECTURE.md §5/§6 (ingest = ack; reply via send seam; worker)
  - MODIFY: core/AGENTS.md (+ symlinked CLAUDE.md), README env table
```

---

## Validation Loop

### Level 1: Core unit/integration (`cd core && uv run pytest -q`)
```python
# test_coalescing.py
- ingest_schedules_instead_of_replying:
    debounce=5 → POST /ingest returns messages==[] and sets debounce_due_at
- worker_coalesces_burst_into_one_turn:
    3 pending inbound → run_coalesced_turn called ONCE with the ordered batch;
    one outbound persisted; channel_send.send_message called per reply (mock)
- batch_marked_processed_and_window_cleared:
    after process → processed_at set on all 3, debounce_due_at IS NULL
- message_during_turn_opens_next_window:
    inbound arriving after claim but before clear → still pending → next tick
- human_mode_under_debounce_schedules_nothing:
    mode==human → inbound persisted, debounce_due_at NULL, no dispatch
- synchronous_fallback_when_disabled:
    debounce=0 → /ingest returns the reply inline; worker not started (existing
    ingest tests pass UNCHANGED under debounce=0)
- skip_locked_claim_is_exclusive:
    two concurrent claim_due() over one due row → only one gets it
- history_excludes_pending_batch:
    pending inbound never appears in _history_messages output
# Hermeticity: stub orchestrator.run_turn/run_coalesced_turn and channel_send
# (no real LLM key, no real gateway) — same posture as test_ingest_auth.py
```

### Level 2: Live e2e (Telegram, the cheap channel)
```bash
# Generated project, debounce default 5s, CHANNEL_TELEGRAM_SEND_URL wired.
# Fire 3 quick messages: "Hola" / "una consulta" / "sobre mi pedido último".
# Expect: ~5s after the last, ONE coherent reply that addresses all three.
# Logs: one run_turn, one /send round-trip, debounce_due_at cleared.
# Then set INBOUND_DEBOUNCE_SECONDS=0, restart → immediate per-message replies.
```

---

## Final Checklist
- [ ] Migration 007 applies + backfills historical inbound as processed
- [ ] `INBOUND_DEBOUNCE_SECONDS` (default 5) in config + `.env.example`
- [ ] Ingest schedule path (deferred) + synchronous path (=0) both correct
- [ ] `run_coalesced_turn` orders the batch + excludes it from history
- [ ] Worker claims via `FOR UPDATE SKIP LOCKED`; dispatch is post-commit, best-effort
- [ ] Worker started/stopped in lifespan only when debounce>0
- [ ] Startup warns if debounce>0 and an active channel has no send URL
- [ ] Gateways treat empty `/ingest` ack as success
- [ ] Etapa 1 advisory lock retained in ingest AND coalesced turn
- [ ] Tests green in CI (hermetic); live Telegram e2e passes both modes
- [ ] ADR-008 + ARCHITECTURE §5/§6 + READMEs/AGENTS updated

---

## Anti-Patterns to Avoid
- ❌ Don't reply in the `/ingest` body in deferred mode — it must go via `/send`.
- ❌ Don't watermark on `created_at` (gateway clock, non-monotonic) — use
  `processed_at IS NULL`.
- ❌ Don't dispatch inside the turn transaction — commit first, then send.
- ❌ Don't let a `ChannelSendError` roll back the (already-persisted) turn.
- ❌ Don't run the worker without `SKIP LOCKED` — two replicas would double-fire.
- ❌ Don't drop the Etapa 1 lock — the upsert race under a burst is still real.
- ❌ Don't schedule a debounce for human-mode threads.
- ❌ Don't add Redis/a broker — Postgres is the queue (ADR-002).

---

## Notes

**Default window = 5s (decided 2026-06-14).** Asymmetric cost: overshooting only
adds a little latency (perceived ≈ window + turn time), but undershooting fires
mid-typing → splits the burst → two turns → defeats the feature. 5s tolerates a
slow mobile typist while staying responsive. Env-tunable down to 3s for fast
audiences; `0` disables.

**Worker model (v1):** a single in-process asyncio loop in the core, polling
~1s. `FOR UPDATE SKIP LOCKED` already makes it correct under N replicas, so
horizontal scale needs no new code. `LISTEN/NOTIFY` to wake the loop instead of
polling is a clean follow-up (lower latency, less idle DB chatter) — not v1.

**Why this is ADR-worthy:** it flips the core's reply contract from synchronous
to deferred and introduces a background worker. ADR-008 must cover: ingest =
fire-and-coalesce; reply path = send seam; worker model + multi-worker safety;
the `=0` synchronous fallback; and the send-URL prerequisite.

**Out of scope (future):** typing-indicator-aware window extension (extend while
the user is composing — most channels send a "typing" event); retry/redelivery
of a failed deferred dispatch; per-conversation window override.
