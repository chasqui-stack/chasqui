# PRP: Sprint 5 — Admin Panel

> **Version:** 1.0
> **Created:** 2026-06-10
> **Status:** In Progress

---

## Goal

Operators configure the agent **without redeploys**: edit the system prompt, manage the FAQ knowledge base, toggle/configure tools (auto-forms from `config_schema()`), and inspect any conversation — all from the React admin. Two halves:

1. **Core** grows the missing admin endpoints (agent config, registry listing, read-only conversations/memories, FAQ search preview).
2. **Admin** grows the feature pages on top of the existing bootstrap (auth/layout/shadcn already in place) — **with i18n from day one** (react-i18next, es/en, ships in the first release — Willy's call 2026-06-10).

## Why

- Everything Willy does today via `curl`/SQL (edit prompt, load FAQs, toggle tools, read threads) becomes UI — the difference between a dev stack and an operable product.
- `config_schema()` → JSON Schema → auto-form closes the module-contract loop from Sprint 4: a new module's knobs appear in the admin **with zero admin-code changes**.
- i18n retrofitting costs 10x; starting with `t()` everywhere makes the first release bilingual for free.

## What

### Part A — Core endpoints (`chasqui-stack/core`)

| Endpoint | Verb | Purpose |
|---|---|---|
| `/admin/config` | GET/PUT | The `agent_config` singleton: `system_prompt`, `enabled_tools`, `tool_config`, `updated_at`. PUT is partial (any subset) and **validates each `tool_config` entry against the owning module's `config_schema()`** → 422 on bad values. |
| `/admin/tools` | GET | Registry listing: per module → `name`, tools (`name`, `description`, `enabled`), `config_schema` (Pydantic `model_json_schema()` or `null`), `config_key`, current config values. Feeds the auto-forms. |
| `/admin/contacts` | GET | Paginated contact list (search by name/external_id) with last-message preview + message count. |
| `/admin/contacts/{id}` | GET | Contact detail. |
| `/admin/contacts/{id}/messages` | GET | Read-only paginated timeline (direction, type, text, created_at, meta). Never returns `media_url` blobs (history is text-only anyway). |
| `/admin/contacts/{id}/memories` | GET | The contact's long-term memories (content + timestamps, **never the embedding**). |
| `/admin/modules/faq/search` | GET | NEW in the faq module: `?q=&top_k=` retrieval **preview** (entry + similarity score) so operators can test grounding before the agent does. |

### Part B — Admin SPA (`chasqui-stack/admin`)

1. **i18n foundation FIRST** — react-i18next + `src/locales/{en,es}.json`; retrofit the existing bootstrap (Login, Dashboard, Sidebar, Header) to `t()`; header language switcher; `VITE_DEFAULT_LOCALE` (+ `.env.example`). HARD RULE from this commit on: no hardcoded UI strings.
2. **Prompt editor** (`/prompt`) — monospace textarea over GET/PUT `/admin/config`, dirty-state guard, save toast. Effect next turn, no redeploy.
3. **FAQ manager** (`/faq`) — table (question, tags, has_embedding badge, updated_at), create/edit dialog (react-hook-form + zod), delete confirm, "Re-embed all" button, **search preview box** (the new endpoint) showing matches + similarity.
4. **Tools** (`/tools`) — module cards from `/admin/tools`: per-tool enable switch (missing key = enabled!) → PUT `enabled_tools`; **`SchemaForm`** auto-rendered from the JSON Schema → PUT `tool_config`.
5. **Conversations** (`/conversations`) — contacts list → read-only thread timeline (in/out bubbles, type badges) + memories side panel.
6. **Dashboard** — light stats from existing data (contacts, FAQ entries count) — no new core endpoint unless free.

### Success Criteria

- [ ] Editing the system prompt in the admin changes the agent's next reply with no redeploy (manual e2e).
- [ ] FAQ entries manageable from the UI; re-embed + search preview work.
- [ ] `faq_search` can be toggled off and its `top_k`/`min_similarity` edited from the UI via the auto-form — and the agent honors it next turn.
- [ ] Any contact's conversation (and memories) readable from the UI.
- [ ] The whole UI renders in Spanish AND English via the header switcher — zero hardcoded strings.
- [ ] UI follows `admin/DESIGN.md` (lime `#c2ef4e` on violet-midnight `#150f23`, Rubik/Monaco).
- [ ] `make test` (core) + `npm run build && npm run lint && npm test` (admin) green.

---

## All Needed Context

### Documentation & References

```yaml
- file: admin/DESIGN.md                       # MANDATORY before any UI — tokens, components
- file: admin/AGENTS.md                       # SPA-only, no BFF/SSR; npm; Node 22
- file: core/app/models/agent_config.py       # singleton; enabled_tools {tool: bool}, tool_config {key: {...}}
- file: core/app/modules/registry.py          # get_modules(); optional hooks via getattr
- file: core/app/modules/faq/__init__.py      # FaqSearchConfig + _tool_config reads tool_config["faq_search"]
- file: core/app/modules/faq/admin.py         # module-local schema pattern (has_embedding, never the vector)
- file: core/app/controllers/admin/auth.py    # admin controller + CurrentAdmin pattern
- file: core/tests/test_faq.py                # JWT-without-DB header pattern (create_admin_access_token)
- file: admin/src/lib/api-client.ts           # axios + refresh-token interceptor (done)
- file: admin/src/router.tsx                  # routes placeholder: "Prompts, FAQ/RAG, Tools, Conversations land in later sprints"
- ref:  saas-maker/admin, psicolab/admin      # prior art for pages/layout
- url:  https://react.i18next.com/            # useTranslation, I18nextProvider
```

### Key decisions (made in this PRP)

1. **Contact-centric conversation API** — the schema enforces one conversation per contact (`conversations.contact_id` UNIQUE), so the admin never needs a conversation id: `/admin/contacts/{id}/messages` joins through it. Simpler UI, no leaky abstraction.
2. **`/admin/config` is the singleton editor** (GET returns the one row, PUT partial-updates it). It lives in a new `app/controllers/admin/config.py`; tool toggles and tool configs are **the same row**, so the Tools page PUTs the same endpoint. `updated_at` returned for optimistic-concurrency display (not enforced).
3. **PUT validates `tool_config` against module schemas**: for each key in the payload, find the module whose `config_key` matches and run `schema.model_validate()`; unknown keys → 422 (typo protection), validation error → 422 with field detail. `enabled_tools` keys are validated against registered tool names.
4. **`config_key` ≠ module name**: faq's knobs live under `tool_config["faq_search"]` (the *tool* name, per Sprint 4). The registry listing must expose the key explicitly so the UI never guesses. Convention: a module's `config_schema()` pairs with a `config_key` attribute (default: module name); faq sets `config_key = "faq_search"`.
5. **`SchemaForm` is hand-rolled, not a library** — `config_schema()` models are flat by convention (FaqSearchConfig: int + float with ge/le). A ~100-line renderer for `string|number|integer|boolean` with min/max/description covers the contract; document the flatness convention in core/AGENTS.md instead of shipping a 50kB JSON-Schema-form dependency. (Omakase: convention over configuration.)
6. **i18n = react-i18next + flat JSON locales** (`src/locales/en.json`, `es.json` — the React analog of Rails' `config/locales/*.yml`). Default from `VITE_DEFAULT_LOCALE` (wizard-able in Sprint 6), persisted choice in `localStorage`, switcher in the header. Keys namespaced by page (`"faq.title"`, `"common.save"`).
7. **FAQ search preview lives in the module** (`faq/admin.py` gains `GET /search`) — keeps the self-contained proof intact; the core gains nothing FAQ-specific.
8. **Pagination convention**: `?limit=&offset=` + `{items, total}` envelope — same shape for contacts and messages; the admin gets one reusable `Pagination` hook/component (bootstrap already has the component).

### Known Gotchas

```python
# 1. enabled_tools semantics: MISSING KEY = ENABLED (new modules work out of the
#    box). The UI toggle must default to ON when the key is absent, and write
#    {tool: false} to disable — never prune keys it didn't touch.
# 2. agent_config is a singleton seeded by migration 003 — GET must handle the
#    fresh-DB case (create-if-missing) instead of 500ing.
# 3. JSONB partial update: assigning a mutated dict to the SQLModel attr is
#    fine, but session.refresh() does NOT autoflush — flush first (Sprint 3/4).
# 4. Never serialize embeddings (memories) or media data-URIs (messages) in
#    admin responses — has_embedding-style booleans only.
# 5. CORS: settings.cors_origins defaults to "*" — dev OK; document
#    CORS_ORIGINS=https://admin.<domain> for prod (Sprint 6 wizard).
# 6. Tests must not hit the network: admin-route tests reuse the
#    create_admin_access_token(uuid4(), ...) header pattern (no DB admin row).
# 7. react-i18next: components re-render on language change ONLY through
#    useTranslation() — strings read outside components (zod messages, consts)
#    must use i18n.t lazily or live inside render. Prefer t() in render.
# 8. zod error messages are UI strings too — map them via t() (params or
#    error maps), don't hardcode Spanish/English in schemas.
# 9. shadcn/ui components ship English defaults (e.g. dialog close sr-only) —
#    acceptable; the HARD RULE covers OUR strings.
# 10. Vite env vars must be prefixed VITE_ and are baked at build time —
#     VITE_DEFAULT_LOCALE is a default, not a runtime switch (that's the
#     localStorage choice).
# 11. The messages timeline orders by created_at: reuse the existing
#     ix_messages_conversation_created index (query by conversation_id +
#     ORDER BY created_at).
# 12. Willy's local DB has the older Spanish system prompt (it's data) — the
#     editor will show/overwrite exactly that; expected, not a bug.
```

---

## Implementation Blueprint

### Task order

```yaml
Task 1 - Core, config endpoints:
  - CREATE core/app/controllers/admin/config.py (GET/PUT /admin/config; partial update;
    tool_config validated via module config_schema; enabled_tools keys vs registered tools)
  - CREATE core/app/schemas/admin_config.py (AgentConfigResponse, AgentConfigUpdate)
  - MODIFY core/app/main.py (include router under /admin, JWT)
Task 2 - Core, registry listing:
  - MODIFY core/app/modules/faq/__init__.py (config_key = "faq_search" on FaqModule)
  - CREATE core/app/controllers/admin/tools.py (GET /admin/tools — modules, tools,
    json schema via model_json_schema(), config_key, current values, enabled flags)
Task 3 - Core, conversations read-only:
  - CREATE core/app/controllers/admin/contacts.py (list + detail + messages + memories;
    {items, total} envelope; limit/offset)
Task 4 - Core, FAQ search preview:
  - MODIFY core/app/modules/faq/admin.py (GET /search?q=&top_k= → [{entry, similarity}])
Task 5 - Core tests:
  - CREATE core/tests/test_admin_config.py (GET seeds-if-missing; PUT partial; 422 unknown
    config key; 422 schema violation; 422 unknown tool in enabled_tools; 401 without JWT)
  - CREATE core/tests/test_admin_contacts.py (pagination, search, messages order, memories
    without embeddings)
  - EXTEND core/tests/test_faq.py (search preview endpoint, incl. embeddings-outage → [])
Task 6 - Admin, i18n foundation:
  - npm i i18next react-i18next i18next-browser-languagedetector
  - CREATE src/i18n.ts + src/locales/{en,es}.json; wire in main.tsx
  - RETROFIT Login/Dashboard/Sidebar/Header to t(); language switcher in Header
  - CREATE .env.example (VITE_API_BASE_URL, VITE_DEFAULT_LOCALE)
Task 7 - Admin, prompt editor page (/prompt)
Task 8 - Admin, FAQ manager page (/faq) + search preview box
Task 9 - Admin, tools page (/tools) + SchemaForm component (flat JSON Schema renderer)
Task 10 - Admin, conversations page (/conversations + /conversations/:contactId)
Task 11 - Admin, dashboard cards + router/sidebar wiring + vitest (SchemaForm, locale
  completeness: en.json and es.json have identical key sets)
Task 12 - Docs (end-of-sprint rule):
  - core/README.md + AGENTS.md (new admin endpoints, config_key convention, flat-schema rule)
  - admin/README.md + AGENTS.md (i18n posture, page map)
  - sprint-05 doc checkboxes; parent AGENTS if needed
```

### API shapes (core)

```jsonc
// GET /admin/tools
{ "modules": [{
    "name": "faq",
    "tools": [{"name": "faq_search", "description": "...", "enabled": true}],
    "config_key": "faq_search",
    "config_schema": { /* JSON Schema or null */ },
    "config": {"top_k": 4, "min_similarity": 0.5}   // merged defaults + stored
}]}

// PUT /admin/config  (any subset)
{ "system_prompt": "...", "enabled_tools": {"faq_search": false},
  "tool_config": {"faq_search": {"top_k": 6}} }
```

## Validation Loop

```bash
cd core && make test                          # incl. new admin endpoint tests
cd admin && npm run build && npm run lint && npm test
# manual: core :8090 + admin :5191 →
#   login → edit prompt → send WhatsApp message → reply reflects new prompt
#   FAQ: create entry in UI → preview search → ask over WhatsApp (grounded)
#   Tools: disable faq_search → agent stops grounding; set top_k=1 → narrower
#   Conversations: read the thread just generated; switch es/en — full UI flips
```

## Final Checklist

- [x] /admin/config GET/PUT with schema-validated tool_config
- [x] /admin/tools registry listing (config_key + JSON Schema)
- [x] /admin/contacts (+messages, +memories) read-only
- [x] FAQ /search preview endpoint
- [x] Core tests green (74/74, was 54)
- [x] i18n foundation + bootstrap retrofit + switcher
- [x] Pages: prompt, faq, tools (SchemaForm), conversations, dashboard
- [x] Admin build/lint/test green; locale key-parity test (10/10)
- [ ] Manual e2e with Willy (prompt edit → behavior change, no redeploy)
- [ ] Docs updated (end-of-sprint rule)

## Anti-Patterns to Avoid

- ❌ Hardcoded UI strings (the i18n HARD RULE — everything through `t()`)
- ❌ FAQ-specific code in the core controllers (preview belongs to the module)
- ❌ Pruning `enabled_tools`/`tool_config` keys the UI didn't touch
- ❌ Returning embeddings or data-URIs from admin endpoints
- ❌ A JSON-Schema-form library for what a flat-schema convention solves
- ❌ Trusting `tool_config` writes without `config_schema()` validation
- ❌ BFF/SSR creep — the admin stays a Vite SPA on the core REST API
