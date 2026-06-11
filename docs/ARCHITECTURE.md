# Chasqui — Architecture

> **Status:** design draft. **Codename:** `Chasqui` — after the relay messengers of the Inca empire, who carried messages across the network. A fitting metaphor for a messaging-agent gateway.

## 1. What Chasqui is

Chasqui is a **base development stack** for building custom AI agents on WhatsApp. You generate a project from it (one project per company), and you get a working conversational agent out of the box — single conversation thread per contact, long-term memory, an FAQ knowledge base with RAG, an admin panel with editable prompts, and a **tool/module system** where you build the differentiating logic for that specific company.

**The philosophy is omakase** — in the Rails sense ([*Rails is omakase*](https://dhh.dk/2012/rails-is-omakase.html), [*The Rails Doctrine*](https://rubyonrails.org/doctrine)): a curated, opinionated menu chosen by someone with skin in the game, instead of a buffet of abstractions. You can substitute dishes (the LLM and embeddings are `.env` swaps), but the menu has an owner: Postgres + pgvector is the stack's identity (ADR-002), conventions beat configuration, and the energy goes into your agent's differentiating logic — not into re-deciding plumbing. Common things (prompts, FAQs, enabling tools) are **configurable from the admin panel**; genuinely new logic is **code** you add through the tool system.

It ships with a project generator (**`uvx chasqui new`**, the [`cli`](https://github.com/chasqui-stack/cli) repo) that scaffolds a personalized, configured project — LLM, embeddings/dims, database location, WhatsApp credentials — ready to run locally and deploy. A `chasqui generate module` subcommand scaffolds new tool modules, `rails generate`-style.

## 2. Design principles

1. **Proven, low-surprise topology.** Gateway → Core → Admin. Three deployables — the minimum that the runtime constraints allow. "Boring" here is a virtue (see *Choose Boring Technology*): well-understood components, predictable failure modes, low operational surprise. No speculative services.
2. **Channel-pluggable.** The gateway is an *adapter*. WhatsApp (PyWa) is channel #1; web widget, Telegram, Instagram are future adapters that never touch the core.
3. **Tool-pluggable.** Agent tools are each company's differentiator. A clean **Tool Registry** lets you add them as self-contained modules without modifying the core.
4. **Config vs. code.** Frequently-changing things (prompts, FAQs, toggling tools) are editable from the admin with no redeploy. New behavior is code you add via the tool system.
5. **The core is the heart.** FastAPI + LangGraph + Postgres. All business logic lives here. Logic is never scattered across services.

## 3. Service topology

```
                         channel #1: WhatsApp (PyWa)
                         [future: web widget, telegram, IG]
                                    │
                                    ▼
┌──────────────────────┐   POST /ingest         ┌────────────────────────────┐
│   GATEWAY            │   (canonical message)   │   CORE BACKEND             │
│   PyWa + FastAPI     │ ──────────────────────► │   FastAPI                  │
│   ──────────────     │                         │   ────────────            │
│   • receives webhooks│ ◄────────────────────── │   • LangGraph orchestrator │
│   • normalizes to    │   canonical             │   • Tool Registry          │──► PostgreSQL
│     canonical message│   response(s)           │   • Memory + RAG (pgvector)│    (+ pgvector)
│   • renders output   │                         │   • Admin auth             │
│     back to channel  │                         │   • Business services      │
│   • stateless        │                         │   • Tool modules           │
└──────────────────────┘                         └────────────────────────────┘
                                                              ▲
                                                              │ REST (JWT)
                                                 ┌────────────────────────────┐
                                                 │   ADMIN PANEL              │
                                                 │   React 19 + Vite SPA      │
                                                 │   ────────────            │
                                                 │   • editable prompts       │
                                                 │   • FAQ / RAG (CRUD)       │
                                                 │   • tool enable/config     │
                                                 │   • conversation inspection│
                                                 └────────────────────────────┘
```

### 3.1 Gateway (PyWa) — channel adapter

- **Single responsibility:** translate between WhatsApp and the core's canonical contract. Receive webhooks, normalize (text, audio, image, buttons), POST to the core, then render the core's response back to the channel.
- **Stateless.** No database, no business logic. If it restarts, no state is lost.
- **Fast.** Meta enforces a webhook timeout → the gateway acks `200` immediately and processes against the core asynchronously.
- **PyWa 4.x (beta, BSUID-first)** — see §10 for the BSUID identity change.
- Adding a channel = a new adapter that produces/consumes the **same canonical message**.

### 3.2 Core Backend (FastAPI + LangGraph) — the heart

- All logic: models, tables, services, orchestrator, tools, memory, RAG, admin auth.
- A single PostgreSQL connection, with `pgvector` for RAG embeddings.
- Exposes:
  - `POST /ingest` — canonical entry point from any gateway.
  - REST API for the admin (JWT).
- Structure: `core / models / schemas / controllers / services` + a `modules/` package for tool modules.

### 3.3 Admin Panel (React + Vite SPA)

- React 19, TypeScript, Vite, TanStack Query, **Tailwind CSS** + **shadcn/ui**.
- Talks **directly** to the core over REST. No BFF.
- Visual language follows [`docs/design/DESIGN.md`](./design/DESIGN.md) (a Sentry-inspired token set: deep violet-midnight canvas, electric-lime accent, Rubik for UI / Monaco for code). The `DESIGN.md` format follows the [google-labs-code/design.md](https://github.com/google-labs-code/design.md) methodology — point your coding agent at it before writing UI.

## 4. Authentication model

Deliberately minimal:

- **Only admins authenticate.** The admin panel is for operators. JWT-based admin auth with a small number of admin accounts is enough — no organizations, no member roles, no self-serve signup.
- **End users never log in.** WhatsApp contacts are just `contacts` (§6). They never reach the admin panel and have no credentials. Identity comes from the channel (their BSUID), not from an auth system.

## 5. Canonical message contract (gateway ↔ core)

The seam that makes the system multi-channel. The core never knows WhatsApp exists — it only speaks this contract, so a future web channel is "just another adapter."

```jsonc
// Inbound: gateway → core  POST /ingest
{
  "channel": "whatsapp",            // "whatsapp" | "web" | "telegram" | ...
  "contact": {
    "external_id": "<bsuid>",       // BSUID for WhatsApp (see §10), channel-scoped id otherwise
    "wa_id": "51999888777",         // optional, may be null under BSUID
    "display_name": "Juan Pérez",
    "metadata": {}
  },
  "message": {
    "type": "text",                 // text | audio | image | button | ...
    "text": "Do you have a store in Miraflores?",
    "media_url": null,
    "raw": {}                       // original channel payload if needed
  },
  "received_at": "<iso8601>"
}
```

```jsonc
// Outbound: core → gateway (agent reply; may be 1..N messages)
{
  "messages": [
    { "type": "text", "text": "Yes, at Av. Larco 123. Want the hours?" }
    // { "type": "buttons", ... }, { "type": "image", ... }
  ],
  "conversation_id": "<uuid>"
}
```

An **empty `messages` list is silence** — that's how human-mode conversations
(§5.1) go quiet on every channel with zero gateway changes.

### 5.1 Canonical outbound contract — `POST /send` (ADR-004)

The mirror of `/ingest`, for **admin-initiated** messages (the human-handoff
inbox). Every gateway exposes it with the same `INTERNAL_API_KEY`; the core
resolves the URL per channel from `.env` (`CHANNEL_WHATSAPP_SEND_URL` — one
var per channel, zero core code for a new one):

```jsonc
// core → gateway  POST /send
{
  "contact": { "channel": "whatsapp", "external_id": "<bsuid>", "wa_id": "51999888777" },
  "message": {
    "type": "text",                  // text | image | document | audio
    "text": "Hi, this is Ana from the team…",   // body, or caption for image/document
    "media_url": null,               // base64 data: URI for media — the MIRROR of inbound
    "filename": null                 // documents: what the channel shows the user
  }
}
// 200 → { "status": "sent", "message_id": "<channel id>" }
// errors → { "detail": { "code": "NO_WA_ID" | "WINDOW_EXPIRED" | "SEND_FAILED" | "UNSUPPORTED_TYPE" | "INVALID_MEDIA", "message": "…" } }
```

Error `code`s flow through to the admin panel (e.g. WhatsApp's 24h
customer-service window → `WINDOW_EXPIRED`). Conversation ownership lives in
`conversations.mode` (`"agent" | "human"`): the ingest pipeline checks it
FIRST and runs no agent turn in human mode. Full rationale:
`docs/design/adr-004-conversation-mode-outbound-send.md`.

## 6. Core domain model

- **`contacts`** — the person on the other side. Unique by `(channel, external_id)`, where `external_id` is the **BSUID** for WhatsApp. `wa_id` is stored when available but is no longer the primary key (§10).
- **`conversations`** — a **single thread per contact**. `mode` (`"agent" | "human"`, indexed) says who owns the replies (ADR-004); orchestrator state lives in `conversation_state` (or embedded).
- **`messages`** — full inbound/outbound history with type + metadata. This is the LLM context.
- **`memories`** — long-term memory extraction so the agent doesn't forget the past (summaries, facts about the contact). A memory retriever injects what's relevant into the prompt. Backed by `pgvector`.
- **Orchestrator (LangGraph):** a configurable state machine. The base ships a minimal graph (router → tools → respond); each project extends it.

## 7. What the base ships vs. what you build

### In the base
- Admin authentication (admins only).
- Single conversation thread per contact + message history + long-term memory.
- **FAQ knowledge base with RAG** (mini-RAG over `pgvector`): admins load FAQs/documents, they're embedded, and a retriever tool answers from them. This is a common need, so it's built in.
- **Editable prompts** from the admin (system prompt, persona, rules).
- **Tool Registry** + a couple of generic example tools (e.g. `faq_search`, a lead-capture / human-handoff tool).
- WhatsApp gateway (PyWa) wired up.
- Admin panel: prompts, FAQ-RAG, tool enable/config, conversation inspection.
- Project generator + Kamal deploy config + docker-compose for collaborators.

### What you build per company (the differentiator)
- **Custom tools** via the Tool Registry (§8).
- Reference example — a **"Commercial Locations"** module (a CRUD of stores in the admin + a retriever tool the agent can call). Not in the base; it lives in the docs as a reference template for the *"customer-defined collection + auto-generated retriever"* pattern, which generalizes to product catalogs, branches, hours, pricing, policies.
- Particular integrations (CRMs, scheduling, the client's own APIs).
- Adjustments to the orchestrator graph.

## 8. Tool Registry — the differentiating piece

Tools are what make each agent unique. Chasqui builds on LangChain/LangGraph's native tool model rather than inventing its own.

### 8.1 How a tool is defined (LangChain native)

Simple tools use the `@tool` decorator; complex inputs use a Pydantic `args_schema`. Type hints define the input schema and the docstring becomes the description the model reads.

```python
from langchain.tools import tool, ToolRuntime
from pydantic import BaseModel, Field

class StoreLookupInput(BaseModel):
    city: str = Field(description="City to search stores in")

@tool(args_schema=StoreLookupInput)
def store_lookup(city: str, runtime: ToolRuntime) -> str:
    """Look up commercial locations in a given city."""
    tenant_cfg = runtime.context           # immutable per-project config
    # ...query DB, return human-readable text or structured data
    return "Av. Larco 123, Miraflores — open 9am-9pm"
```

Key LangChain mechanics Chasqui relies on:
- **`@tool` / `StructuredTool.from_function`** to declare tools; `args_schema` (Pydantic) for validated, typed inputs.
- **`ToolRuntime`** (injected, not part of the schema) for access to graph `state`, immutable `context` (per-project config), persistent `store` (`BaseStore` long-term memory), and `tool_call_id`. Reserved param names: `runtime`, `config`.
- **`bind_tools([...])`** to expose the active tool set to the model.
- **`ToolNode`** (`langgraph.prebuilt`) to execute tool calls inside the LangGraph orchestrator.
- A tool may return a **`Command`** (`langgraph.types`) to update graph state alongside a `ToolMessage`.
- **Error handling / dynamic tool filtering** via agent middleware (`wrap_tool_call`, `wrap_model_call`) — e.g. enable/disable tools per project at runtime.

### 8.2 The Chasqui module contract

A **module** wraps one or more tools plus the things they need (tables, admin endpoints). The core discovers registered modules at startup and feeds their tools to the orchestrator. Adding behavior = adding a module; the core stays untouched.

```python
# Sketch of a tool module
class ToolModule(Protocol):
    name: str
    def register_tools(self) -> list[BaseTool]: ...        # the LangChain tools
    def register_models(self) -> list[type]: ...           # optional: SQLModel tables + Alembic
    def register_admin_routes(self, router) -> None: ...    # optional: admin CRUD/config UI backing
    def config_schema(self) -> type[BaseModel] | None: ...  # optional: per-project settings
```

Each module is self-contained: its tool schema, its optional tables/migrations, its optional admin UI, and its enable/config state per project. This is the open-source extension point — contributors add capabilities as modules.

### 8.3 The reusable archetype

The "Commercial Locations" example is really an archetype: *"customer-defined collection + auto-generated retriever tool."* The same shape covers most "custom tool" requests (catalogs, branches, hours, pricing, policies). A semi-generic version of this module — admin defines a schema, fills data, a retriever tool is auto-exposed — is on the roadmap so this turns into configuration instead of code for the common case.

## 9. Repository strategy

Everything lives under the GitHub organization **`chasqui-stack`**. Because the org namespace already says "chasqui", repos drop the prefix and stay clean (`chasqui-stack/core` rather than `chasqui-stack/chasqui-core`).

A parent repo holds the orchestration; each service is a **git submodule** pointing at its own repository, for independent deployment.

```
chasqui-stack/chasqui          # parent repo (orchestration, docs, generator)
├── core/      → chasqui-stack/core       (submodule: FastAPI backend)
├── admin/     → chasqui-stack/admin      (submodule: React/Vite admin)
├── whatsapp/  → chasqui-stack/whatsapp   (submodule: PyWa gateway)
├── docs/                      # architecture, design, ADRs, roadmap
├── generate-project.sh        # project generator (rsync + variable substitution)
├── docker-compose.yml         # one-command spin-up for collaborators
└── README.md
```

| Repo | Role |
|------|------|
| `chasqui-stack/chasqui` | Parent: docs, generator, compose, submodules |
| `chasqui-stack/core` | FastAPI + LangGraph backend |
| `chasqui-stack/admin` | React/Vite admin panel |
| `chasqui-stack/whatsapp` | PyWa gateway (WhatsApp channel) |

Future channel adapters become their own repos under the same org (`chasqui-stack/web`, `chasqui-stack/telegram`), each consuming the canonical message contract.

## 10. Tech stack

| Layer | Technology |
|-------|------------|
| Gateway | Python, **PyWa 4.x (beta, BSUID-first)**, FastAPI, httpx, Sentry |
| Core | FastAPI, SQLModel, PostgreSQL + pgvector, Alembic, JWT, LangChain, LangGraph |
| Admin | React 19, TypeScript, Vite, TanStack Query, Tailwind CSS, shadcn/ui |
| LLM | Configurable via LangChain (Gemini / Claude / etc.) |
| Package mgrs | `uv` (core/gateway), `npm` (admin) |
| Infra | Docker, docker-compose (collaborators), **Kamal 2** (deploy, part of the stack), Sentry |

### WhatsApp identity: BSUID

WhatsApp is moving to **Business-Scoped User IDs (BSUID)**, a breaking change that replaces phone-number-based `wa_id` as the primary user identifier. PyWa 4.x is BSUID-first:

- `User.bsuid` is the **primary identifier**; `User.wa_id` becomes **optional** (may be `None`).
- New `User` fields: `bsuid`, `username`, `parent_bsuid`, `country_code`.
- Default `WhatsApp(user_identifier_priority="bsuid -> wa_id")`; replies/blocks use BSUID internally.
- Install beta: `uv add "pywa --prerelease=allow"` (or `pip install -U pywa --pre`).

**Design implication:** `contacts.external_id` stores the **BSUID**, not `wa_id`. `wa_id` is kept as an optional secondary attribute. This is baked into the canonical contract (§5) and domain model (§6) from day one to avoid a painful migration later.

## 11. Local development & deployment

**Local dev (native, primary).** PostgreSQL, Python (`uv`) and Node (`npm`) run directly on the developer machine — no containers required for day-to-day work.

```bash
# core
cd core && uv sync && make migrate && make dev
# gateway
cd whatsapp && uv sync && make dev
# admin
cd admin && npm install && npm run dev
```

**docker-compose** is provided as a one-command spin-up for collaborators who prefer not to install the toolchain locally.

**Deployment — Kamal 2** is part of the stack. Each service has `config/deploy.yml` + `.kamal/secrets`, proxied by hostname with auto-TLS.

| Service | Subdomain (example) | Port |
|---------|---------------------|------|
| Core    | `api.<domain>`      | 8090 |
| Gateway | `wsp.<domain>`      | 8000 |
| Admin   | `admin.<domain>`    | 3000 |

## 12. Roadmap (post v1)

- **Web channel** (e.g. a logged-in web agent) — a new adapter + an SSR frontend where streaming/SSR earn their place.
- Semi-generic **"customer-defined collection + retriever"** module.
- Additional channel adapters (Telegram, Instagram).
- Background workers / queues (`arq` / Celery) only when needed — FastAPI background tasks until then.
