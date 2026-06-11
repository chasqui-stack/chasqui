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

## Philosophy

**Omakase, the Rails way** ([*Rails is omakase*](https://dhh.dk/2012/rails-is-omakase.html), [*The Rails Doctrine*](https://rubyonrails.org/doctrine)): a curated, opinionated menu — substitutable dishes (LLM/embeddings via `.env`), but the menu has an owner (Postgres+pgvector is identity, ADR-002). Conventions over configuration; decisions get written down as ADRs.

## Planning & workflow

- **Architecture / design:** `docs/ARCHITECTURE.md`, `docs/design/DESIGN.md`.
- **Decisions → ADRs:** `docs/design/adr-NNN-*.md`. Any non-obvious architectural decision (DB, dims, protocols) gets an ADR in the same PR/commit — **docs-as-code, no GitHub wiki** (wikis drift; `docs/` is versioned and reviewed with the code).
- **End-of-sprint rule:** a sprint isn't closed until docs reflect it (service READMEs/AGENTS, ADRs for decisions taken, design docs for new archetypes).
- **PRPs:** feature planning lives here in `PRPs/` (prp-manager skill: `npx skills add https://github.com/willywg/prp-manager --skill prp-manager`). Write a PRP before non-trivial features.
- **Sprint plan:** `docs/sprints/` — **internal, gitignored** (contains local paths). Not public.
- **Tracking:** issues live in each service repo; epics/cross-cutting here. Board: *Chasqui Roadmap* (org-level Project), grouped by `Sprint`/`Service`.
- **Branches:** `feat/<short>`, `fix/<short>`, `docs/<short>`; conventional commits; PR `Closes #N`.
- **Releasing a version:** strict order — tag services (`vX.Y.Z`) → bump parent submodules + tag → CLI pins the stack tag and publishes to PyPI by pushing its own tag (trusted publishing). **The full ceremony is documented in the [`cli` repo's AGENTS.md](https://github.com/chasqui-stack/cli/blob/main/AGENTS.md)** — read it before tagging anything.

## Conventions

- Every repo has `AGENTS.md` (source of truth) + `CLAUDE.md` symlink.
- **English-only codebases** (code, comments, LLM-facing prompt strings, API errors) — agents localize via the DB system prompt ("reply in the user's language"); the only user-facing literal (`FALLBACK_REPLY`) lives in `.env`. UI i18n is an admin-frontend concern.
- Don't put business logic outside `core/`.
- Don't break the canonical contract or commit secrets (`.kamal/secrets`, `.env`).
- Submodules pin commits — bump them intentionally and commit the pointer here.

## Local dev

Native (primary): PostgreSQL + `uv` + `npm` per service README. `docker-compose up` for collaborators.
