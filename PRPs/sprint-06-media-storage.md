# PRP: Sprint 6 — Media Storage (S3-compatible)

> **Version:** 1.0
> **Created:** 2026-06-10
> **Status:** Completed

---

## Goal

Images and audio **survive the turn**: stored in an S3-compatible bucket, visible/playable in the admin conversation timeline. Today the multimodal LLM processes media in-turn but nothing is persisted — the operator sees `[image]` type badges. After this sprint:

1. **Core** uploads inbound media to the bucket on ingest (`media/<contact_id>/<message_id>.<ext>` → object key in `messages.media_url`) and serves it via JWT-protected presigned URLs.
2. **Admin** renders `<img>` inline and `<audio controls>` in the chat bubbles, falling back to today's type badge when there's no object.
3. **Parent** ships a local S3-compatible bucket (RustFS) in `docker-compose.yml` so collaborators get one for free.

Storage is **optional**: unset → exactly today's behavior. **Ships in the first release** (Willy's call, 2026-06-10). Design locked in [ADR-003](../docs/design/adr-003-media-storage.md).

## Why

- "El panel mostrando `[imagen]` no es algo completo" — first-release quality bar.
- One boto3 client + 4 env vars covers AWS S3 / R2 / Spaces / B2 / local RustFS (omakase, ADR-002 shape — ask *where is your bucket*, never *which engine*).
- The gateway keeps inlining base64 — canonical contract untouched, zero `whatsapp/` changes, future channels inherit persistence for free.
- Document-RAG (post-MVP) reuses this exact layer.

## What

### Part A — Core (`chasqui-stack/core`)

| Piece | Behavior |
|---|---|
| `app/core/storage.py` | boto3 S3 client from `.env`; `is_configured()`, `put_media(key, data, content_type)` (async via `asyncio.to_thread`), `presigned_get(key, expires)`. Graceful no-op posture when unconfigured. |
| Ingest persistence | If configured and inbound `media_url` is a `data:` URI: parse mime + base64, upload `media/<contact_id>/<message_id>.<ext>` with `ContentType`, store the **key** in `messages.media_url`. Failures → log + NULL (never break the turn). Unconfigured → NULL (today). |
| `GET /admin/media/{message_id}` | JWT (admin router). 404 if message missing or no stored object; 503 if storage unconfigured but a key exists. Returns `{"url": <presigned>, "expires_in": 300}` as **JSON** (not redirect — see gotcha 1). |
| `has_media` flag | `MessageItem` grows `has_media: bool` = media_url holds a stored key (`media/` prefix). The raw key/blob is still never serialized. |
| Settings + `.env.example` | `storage_endpoint_url`, `storage_bucket`, `storage_access_key`, `storage_secret_key`, `storage_region` (+ `storage_configured` property). |

### Part B — Admin (`chasqui-stack/admin`)

- `useMediaUrl(messageId, enabled)` hook → `GET /admin/media/{id}`, `staleTime` < presign expiry.
- `MessageMedia` component in the conversation bubbles: `type === "image"` → `<img>` (click = open full in new tab), `type === "audio"` → `<audio controls>`; loading skeleton; on error or `!has_media` → today's type badge. All new strings via `t()` (es/en), as always.

### Part C — Parent

