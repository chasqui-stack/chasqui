# PRP: Sprint 12 — STT fallback for inbound audio

> **Version:** 1.0
> **Created:** 2026-06-15
> **Status:** Draft
> **Tracks:** chasqui#15 (STT fallback for audio).
> **Decisions to record:** ADR-010 (opt-in STT, gated on `caps.audio=False`;
> one OpenAI-compatible client, **Groq default** for native OGG/Opus + cost;
> separate `STT_API_KEY`; graceful degradation preserved).

---

## Goal

When a contact sends a **voice note** and the configured LLM **can't hear**
(`caps.audio = False` — Anthropic, plain OpenAI GPT, most of the registry),
**transcribe the audio before the turn** so the agent answers its content as if
it were text. Today that case dead-ends in a graceful *"please type it"* fallback
(`orchestrator._current_message`). Native-audio models (Gemini) are unaffected —
they keep getting the audio block directly.

End state: an **opt-in** STT step (`STT_PROVIDER` set) that, for audio + an
audio-less LLM, calls an **OpenAI-compatible** transcription endpoint
(**Groq `whisper-large-v3-turbo`** by default — native OGG/Opus, ~$0.00067/min),
sets the inbound's text to the transcript, and lets the existing turn machinery
render it. Disabled or failing → **exactly today's** graceful fallback. STT never
breaks a turn.

## Why

- **Voice notes are first-class on messaging.** On WhatsApp/Telegram a huge share
  of inbound is audio. With an audio-less LLM the agent is currently useless for
  it — it can only ask the user to retype.
- **Cheap, well-understood, opt-in.** Transcription is a solved, sub-cent step.
  Gating it behind `STT_PROVIDER` keeps the default install unchanged.
- **The format constraint forces the provider choice (ADR-010).** WhatsApp/
  Telegram voice is **OGG/Opus**; OpenAI's transcription API doesn't list OGG, so
  it would need ffmpeg. **Groq lists OGG natively** and is the cheapest — so it's
  the default, via a client that also speaks OpenAI by swapping `STT_BASE_URL`.
- **It's the last open issue** for the v0.2.x audio story (native audio shipped
  with the multimodal turn; this completes the matrix for non-audio models).

## What

One new service + a pre-turn hook, gated by one env var:

1. **A transcription service** (`app/services/transcription.py`): `httpx`
   multipart `POST {base_url}/audio/transcriptions`, `response_format=text`.
   `stt_enabled()` + `async transcribe(audio: bytes, mime: str) -> str | None`.
2. **A pre-turn STT pass** in the orchestrator: for an `audio` inbound, when
   `not caps.audio` and STT is enabled and media bytes are present, transcribe and
   set `inbound.text` to the transcript. Runs in **both** turn paths (synchronous
   `run_turn` and coalesced `run_coalesced_turn`).
