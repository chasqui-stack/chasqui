<div align="center">

# Chasqui

**The omakase stack for building custom AI agents on WhatsApp.**

[![PyPI](https://img.shields.io/pypi/v/chasqui?label=chasqui%20CLI)](https://pypi.org/project/chasqui/)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](./LICENSE)

</div>

```bash
uvx chasqui new my-agent
```

One command, one wizard, and you have a running WhatsApp AI agent: a single
conversation thread per contact, long-term memory, an FAQ knowledge base with
RAG, **multimodal in and out** (images, documents, voice notes), a **human
handoff inbox** where operators take over and reply from the panel, lead
capture, and a pluggable tool/module system where you build each company's
differentiating logic.

> Named after the *chasqui* — the relay messengers of the Inca empire who
> carried messages across the network.

**Omakase, the Rails way.** Like [Rails](https://rubyonrails.org/doctrine),
Chasqui is a curated menu, not a buffet: someone already chose pieces that
work well together so your energy goes into your agent, not into plumbing.
You can substitute dishes — the LLM and the embeddings are a `.env` swap
(Gemini, Claude, GPT, OpenRouter, Ollama) — but the menu has an owner:
Postgres + pgvector is the stack's identity, and conventions beat
configuration. Decisions are written down in [`docs/design/`](./docs/design/)
as ADRs.

## Quickstart

Prerequisites: [`uv`](https://docs.astral.sh/uv/), Node 22, and a PostgreSQL
with the pgvector extension (or use the generated docker-compose).

```bash
uvx chasqui new my-agent      # the wizard asks: LLM, embeddings, where's
cd my-agent                   # your Postgres, WhatsApp creds (skippable),
                              # language, first admin — then provisions
                              # everything (deps, db, migrations, seed)

cd core && make dev           # API on :8090
cd whatsapp && make dev       # WhatsApp gateway on :8000
cd admin && npm run dev       # operator panel on http://localhost:5191
```

Nothing the wizard wrote is locked in: it all lives in each service's `.env`
(the LLM swap applies on the next message — no redeploy). The one
provision-time choice is `EMBEDDING_DIM`, baked into the schema by the first
migrate ([ADR-001](./docs/design/adr-001-embeddings-provider-dims.md)).

To hack on the stack itself instead, clone this repo with
`--recurse-submodules` and follow each service's README;
`docker compose up` brings up Postgres + core + admin in one command.

## Architecture

Three services, orchestrated by this parent repo as git submodules:

```
   WhatsApp ──►  whatsapp/ (PyWa gateway)  ──► core/ (FastAPI + LangGraph) ──► Postgres + pgvector
                  stateless adapter             the heart: ingest, agent,        ▲
                                                memory, RAG, tool registry       │ REST
                                                                          admin/ (React + Vite SPA)
```

| Service | Stack | Role |
|---------|-------|------|
| [`core`](https://github.com/chasqui-stack/core) | FastAPI · LangGraph · SQLModel · Postgres/pgvector | Ingest, orchestrator, memory, RAG, tool registry, handoff inbox, admin auth |
| [`admin`](https://github.com/chasqui-stack/admin) | React 19 · Vite · Tailwind · shadcn/ui | Operator panel: prompts, FAQ, tools, conversations, inbox, leads |
| [`whatsapp`](https://github.com/chasqui-stack/whatsapp) | PyWa 4.x (BSUID-first) · FastAPI | WhatsApp channel adapter (in and out) |
| [`cli`](https://github.com/chasqui-stack/cli) | typer · PyPI `chasqui` | `chasqui new` / `chasqui generate module` |

Services talk only through a **canonical message contract** — the core never
knows a channel exists, which is why a human-mode conversation is silent on
*every* channel and a new channel is one gateway away. Full design:
**[`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md)**.

## Extending it

Capabilities are **Tool Modules**: self-contained packages the core
auto-discovers, each contributing tools (with their prompt in the
docstring), optional tables, admin routes, and config knobs that the panel
auto-renders as a form — zero frontend work.

```bash
chasqui generate module price_check --with-models --with-admin
```

Guide: [`docs/MODULES.md`](./docs/MODULES.md).

## Deploying

Kamal 2, three services, auto-TLS — guide: [`docs/DEPLOY.md`](./docs/DEPLOY.md).

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md).

## License

[Apache-2.0](./LICENSE).
