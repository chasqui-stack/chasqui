# Deploying Chasqui with Kamal 2

Three services, one VM (or three — Kamal doesn't care), auto-TLS via the
Kamal proxy. Each service ships its own `config/deploy.yml` +
`.kamal/secrets.example`; `chasqui new` can pre-fill the placeholders
(domain, registry user, server IP) if you answer the deploy questions.

| Service | Hostname (convention) | Port | Image |
|---|---|---|---|
| core | `api.<domain>` | 8090 | `<registry-user>/<project>-core` |
| whatsapp gateway | `wsp.<domain>` | 8000 | `<registry-user>/<project>-whatsapp` |
| admin (static SPA) | `admin.<domain>` | 3000 | `<registry-user>/<project>-admin` |

Postgres + pgvector runs as a Kamal **accessory** of the core
(`pgvector/pgvector:pg17` — never the plain `postgres` image, ADR-001/002:
`config/init.sql` runs `CREATE EXTENSION vector` on first boot).

## Prerequisites

- A VM you can SSH into as the `ssh.user` from `deploy.yml` (Docker gets
  installed by `kamal setup`).
- DNS `A` records for `api.`, `wsp.` and `admin.<domain>` → the server IP
  (the proxy provisions Let's Encrypt certificates automatically).
- A Docker registry account (Docker Hub works).
- [Kamal 2](https://kamal-deploy.org) on your machine: `gem install kamal`.

## 1. Core (deploy this first)

```bash
cd core
cp .kamal/secrets.example .kamal/secrets   # fill it in
# edit config/deploy.yml: server IP, registry username, host, LLM/embedding
# provider+model (the non-secret env), CORS_ORIGINS=https://admin.<domain>
kamal setup        # first time: provisions Docker, the proxy, the pgvector accessory
kamal deploy       # subsequent deploys
kamal app exec "alembic upgrade head"      # or the `db-migrate` alias
kamal app exec "python scripts/create_admin.py --email you@x.com --name You --password ..."
```

Notes:
- `INTERNAL_API_KEY` must be **byte-identical** in the core's and the
  gateway's secrets.
- `EMBEDDING_DIM` is baked into the schema by the first migrate (ADR-001) —
  set it (env `clear:` block) before that ever runs.
- Media storage (`STORAGE_*`) is optional and must be a **managed** bucket
  (S3/R2/Spaces/B2 — ADR-003: never a self-hosted accessory in prod).
- The accessory publishes Postgres on `127.0.0.1:5434` only; the core
  reaches it as `<service>-core-db:5432` on the Docker network.

## 2. WhatsApp gateway

```bash
cd whatsapp
cp .kamal/secrets.example .kamal/secrets   # WA_* + the same INTERNAL_API_KEY
# edit config/deploy.yml: server IP, registry username, host,
# CORE_URL=https://api.<domain>
kamal setup && kamal deploy
```

Then point the Meta webhook at production (Meta Business Suite → WhatsApp →
Configuration): callback `https://wsp.<domain>/`, verify token =
`WA_VERIFY_TOKEN` from your secrets. `WA_PHONE_ID` is required for operator
replies from the inbox (ADR-004).

## 3. Admin panel

The SPA bakes the API URL **at build time** (`VITE_API_BASE_URL` is a
builder arg, not a runtime env):

```bash
cd admin
cp .kamal/secrets.example .kamal/secrets   # registry password only
# edit config/deploy.yml: server IP, registry username, host, and
# builder.args.VITE_API_BASE_URL=https://api.<domain>
kamal setup && kamal deploy
```

Changing the API URL means re-deploying (it's in the bundle).

## Day 2

```bash
kamal app logs -f        # or the `logs` alias
kamal deploy             # ship a new version
kamal rollback <version>
kamal app exec -i bash   # shell into the container (`app-terminal` alias)
```

Order matters on first install: **core → gateway → admin** (the gateway
health-checks against a reachable core; the admin needs the API's CORS to
allow `https://admin.<domain>`).