- Local S3-compatible service in `docker-compose.yml` (S3 + console ports, volume) + a one-shot init job that creates the bucket — `docker-compose up` = working local bucket, zero setup. **RustFS, not MinIO** (archived 2026-04-25, Willy's catch — see ADR-003): Apache-2.0, MinIO-shaped (:9000/:9001, env creds); bucket bootstrap via `amazon/aws-cli` (`mc` is MinIO tooling too).
- Docs: ADR-003 (done), core README/AGENTS storage section, sprint doc ticked.

### Success Criteria

- [x] Sending a photo/audio over WhatsApp → object lands in the bucket; admin timeline shows the actual image / playable audio.
- [x] Storage unconfigured → everything works exactly as today (`media_url` NULL, badges in the panel).
- [x] Upload failure (bad creds, bucket down) → turn still answers; message persisted with NULL `media_url`; error logged.
- [x] `docker-compose up` gives a working local bucket with zero extra setup.
- [x] `make test` (core) + `npm run build && npm run lint && npm test` (admin) green.

---

## All Needed Context

### Documentation & References

```yaml
- file: docs/design/adr-003-media-storage.md     # the decision — read first
- file: core/app/services/ingest_service.py      # the data: URI is discarded at persist today (lines ~105-117)
- file: core/app/models/message.py               # media_url ALREADY EXISTS (1024 chars) → NO migration this sprint
- file: core/app/schemas/admin_contacts.py       # MessageItem (gets has_media; media_url stays out)
- file: core/app/controllers/admin/contacts.py   # admin controller pattern (router included under JWT admin_router)
- file: core/app/core/config.py                  # Settings — add the storage block
- file: core/tests/conftest.py                   # transactional-rollback DB isolation; JWT-without-DB header pattern
- file: admin/src/pages/ConversationDetailPage.tsx  # the bubbles to extend
- file: admin/src/hooks/useContacts.ts           # TanStack hook pattern
- file: admin/DESIGN.md                          # before any UI
- file: docker-compose.yml                       # parent compose to extend
- url:  https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3/client/generate_presigned_url.html
```

### Key decisions (made in this PRP, on top of ADR-003)

1. **`boto3` + `asyncio.to_thread`, not `aioboto3`** — boto3 is sync; the upload is one call per media message inside a turn that already spends seconds on the LLM. `to_thread` keeps the event loop free without adopting aioboto3's version-pinning treadmill. `generate_presigned_url` is pure local computation (no network) — called sync.
2. **`media_url` stores the object KEY, not a URL** (`media/<contact_id>/<message_id>.<ext>`). Endpoint/bucket can change without rewriting rows. `has_media` / the media endpoint recognize stored objects by the `media/` prefix.
3. **The media endpoint returns JSON, not a redirect** — `<img src>` can't send the JWT header, so the SPA fetches `{url}` with axios and feeds the presigned URL (direct to bucket) to `<img>`/`<audio>`. Plain `<img>`/`<audio>` loads aren't CORS-gated, so the bucket needs no CORS config for this.
4. **Extension from the data-URI mime** via a small explicit map (`image/jpeg→jpg`, `image/png→png`, `image/webp→webp`, `audio/ogg→ogg`, `audio/mpeg→mp3`, `video/mp4→mp4`, `application/pdf→pdf`) with `mimetypes.guess_extension` then `.bin` as fallback. `ContentType` set on upload so the presigned GET serves the right header (required for `<audio>`).
5. **Upload happens at persist time, after the turn** — same place the data: URI is discarded today. The `Message` row is constructed first (client-side uuid4) so the key can embed `message_id` before flush.
6. **Outbound media is out of scope** — agent replies are text; outbound `media_url` (if a tool ever sets one) persists as-is and is not flagged `has_media` unless it's a stored key.
7. **No size guard added** — Meta already bounds media (~5MB images / ~16MB audio) and the base64 already flows through `/ingest` today; nothing new to limit.
8. **Bucket bootstrap via a one-shot `amazon/aws-cli` service** in compose (retry loop until the storage container accepts connections, then `s3 mb`) — no manual console step, no MinIO-owned tooling.
9. **`STORAGE_PUBLIC_ENDPOINT_URL` (added during implementation):** presigned URLs embed the signing endpoint — in compose the core uploads via `http://storage:9000` but the browser only reaches `http://localhost:9000`. Optional override; presigning is local, so a second boto3 client for it is free. Unset = same endpoint (native dev, AWS, R2).

### Known Gotchas

```python
# 1. <img>/<audio> tags CANNOT send Authorization headers — never design the
#    media endpoint as a redirect the browser follows from a src attribute.
#    JSON {url} + axios fetch, then src = presigned URL.
# 2. Presigned URLs expire (300s): the admin hook must use staleTime < expiry
#    (refetch on remount/expiry) — never cache the URL in component state.
# 3. boto3 import cost is real (~150ms): import inside storage.py only; the
#    client is a lazy module-level singleton so unconfigured deployments and
#    tests never touch it.
# 4. Self-hosted S3 (RustFS et al.) needs endpoint_url + path-style; AWS neither.
#    Config: endpoint_url=None → AWS default. boto3 handles path-style
#    automatically when endpoint_url is set (s3={"addressing_style": "path"}
#    explicitly, to be safe with self-hosted stores).
# 5. Upload failures NEVER break the turn (ADR-003): try/except around the
#    whole upload, log, persist media_url=NULL — the embeddings pattern.
# 6. The turn runs BEFORE inbound persist (orchestrator history must not see
#    the current message) — upload at persist keeps that invariant; do not
#    move ingest steps around.
# 7. tests: stub the storage module (monkeypatch put_media/presigned_get/
#    is_configured), never the boto3 SDK internals; no network in tests.
# 8. settings is a module-level singleton read at import — tests that need
#    "configured storage" monkeypatch app.core.storage functions or settings
#    attrs, not env vars.
# 9. data: URI parsing: split on the first comma; header is
#    "data:<mime>;base64". Malformed URI → treat as upload failure (log+NULL).
# 10. MessageItem keeps media_url OUT of the schema (Sprint 5 rule: the admin
#     never receives blobs/keys) — has_media boolean only.
# 11. Admin i18n HARD RULE: loading/error/fallback strings for media go
#     through t() in BOTH en.json and es.json (key-parity test will fail
#     otherwise).
# 12. Node 22 for any admin command: source ~/.nvm/nvm.sh && nvm use.
```

---

## Implementation Blueprint

### Tasks (in execution order)

```yaml
Task 1: ADR-003                                          # DONE
  - CREATE: docs/design/adr-003-media-storage.md

Task 2: Core settings + storage service
  - MODIFY: core/app/core/config.py        (storage_* fields + storage_configured property)
  - CREATE: core/app/core/storage.py       (lazy boto3 client, is_configured, put_media, presigned_get, ext-from-mime)
  - MODIFY: core/.env.example              (storage block, local-bucket defaults commented)

Task 3: Ingest persists media
  - MODIFY: core/app/services/ingest_service.py  (data: URI → upload → key in media_url; try/except log+NULL)

Task 4: Media endpoint + has_media
  - CREATE: core/app/controllers/admin/media.py  (GET /admin/media/{message_id} → {url, expires_in})
  - MODIFY: core/app/schemas/admin_contacts.py   (MessageItem.has_media)
  - MODIFY: core/app/controllers/admin/contacts.py (populate has_media)
  - MODIFY: core/app/main.py                     (include media router under admin_router)

Task 5: Core tests
  - CREATE: core/tests/test_storage.py           (ext map, key shape, unconfigured no-op, stubbed client)
  - CREATE: core/tests/test_admin_media.py       (auth, 404s, 503 unconfigured, 200 stubbed presigned)
  - MODIFY: core/tests/test_ingest.py or new     (ingest-with-media persists key; failure → NULL; unconfigured → NULL)

Task 6: Admin media rendering
  - CREATE: admin/src/hooks/useMediaUrl.ts
  - CREATE: admin/src/components/conversations/MessageMedia.tsx
  - MODIFY: admin/src/pages/ConversationDetailPage.tsx (render media in bubbles)
  - MODIFY: admin/src/types (Message type: has_media), locales en/es
  - TEST:   build + lint + vitest

Task 7: Parent local bucket + docs
  - MODIFY: docker-compose.yml (rustfs storage + aws-cli bucket-init)
  - MODIFY: core/README.md + core/AGENTS.md (storage section; update "never persisted" media note)
  - MODIFY: docs/sprints/sprint-06-media-storage.md (tick tasks)

Task 8: E2E with Willy
  - Photo + audio over WhatsApp → object in the local bucket, visible/playable in the panel.
```

---

## Validation Loop

```bash
# Core
cd core && make test                       # or .venv/bin/python -m pytest

# Admin (Node 22!)
cd admin && source ~/.nvm/nvm.sh && nvm use
npm run build && npm run lint && npm test

# Integration (local bucket — RustFS)
docker compose up -d storage storage-init  # bucket auto-created
# set STORAGE_* in core/.env → restart core → send photo by WhatsApp
# check: object in the storage console (:9001), image visible in admin
```

---

## Final Checklist

- [x] ADR-003 written
- [x] storage.py + settings + .env.example
- [x] Ingest uploads + persists key; failure/unconfigured → NULL
- [x] GET /admin/media/{message_id} + has_media
- [x] Core tests green (storage, ingest-with-media, endpoint)
- [x] Admin renders image/audio with badge fallback (i18n'd)
- [x] Admin build + lint + tests green
- [x] Local bucket (RustFS) in docker-compose with bootstrap
- [x] Docs updated (README/AGENTS, sprint doc)
- [x] Manual e2e with Willy (photo + audio over WhatsApp → RustFS bucket → visible/playable in the panel; accepted 2026-06-10)

---

## Anti-Patterns to Avoid

- ❌ Don't build a storage-backend abstraction (ADR-003: boto3 S3-compatible only).
- ❌ Don't change the canonical contract or touch `whatsapp/` — base64 inlining stays.
- ❌ Don't let an upload failure 500 the turn (log + NULL, like embeddings).
- ❌ Don't serialize `media_url` keys or blobs in admin responses (`has_media` only).
- ❌ Don't make the media endpoint a redirect consumed from `<img src>` (no JWT header there).
- ❌ Don't feed stored media back into LLM history (text-only, unchanged).
- ❌ Don't hardcode UI strings (i18n HARD RULE).
