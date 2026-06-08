# AGENTS.md — Chasqui (parent)

Chasqui is a base development stack for building custom AI agents on WhatsApp. This is the **parent repo**: it orchestrates three services as git submodules and holds the docs, planning, and project generator.

> **Read [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) before working.** It is the source of truth for the design.

## Services (git submodules)

| Path | Repo | Stack | Role |
|------|------|-------|------|
| `core/` | `chasqui-stack/core` | FastAPI + LangGraph + Postgres/pgvector | The heart: ingest, orchestrator, memory, RAG, tool registry, admin auth |
| `admin/` | `chasqui-stack/admin` | React 19 + Vite + Tailwind + shadcn/ui | Operator panel (prompts, FAQ-RAG, tool config, conversations) |
| `whatsapp/` | `chasqui-stack/whatsapp` | PyWa 4.x (BSUID-first) + FastAPI | Stateless WhatsApp channel adapter |

Services talk only through the **canonical message contract** (`docs/ARCHITECTURE.md` §5). The core never knows a channel exists.

## Planning & workflow

- **Architecture / design:** `docs/ARCHITECTURE.md`, `docs/design/DESIGN.md`.
- **PRPs:** feature planning lives here in `PRPs/` (prp-manager skill: `npx skills add https://github.com/willywg/prp-manager --skill prp-manager`). Write a PRP before non-trivial features.
- **Sprint plan:** `docs/sprints/` — **internal, gitignored** (contains local paths). Not public.
- **Tracking:** issues live in each service repo; epics/cross-cutting here. Board: *Chasqui Roadmap* (org-level Project), grouped by `Sprint`/`Service`.
- **Branches:** `feat/<short>`, `fix/<short>`, `docs/<short>`; conventional commits; PR `Closes #N`.

## Conventions

- Every repo has `AGENTS.md` (source of truth) + `CLAUDE.md` symlink.
- Don't put business logic outside `core/`.
- Don't break the canonical contract or commit secrets (`.kamal/secrets`, `.env`).
- Submodules pin commits — bump them intentionally and commit the pointer here.

## Local dev

Native (primary): PostgreSQL + `uv` + `npm` per service README. `docker-compose up` for collaborators.