3. **Render the transcript** in `_current_message`'s audio `else` branch as a
   voice-message turn (no audio content block — the LLM can't take one).

**Escape hatch / default:** `STT_PROVIDER=""` (default) ⇒ disabled ⇒ today's
exact graceful fallback. No new dependency, no behavior change for anyone who
doesn't opt in.

### Success Criteria
- [ ] Audio inbound + audio-less LLM + STT enabled → transcript reaches the agent
      as a Human turn; the reply addresses the spoken content.
- [ ] Native-audio LLM (Gemini) path **unchanged** — never calls STT.
- [ ] STT disabled (`STT_PROVIDER=""`) → byte-for-byte today's graceful fallback.
- [ ] STT error/timeout/oversize → graceful fallback (turn never fails); logged.
- [ ] Works in **both** turn paths: synchronous and coalesced (ADR-008,
      media re-hydrated from the bucket).
- [ ] Default provider Groq `whisper-large-v3-turbo`, OGG sent as-is (no
      transcoding); OpenAI selectable via `STT_BASE_URL`/`STT_MODEL`.
- [ ] `STT_API_KEY` is separate from the LLM key; required only when enabled.
- [ ] ADR-010 written; ARCHITECTURE updated (audio degradation now has an STT
      branch); `.env.example` + READMEs/AGENTS reflect STT; CLI wizard opt-in
      (cross-repo) tracked.

---

## All Needed Context

### Documentation & References
```yaml
- file: docs/design/adr-010-stt-audio-fallback.md
  why: the decision this PRP implements (provider choice, gating, key separation)
- file: core/app/services/orchestrator.py
  why: _current_message (audio branch ~154-177) is the degradation point; run_turn
       (~237) and run_coalesced_turn (~282) are the two hooks; _parse_data_uri
       (~110) + _message_to_inbound (~260) give the media bytes
- file: core/app/core/llm_capabilities.py
  why: caps.audio — STT runs only when False (resolve_capabilities)
- file: core/app/core/config.py
  why: add stt_* settings (mirror the openai_base_url pattern at line 41)
- file: core/app/services/channel_send.py
  why: the httpx.AsyncClient pattern to mirror (timeout, error mapping) ~87-96
- file: core/app/core/storage.py
  why: audio/ogg → ogg mapping (~36) confirms inbound mime; get_media for bytes
- url: https://console.groq.com/docs/speech-to-text
  why: default provider — models, ogg in supported formats, OpenAI-compatible
       endpoint api.groq.com/openai/v1/audio/transcriptions, 25MB cap
- url: https://platform.openai.com/docs/api-reference/audio/createTranscription
  why: the de-facto multipart shape (model, file, response_format) — same on Groq
```

### Current flow (graceful degradation, today)
```
audio inbound → _current_message(inbound, caps):
   if caps.audio and media:            # Gemini → native audio block
       HumanMessage([text, {audio block}])
   else:                               # ← audio-less LLM dead-ends here
       HumanMessage("[voice message you cannot listen to. Ask them to write it.]")
```

### Desired flow (STT enabled, audio-less LLM)
```
audio inbound → run_turn / run_coalesced_turn:
   inbound = await _transcribe_if_needed(inbound, caps)     # ← new pre-turn pass
        # only when: inbound.type=="audio" and not caps.audio
        #            and stt_enabled() and media bytes present
        # success → inbound.text = transcript
        # error/oversize/timeout → inbound unchanged (logged)
   _current_message(inbound, caps):
       if caps.audio and media: ...native...                # unchanged
       else:
           if inbound.text:                                 # ← transcript (or caption)
               HumanMessage('voice message; transcript: "<text>". Respond naturally...')
           else:
               HumanMessage("[voice message ... ask them to write it]")   # unchanged
```

### Known gotchas & conventions
```python
# 1. caps.audio gate is mandatory. NEVER transcribe when caps.audio is True —
#    Gemini answers audio better natively and it'd be wasted spend. STT is the
#    NON-audio path only.

# 2. _current_message is SYNC; transcription is async (httpx). Do STT as an async
#    PRE-PASS that returns a (possibly) modified InboundMessage, then call the
#    sync _current_message. Do NOT try to await inside _current_message.

# 3. Media bytes: _parse_data_uri returns (mime, b64_str). base64.b64decode it to
#    bytes for the multipart 'file'. In the coalesced path media is re-hydrated by
#    _message_to_inbound (ADR-008/003); if storage is unset there are no bytes →
#    STT can't run → graceful fallback (same as native-audio-without-storage).

# 4. OGG goes to Groq AS-IS — no transcoding. Send filename "audio.ogg" + the
#    real mime in the multipart so the provider's sniffer is happy. For OpenAI
#    (no OGG support) that's the operator's caveat (ADR-010), not our problem.

# 5. STT failure is NEVER fatal. Wrap the call: any httpx/timeout/HTTP-error/
#    oversize → log a warning, return None → inbound unchanged → existing text
#    fallback. Mirror channel_send / notify_service best-effort posture.

# 6. 25MB cap (Groq free tier). Guard: if len(bytes) > STT_MAX_BYTES → skip + log
#    (voice notes are tiny; this is the pathological guard only).

# 7. Separate credential. STT_API_KEY is independent of the LLM key (the whole
#    point is LLM != audio provider). If STT_PROVIDER set but STT_API_KEY empty →
#    log once at startup and treat STT as disabled (don't 500 mid-turn).

# 8. response_format=text returns a plain-text body (not JSON) — read response.text,
#    strip. (json mode returns {"text": ...}; text mode is simpler.)

# 9. No new dependency: httpx is already in pyproject. Do NOT add the openai or
#    groq SDK.
```

---

## Implementation Blueprint

### Config (`app/core/config.py`)
```python
# Speech-to-text fallback (ADR-010). Transcribe inbound audio when the LLM lacks
# native audio input (caps.audio False). Empty provider = DISABLED → the existing
# graceful "ask for text" fallback. OpenAI-compatible API; Groq default (native
# OGG/Opus, cheapest). The credential is SEPARATE from the LLM key on purpose.
stt_provider: str = ""            # "" disabled | "groq" | "openai" | "<compatible>"
stt_base_url: str | None = None   # default derived from provider when empty
stt_model: str = "whisper-large-v3-turbo"
stt_api_key: str | None = None    # required when enabled; not the LLM key
stt_language: str | None = None   # ISO-639-1 hint; empty = auto-detect
stt_timeout_seconds: float = 30.0
stt_max_bytes: int = 25 * 1024 * 1024  # provider request cap (Groq free tier)

# provider → default base_url (an explicit stt_base_url overrides):
#   groq   → https://api.groq.com/openai/v1
#   openai → https://api.openai.com/v1
```

### Transcription service (`app/services/transcription.py`, new)
```python
_DEFAULT_BASE_URLS = {
    "groq": "https://api.groq.com/openai/v1",
    "openai": "https://api.openai.com/v1",
}

def stt_enabled() -> bool:
    return bool(settings.stt_provider and settings.stt_api_key)

def _base_url() -> str:
    return (settings.stt_base_url
            or _DEFAULT_BASE_URLS.get(settings.stt_provider, "")).rstrip("/")

async def transcribe(audio: bytes, mime: str) -> str | None:
    """Best-effort transcription. None on any failure (caller falls back)."""
    if not stt_enabled() or not _base_url():
        return None
    if len(audio) > settings.stt_max_bytes:
        logger.warning("Audio %d bytes over STT cap — skipping transcription", len(audio))
        return None
    ext = storage.ext_for_mime(mime) or "ogg"           # reuse storage map
    files = {"file": (f"audio.{ext}", audio, mime)}
    data = {"model": settings.stt_model, "response_format": "text"}
    if settings.stt_language:
        data["language"] = settings.stt_language
    headers = {"Authorization": f"Bearer {settings.stt_api_key}"}
    try:
        async with httpx.AsyncClient(timeout=settings.stt_timeout_seconds) as client:
            r = await client.post(f"{_base_url()}/audio/transcriptions",
                                  data=data, files=files, headers=headers)
            r.raise_for_status()
    except httpx.HTTPError as exc:
        logger.warning("STT request failed (%s) — text fallback", exc)
        return None
    text = r.text.strip()
    return text or None
```

### Orchestrator hook (`app/services/orchestrator.py`)
```python
async def _transcribe_if_needed(inbound: InboundMessage,
                                caps: ModelCapabilities) -> InboundMessage:
    """STT pre-pass: audio + audio-less LLM + STT on → set inbound.text."""
    if inbound.type != "audio" or caps.audio or not transcription.stt_enabled():
        return inbound
    media = _parse_data_uri(inbound.media_url)
    if not media:
        return inbound
    mime, b64 = media
    transcript = await transcription.transcribe(base64.b64decode(b64), mime)
    if not transcript:
        return inbound
    logger.info("STT transcribed inbound audio (%d chars)", len(transcript))
    return inbound.model_copy(update={"text": transcript})  # keep type="audio"

# run_turn: before building messages
inbound = await _transcribe_if_needed(inbound, caps)
# (caps already computed via _capabilities(); compute it once and pass down)

# run_coalesced_turn: map the pass over each inbound in the batch
#   batch_inbounds = [await _transcribe_if_needed(_message_to_inbound(m)..., caps) ...]

# _current_message audio else-branch becomes:
else:
    if inbound.text:   # STT transcript (or a rare audio caption)
        return HumanMessage(
            f'The user sent a voice message. Transcript: "{inbound.text}". '
            "Respond to its content naturally. Do NOT say you transcribed it."
        )
    return HumanMessage(
        "[The user sent a voice message you cannot listen to. "
        "Kindly ask them to write it as text.]"
    )
```

### Tasks (in execution order)
```yaml
Task 0 (GATE — do first): verify Groq accepts a real WhatsApp OGG/Opus voice note
  - curl a captured .ogg to api.groq.com/openai/v1/audio/transcriptions with
    whisper-large-v3-turbo, response_format=text. Expect 200 + transcript.
  - This validates ADR-010's core premise (native OGG) before writing code.
    (If it failed, ADR-010's ffmpeg follow-up would re-open — not expected.)

Task 1: Config
  - MODIFY: core/app/core/config.py (stt_* settings above)
  - MODIFY: core/.env.example (document the STT block + "separate key" note)

Task 2: ext_for_mime helper (if not already exposed)
  - MODIFY/CONFIRM: core/app/core/storage.py exposes mime→ext (reuse the audio/ogg
    map at ~36); add a small ext_for_mime(mime) accessor if missing

Task 3: Transcription service
  - CREATE: core/app/services/transcription.py (stt_enabled, transcribe)

Task 4: Orchestrator wiring
  - MODIFY: core/app/services/orchestrator.py
    * add _transcribe_if_needed(inbound, caps)
    * call it in run_turn and run_coalesced_turn (compute caps once)
    * update _current_message audio else-branch to render a transcript

Task 5: Startup sanity
  - MODIFY: core/app/main.py (lifespan) — if stt_provider set but stt_api_key
    empty (or base_url unresolved), log a clear WARNING that STT is effectively
    disabled. Do not crash.

Task 6: Tests (see Validation Loop)

Task 7: Docs + ADR
  - CREATE: docs/design/adr-010-stt-audio-fallback.md  [done with this PRP]
  - MODIFY: docs/ARCHITECTURE.md (audio handling: native → STT → graceful)
  - MODIFY: core/AGENTS.md (+ symlinked CLAUDE.md) media/orchestrator notes,
    README env table

Task 8 (cross-repo): CLI wizard opt-in  → file cli#N
  - Add a wizard question "Enable speech-to-text for voice notes (LLMs without
    native audio)?" → writes STT_PROVIDER/STT_MODEL/STT_API_KEY to core/.env.
  - Default suggestion groq + whisper-large-v3-turbo. Mirrors cli#1 provisioning.
  - Skills: note STT in the chasqui-cli skill env table (sibling skills repo).
```

---

## Validation Loop

### Level 0: Provider gate (manual, once)
```bash
# Capture a real voice note from WhatsApp/Telegram (audio/ogg, Opus) as voice.ogg
curl -s https://api.groq.com/openai/v1/audio/transcriptions \
  -H "Authorization: Bearer $GROQ_API_KEY" \
  -F model=whisper-large-v3-turbo -F response_format=text -F file=@voice.ogg
# Expect: HTTP 200 + the spoken text. Confirms native OGG (ADR-010 premise).
```

### Level 1: Core unit/integration (`cd core && uv run pytest -q`)
```python
# test_transcription.py / test_orchestrator_stt.py  (mock httpx, no real key)
- stt_disabled_returns_none:
    STT_PROVIDER="" → transcribe() returns None without any HTTP call
- transcribe_posts_multipart_and_returns_text:
    enabled → POSTs to {base_url}/audio/transcriptions with model+file+response_format;
    200 text body → stripped transcript returned (httpx mocked / respx)
- transcribe_swallows_http_error:
    mocked 500 / timeout → returns None (no raise)
- oversize_audio_skipped:
    bytes > stt_max_bytes → None, no HTTP call, warning logged
- audio_less_llm_uses_transcript:
    caps.audio=False + STT on + audio inbound → _transcribe_if_needed sets
    inbound.text; _current_message renders the transcript HumanMessage
- native_audio_llm_never_transcribes:
    caps.audio=True → _transcribe_if_needed is a no-op (no HTTP call) — audio
    block path unchanged
- stt_off_keeps_graceful_fallback:
    STT_PROVIDER="" + audio + caps.audio=False → the exact "ask them to write it"
    HumanMessage (byte-for-byte today)
- coalesced_path_transcribes_each_audio:
    run_coalesced_turn over a batch with an audio message → transcript present in
    the assembled turn (media re-hydrated, mocked)
# Hermeticity: mock transcription.transcribe / httpx — no real Groq key in CI.
```

### Level 2: Live e2e (Telegram, the cheap channel)
```bash
# Generated project: LLM=anthropic (caps.audio=False), STT_PROVIDER=groq,
# STT_API_KEY set. Send a voice note in Telegram.
# Expect: the agent answers the SPOKEN content (not "please type it").
# Logs: "STT transcribed inbound audio (N chars)" then a normal turn.
# Then unset STT_PROVIDER, restart → voice note → graceful "please type it".
# Sanity: LLM=google gemini (caps.audio=True) + STT set → NO STT call (native).
```

---

## Final Checklist
- [ ] Level 0 gate passed: Groq transcribes a real WhatsApp/Telegram OGG note
- [ ] `stt_*` settings in config + `.env.example` (separate-key note)
- [ ] `transcription.py`: enabled-guard, multipart, best-effort (None on failure)
- [ ] `_transcribe_if_needed` gated on `type==audio and not caps.audio and enabled`
- [ ] Hooked in BOTH `run_turn` and `run_coalesced_turn`; caps computed once
- [ ] `_current_message` renders a transcript; no-transcript fallback unchanged
- [ ] STT error/timeout/oversize never breaks a turn (logged)
- [ ] Startup warns when STT_PROVIDER set without a usable key/base_url
- [ ] Tests green in CI (hermetic, httpx mocked); live Telegram e2e both modes
- [ ] ADR-010 + ARCHITECTURE + READMEs/AGENTS updated; CLI wizard tracked (cli#N)

---

## Anti-Patterns to Avoid
- ❌ Don't transcribe when `caps.audio` is True — native audio is better + paid-for.
- ❌ Don't await inside the sync `_current_message` — STT is a pre-pass.
- ❌ Don't let an STT failure raise — it must degrade to the text fallback.
- ❌ Don't transcode OGG (no ffmpeg) — Groq takes it natively; that's why it's default.
- ❌ Don't reuse the LLM key implicitly — STT_API_KEY is separate (LLM ≠ audio vendor).
- ❌ Don't add the openai/groq SDK — httpx multipart is enough (already a dep).
- ❌ Don't persist/log the audio bytes or the key — bytes are big, the key is secret.

---

## Notes

**Default provider = Groq `whisper-large-v3-turbo` (decided 2026-06-15, ADR-010).**
The deciding factor is **native OGG/Opus** (WhatsApp/Telegram voice) with **no
transcoding** — which removes ffmpeg as a system dependency for an opt-in feature
— plus the lowest cost (~$0.00067/min) and an OpenAI-compatible API so the same
client serves OpenAI by swapping `STT_BASE_URL`. OpenRouter was rejected (empty
transcription catalogue); Google Chirp 3 is a follow-up (GCP service-account auth
doesn't fit `.env`-key simplicity).

**Why opt-in and not on-by-default:** STT needs a second credential and is a paid
external call. The default install stays zero-config; operators turn it on when
their LLM can't hear and they want voice notes handled.

**Transcript persistence is a deliberate follow-up** (ADR-010): storing the
transcript in message `metadata` would let the admin timeline show what a voice
note said. Not in this PRP — keep the change surgical.

**Out of scope (future):** Google Chirp 3 adapter; ffmpeg shim (only if a future
required provider lacks OGG); diarization; TTS for *outbound* audio (a separate,
larger feature — this PRP is inbound STT only).
