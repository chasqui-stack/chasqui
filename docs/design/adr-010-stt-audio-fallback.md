# ADR-010 — STT fallback for inbound audio (transcribe when the LLM can't hear)

> **Status:** Accepted — 2026-06-15
> **Sprint:** 12 (STT fallback) — chasqui#15
> **Related:** ARCHITECTURE §5 (canonical contract — `audio` media), ADR-003 (media storage / re-hydration), ADR-008 (coalesced turn re-hydrates media), `app/core/llm_capabilities.py` (`caps.audio`), `app/services/orchestrator.py` (`_current_message` degradation)

## Context

A contact sends a **voice note**. The gateway normalizes it into the canonical
contract as an `audio` message with the bytes inlined as a `data:` URI (ADR-003).
What happens next depends on the configured LLM:

- **Native-audio models** (Gemini — `caps.audio = True`) get the audio as a
  content block and answer its content directly. This already works.
- **Every other model** (Anthropic Claude, plain OpenAI GPT, most of the
  registry — `caps.audio = False`) hits the **graceful degradation** in
  `orchestrator._current_message`: *"[The user sent a voice message you cannot
  listen to. Ask them to write it as text.]"* The agent politely asks the user
  to retype. Correct, but a dead end — the user already spoke.

The gap: for an audio-less LLM there is **no way to act on a voice note** even
though transcription is a cheap, well-understood step. chasqui#15 asks for an
**opt-in STT fallback** that transcribes the audio *before* the turn so any LLM
can answer it as if it were text.

The hard constraint is the **audio format**. WhatsApp and Telegram voice notes
are **OGG/Opus** (confirmed: `storage.py` maps `audio/ogg`). That rules some
providers in and out:

| Provider | OGG/Opus | Cost/min | Auth | Notes |
|----------|:---:|---|---|---|
| OpenAI `gpt-4o-mini-transcribe` | ❌ not in supported list (`mp3,mp4,mpeg,mpga,m4a,wav,webm`) | $0.003 | API key | would need **ffmpeg** transcoding, a system dep |
| **Groq `whisper-large-v3-turbo`** | ✅ lists `ogg` | **$0.00067** ($0.04/hr) | API key | **OpenAI-compatible** endpoint |
| Google Chirp 3 (Cloud STT v2) | ✅ OGG_OPUS preferred | ~$0.016 | **GCP service account** | heavier auth, not a `.env` key |
| Gemini API (`GOOGLE_API_KEY`) | ⚠️ OGG **Vorbis** only, Opus unconfirmed | cheap | reuse key | risky for voice notes (Opus) |
| OpenRouter (transcription) | — | — | — | catalogue **empty** — not viable |

## Decision

### 1. STT is an opt-in fallback, gated on `caps.audio = False`

Native audio is always preferred. The STT step runs **only** when the LLM lacks
native audio input (`caps.audio = False`) **and** STT is configured. A
native-audio model (Gemini) never transcribes — its path is byte-for-byte
unchanged. STT unconfigured or failing → the existing graceful "ask for text"
fallback, unchanged. **STT never breaks a turn.**

### 2. One OpenAI-compatible transcription client; **Groq is the default**

The `POST /v1/audio/transcriptions` multipart shape is a **de-facto standard** —
OpenAI **and** Groq implement it identically. So the core ships **one** client
(`app/services/transcription.py`, `httpx` multipart — already a dependency, no
SDK) with a configurable `base_url`, and **any** compatible host works by setting
three env vars. No provider-specific code.

The default is **Groq `whisper-large-v3-turbo`** because it:

- accepts **OGG/Opus natively** → kills the #1 risk (no ffmpeg, no transcoding,
  no system dependency for an opt-in feature),
- is the **cheapest** option (~$0.00067/min, ~4.5× cheaper than OpenAI mini),
- is fast, and OpenAI-compatible (drop-in).

OpenAI is a documented alternative (same client, `STT_BASE_URL=https://api.openai.com/v1`,
`STT_MODEL=gpt-4o-mini-transcribe`) — with the caveat that OGG is not in OpenAI's
supported list, so it suits deployments whose audio is already a supported format
or that accept the `whisper-1` legacy path.

