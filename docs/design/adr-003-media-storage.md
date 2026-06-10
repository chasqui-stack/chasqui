# ADR-003: Media storage — S3-compatible via boto3, optional, core-owned

> **Status:** Accepted — 2026-06-10
> **Context:** Sprint 5's conversation inspection made the gap visible: media
> is processed in-turn (multimodal LLM) but never persisted, so the admin
> timeline shows `[image]` badges instead of the actual photo/audio. Storage
> ships in the first release (Willy's call). Question: which storage
> abstraction, who owns it, and how is media served to the panel?

## Decision

**One client — the S3-compatible API via `boto3` — configured by `.env`,
owned by the core, optional by design, served through short-lived presigned
URLs.** No multi-backend storage abstraction.

## Rationale

1. **S3-compatible ≠ AWS.** The same four env vars (`STORAGE_ENDPOINT_URL`,
   `STORAGE_BUCKET`, `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY`, optional
   `STORAGE_REGION`) cover AWS S3, Cloudflare R2, DigitalOcean Spaces,
   Backblaze B2 **and a local S3-compatible container in docker-compose**
   for dev. An Active-Storage-style engine abstraction (disk/GCS/Azure
   adapters) would buy a permanent test matrix for users we don't have —
   same lesson as ADR-002. The Sprint 8 wizard asks *"where is your
   bucket"*, never *"which storage engine"*.
2. **The core owns storage** (business logic stays in core). The gateway
   keeps inlining media as base64 `data:` URIs in canonical `media_url` —
   the message contract (§5) does not change, and a future Telegram/web
   gateway gets persistence for free. On inbound persist, if storage is
   configured and the message carries a `data:` URI, the core uploads the
   object and stores the **object key** (`media/<contact_id>/<message_id>.<ext>`)
   in `messages.media_url` (today always NULL for media). No migration —
   the column already exists.
3. **Optional by design.** Storage unset → exactly today's behavior (media
   processed in-turn, not persisted) — degraded, not broken. Upload failures
   never break the turn (log + NULL), the same resilience pattern as
   embeddings.
4. **Serving = presigned GET, never public buckets.** `GET
   /admin/media/{message_id}` (JWT) returns a short-lived presigned URL as
   JSON; the browser loads `<img>`/`<audio>` straight from the bucket. The
   bucket stays private, the SPA never holds storage credentials, and the
   core never proxies bytes. (JSON, not a redirect: `<img src>` cannot send
   the `Authorization` header, so the SPA must fetch the URL with axios
   first.) Split-horizon wrinkle: presigned URLs embed the signing endpoint,
   so when the core reaches the bucket through an internal hostname the
   browser can't resolve (docker-compose: `http://minio:9000`), the optional
   `STORAGE_PUBLIC_ENDPOINT_URL` signs with the browser-reachable one.
5. **The LLM context stays text-only.** History fed to the agent is
   unchanged; re-feeding stored media to the model is out of scope.

## What about local-disk storage?

Considered and rejected. A disk backend looks free but isn't: the core
would have to serve the bytes itself (no presigned URLs → an authenticated
streaming endpoint + caching), files don't survive a container redeploy
without volume choreography (Kamal), it breaks the moment the core scales
past one host, and it has no backup story. Meanwhile the "I don't have S3"
case is already covered twice: **unset = degraded mode** (the stack works,
media just isn't persisted) and **free S3-compatible tiers** (Cloudflare R2
10 GB, Backblaze B2 10 GB) cost less effort than operating a disk backend
well. Revisit only if a zero-dependency dev-toy mode shows real demand
(same trigger shape as ADR-002's SQLite note).

## Local dev bucket: RustFS, not MinIO (2026-06-10)

MinIO — the classic choice — **archived its open-source repo on
2026-04-25** (read-only, "no longer maintained", successor is the
commercial AIStor). Pointing an OSS stack's compose at an abandoned image
is a non-starter, so the dev bucket is **RustFS** (Apache-2.0, actively
developed, MinIO-shaped: S3 API on :9000, console on :9001, creds via
`RUSTFS_ACCESS_KEY`/`RUSTFS_SECRET_KEY`; bucket bootstrapped by a one-shot
`amazon/aws-cli` service — `mc` is MinIO Inc. tooling, also avoided).
RustFS is young; that's acceptable because **the dev bucket is throwaway by
design** — production points `STORAGE_*` at a managed bucket, and ADR-003's
whole point is that swapping the compose service is a 10-line change
(verified: boto3 upload → presign → download passes against it). Mature
fallback if RustFS disappoints: SeaweedFS (Apache-2.0, 10+ years).

## What this layer also buys

Document-RAG (post-MVP backlog) reuses exactly this client for uploaded
PDFs/docs — no new decisions needed when it's picked up.

## Revisit trigger

If operators need outbound media (agent or human sending images/audio), the
same client covers upload, but the canonical outbound contract grows a real
`media_url` — that's a Sprint 7+ (handoff) or backlog conversation, not a
storage one.
