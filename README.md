<div align="center">

# Chasqui

**The open-source stack for building custom AI agents on WhatsApp.**

</div>

Chasqui is a base development stack: generate a project, get a working WhatsApp AI agent out of the box — single conversation thread per contact, long-term memory, an FAQ knowledge base with RAG, an admin panel with editable prompts, and a pluggable tool/module system where you build each company's differentiating logic.

> Named after the *chasqui* — the relay messengers of the Inca empire who carried messages across the network.

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
| [`core`](https://github.com/chasqui-stack/core) | FastAPI · LangGraph · SQLModel · Postgres/pgvector | Ingest, orchestrator, memory, RAG, tool registry, admin auth |
| [`admin`](https://github.com/chasqui-stack/admin) | React 19 · Vite · Tailwind · shadcn/ui | Operator panel |
| [`whatsapp`](https://github.com/chasqui-stack/whatsapp) | PyWa 4.x (BSUID-first) · FastAPI | WhatsApp channel adapter |

Full design: **[`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md)**.

## Quickstart

```bash
# clone with submodules
git clone --recurse-submodules https://github.com/chasqui-stack/chasqui.git
cd chasqui

# core (port 8090)
cd core && uv sync && make migrate && make dev

# whatsapp gateway
cd whatsapp && uv sync && make dev

# admin (port 5191)
cd admin && npm install && npm run dev
```

Or `docker-compose up` for a one-command spin-up (Postgres+pgvector + all services).

## Generate a project

```bash
./generate-project.sh   # scaffolds a personalized, branded project for one company
```

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md). Tool modules are the extension point — see `docs/ARCHITECTURE.md` §8.

## License

[Apache-2.0](./LICENSE).
