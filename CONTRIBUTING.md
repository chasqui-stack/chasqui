# Contributing to Chasqui

Chasqui is a base development stack for building custom AI agents on WhatsApp. Before contributing, read [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md).

The system is three services under the `chasqui-stack` org, orchestrated by this parent repo via git submodules:

| Repo | Role |
|------|------|
| `chasqui-stack/core` | FastAPI + LangGraph backend (the heart) |
| `chasqui-stack/admin` | React + Vite admin panel |
| `chasqui-stack/whatsapp` | PyWa gateway (WhatsApp channel) |
| `chasqui-stack/chasqui` | Parent: docs, generator, submodules |

## Project management (no Linear — GitHub-native)

- **Single board:** the org-level GitHub Project **"Chasqui Roadmap"** (`github.com/orgs/chasqui-stack/projects`) is the source of truth across all repos.
- **Issues live in the repo they belong to** (`core`/`admin`/`whatsapp`); epics and cross-cutting work go in the parent. Every issue is added to the board.
- **Board fields:** `Status`, `Sprint` (0–6), `Service`.
- **Labels:** `service:{core,admin,whatsapp,parent}`, `sprint:{0..6}`, `type:{feat,fix,docs,chore}`, `good first issue`.
- **Pick work** from the board's *Ready* column and comment to claim it.

## Planning with PRPs

We use the **prp-manager** skill (Product Requirements Prompts) to plan non-trivial features.

- Install: `npx skills add https://github.com/willywg/prp-manager --skill prp-manager`
- Repo: <https://github.com/willywg/prp-manager>
- PRPs live in the **parent repo** `PRPs/`. Write/refine a PRP *before* implementing a feature, then execute it in the relevant service repo. Each service's `AGENTS.md` points back here.

## Dev setup

Native local dev (PostgreSQL + `uv` + `npm`) is primary — see each service's `README.md`. A root `docker-compose.yml` is provided for a one-command spin-up if you prefer containers.

## Branch & PR flow

- One branch per issue: `feat/<short>`, `fix/<short>`, `docs/<short>`.
- Conventional commits.
- PR references its issue (`Closes #N`), CI green, and updates `AGENTS.md`/`README.md` when behavior changes.

## Testing

Each service ships a unit-test harness; tests are part of the Definition of Done.

- **core / whatsapp** (pytest): `make test` (or `uv run pytest`). Tests live in `tests/`.
- **admin** (vitest): `npm test`. Tests live beside source as `*.test.ts(x)`.

Write unit tests for new logic (services, tools, normalizers, pure helpers). End-to-end tests are optional but welcome — e.g. core `/ingest` against a throwaway test DB, or admin flows via Playwright.

## Writing a Tool Module

Tool modules are the extension point (each company's differentiator). See [`docs/ARCHITECTURE.md` §8](./docs/ARCHITECTURE.md) for the `ToolModule` contract and the *customer-defined collection + retriever* archetype.

## License

By contributing, you agree your contributions are licensed under the repository's `LICENSE`.