### 3. The STT credential is **separate** from the LLM credential

The entire use case is *"my LLM can't hear"* — i.e. the LLM provider is **not**
the audio provider. So STT gets its **own** `STT_API_KEY`, decoupled from
`LLM_PROVIDER`/`*_API_KEY`. It is required when STT is enabled; there is no
implicit fallback to the LLM key (they're usually different vendors — e.g.
Anthropic LLM + Groq STT).

### 4. Config surface (all `.env`, opt-in)

```bash
STT_PROVIDER=            # "" = disabled (default) | groq | openai | <compatible>
STT_BASE_URL=            # optional; default derived from provider
STT_MODEL=whisper-large-v3-turbo
STT_API_KEY=             # required when enabled; separate from the LLM key
STT_LANGUAGE=            # optional ISO-639-1 hint; empty = auto-detect
STT_TIMEOUT_SECONDS=30
```

`STT_PROVIDER` empty ⇒ disabled ⇒ today's behavior exactly. Provider→base_url
defaults: `groq` → `https://api.groq.com/openai/v1`, `openai` →
`https://api.openai.com/v1`; an explicit `STT_BASE_URL` overrides both.

### 5. The transcript is rendered as a voice-message turn (no media block)

When STT succeeds, the inbound's text is set to the transcript and the audio
branch renders a normal `HumanMessage` that *tells the model it was a voice
message* (so tone matches) **without** an audio content block — the LLM can't
take one anyway. Both turn paths funnel through `_current_message`, so the
synchronous turn and the coalesced turn (ADR-008, which re-hydrates media from
the bucket) get STT identically.

## Consequences

**Positive**
- Voice notes become first-class for **any** LLM, not just Gemini — opt-in.
- Native-OGG default means **zero new system dependencies** (no ffmpeg).
- One client serves OpenAI/Groq/any compatible host; swapping providers is three
  env vars. No vendor lock.
- Cheapest viable provider as the default; cost is negligible for voice notes.
- Graceful degradation preserved: off or failing → exactly today's behavior.

**Negative / trade-offs**
- Adds an external call (and its latency, ~hundreds of ms) **before** the turn,
  only on audio + audio-less LLM + STT enabled. Bounded by `STT_TIMEOUT_SECONDS`;
  on timeout/error → text fallback.
- A second credential to manage (`STT_API_KEY`) when enabled.
- 25 MB request cap (Groq free tier); voice notes are far under it, but a guard +
  log is needed for the pathological case.
- Transcription quality is the provider's; mis-hears are possible. Acceptable —
  the alternative today is *nothing*.

## Alternatives considered

- **OpenAI `gpt-4o-mini-transcribe` as default** — rejected as default: OGG isn't
  in its supported formats, forcing an **ffmpeg** transcoding step (a system dep)
  for an opt-in feature. Kept as a selectable provider for already-supported
  formats.
- **ffmpeg transcoding (ogg→wav) in the core** — rejected: a native-OGG provider
  (Groq) removes the need entirely; shipping ffmpeg for an optional path is
  disproportionate. (Re-open only if a future required provider lacks OGG.)
- **Google Chirp 3 (Cloud STT v2)** — best OGG/Opus + multilingual, but auth is a
  GCP **service-account JSON**, which breaks the stack's "drop an API key in
  `.env`" simplicity. A clean **follow-up** as a dedicated provider adapter.
- **Gemini API via `GOOGLE_API_KEY`** — tempting (reuse an existing key) but its
  audio support lists OGG **Vorbis**, not Opus; voice notes are Opus. Too risky
  to default to.
- **OpenRouter transcription** — rejected: the transcription model catalogue is
  empty; not a viable path today.

## Follow-ups (out of scope)

- **Google Chirp 3 provider adapter** (service-account auth) for premium
  multilingual accuracy.
- **ffmpeg transcoding shim** — only if a future required provider lacks OGG.
- **Transcript persistence** — store the transcript in message `metadata` so the
  admin timeline shows what a voice note said (today it shows `has_media`).
- **Per-language model routing / diarization** (e.g. `gpt-4o-transcribe-diarize`).
