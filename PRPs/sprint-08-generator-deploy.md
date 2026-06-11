# PRP: Sprint 8 — `chasqui new` CLI, Deploy & OSS Release

> **Version:** 1.1
> **Created:** 2026-06-11
> **Status:** Completed — accepted by Willy 2026-06-11 (full e2e from the
> published package: `uvx chasqui new` → wizard → agent answers on
> WhatsApp, FAQ-RAG and handoff inbox working; `chasqui` 0.1.x on PyPI)

---

## Goal

The last sprint of the first release. The product is done (Sprints 0–7);
this sprint makes it **usable by people who aren't us**:

1. **`uvx chasqui new <name>`** — a Rails-`new`-style wizard that produces a
   branded, configured, locally-running project from one command. New repo
   `chasqui-stack/cli`, PyPI package `chasqui` (verified free 2026-06-11).
2. **`chasqui generate module <name>`** — à la `rails generate`: scaffolds
   the Sprint-3 tool-module anatomy inside a project.
3. **Deploy story documented** — Kamal 2 guide for the three services
   (core + pgvector accessory, gateway, admin static), per-service
   `deploy.yml` de-templated.
4. **OSS release** — licenses, READMEs, CONTRIBUTING, module-authoring
   guide, repos public, **`v0.1.0` tags everywhere** + CLI on PyPI.

Design locked in [ADR-005](../docs/design/adr-005-cli-generator.md): thin
CLI, wizard-asks-only-`.env`-vars, degit-style fetch at a pinned stack tag,
single-repo output, best-effort resumable provisioning.

## Why

- Today "try Chasqui" = clone 4 repos, hand-write 3 `.env`s, share one
  secret between two of them, know what a BSUID is. After this sprint it's
  one command — the omakase promise completed.
- The enabling principle was built deliberately across Sprints 4–7:
  **everything the wizard asks is already a `.env` var** (LLM, embeddings,
  dims, storage, send-URL, SMTP, fallback reply, locale). The services are
  the template; the generator stays thin forever.
- End-of-sprint rule applies to the release itself: it doesn't exist until
  the docs tell it (README quickstart, deploy guide, module guide).

## What

### Part A — CLI (new repo `chasqui-stack/cli`, PyPI: `chasqui`)

| Piece | Behavior |
|---|---|
| Skeleton | Python ≥3.11 · `typer` + `questionary` + `httpx` · entry point `chasqui` · pytest · Apache-2.0 · GH Actions (test + trusted publishing to PyPI). The CLI never imports service code (runs in uvx's ephemeral env). |
| Preflight | Check `uv`, `node` (22+ for admin), `psql`/Postgres reachability — report versions, warn-don't-block (provisioning degrades per step). |
| Fetch | Codeload tarballs of core/whatsapp/admin at the stack tag pinned in the CLI release (constant). `--ref <branch/tag>` escape hatch for dev. Output: ONE plain repo `<name>/{core,whatsapp,admin}` + root README + docker-compose — **no submodules, no git history**. |
| Rename | Explicit manifest in Python (never blind sed): `deploy.yml` service/image names, `APP_NAME`, default `POSTGRES_DB`, admin `package.json` name, README titles → project slug. |
| Wizard | Each prompt ↔ existing `.env` var (the `.env.example`s are the contract): ① LLM provider+model+API key (google/anthropic/openai/openrouter/ollama) ② embeddings provider/model + `EMBEDDING_DIM` with the provision-time explainer (768 default; 3072 → halfvec, ADR-001) ③ **"Where is your Postgres?"** — local / docker-compose / managed URL → `POSTGRES_*` (never "which engine", ADR-002) ④ WhatsApp credentials (token, app id/secret, phone id — **skippable, fill later**; note: `WA_PHONE_ID` required for the inbox) ⑤ default language → `VITE_DEFAULT_LOCALE` + `FALLBACK_REPLY` written in that language (one question, two files; English-only codebase untouched) ⑥ initial admin (email/password) ⑦ extras step: media bucket (`STORAGE_*`), handoff notifications (`NOTIFY_WEBHOOK_URL` / `SMTP_*`) ⑧ optional deploy params (domain, registry user, server IP) → fill the `deploy.yml` placeholders. |
| Auto-secrets | Generated, never asked: `JWT_SECRET_KEY`, `INTERNAL_API_KEY` (**same value into core + gateway `.env`**), `WA_VERIFY_TOKEN` (printed — the user pastes it into the Meta console). Derived: `CHANNEL_WHATSAPP_SEND_URL=http://localhost:8000/send`, `CORE_URL`, `VITE_API_BASE_URL`, `CORS_ORIGINS`. |
| Provision | In order: write `.env`s → `uv sync` (core, whatsapp) + `npm install` (admin) → `createdb` (local choice only) → `alembic upgrade head` (**after** `.env` — consumes `EMBEDDING_DIM`) → seed admin via `uv run python scripts/create_admin.py --email … --password …`. Each step best-effort: on failure print the exact manual command, continue where safe. |
| Epilogue | `git init` + first commit, then print: the three `make dev`/`npm run dev` commands, panel URL + admin credentials reminder, the ngrok/webhook flow for WhatsApp. |
| Flags | `--defaults` (non-interactive, placeholder creds, CI-friendly) · `--skip-provision` (write-only) · `--ref`. |
| `generate module <name>` | Run inside a project (detects `core/app/modules/`): package with `module` attr + `@tool` stub whose docstring is the how-to guide; `--with-models` (SQLModel table + Alembic migration stub via `register_models()`); `--with-admin` (`register_admin_routes()` CRUD stub); `tests/modules/test_<name>.py` on the ScriptedModel pattern. The admin form comes free later via `config_schema()`. |
| Tests | Pure-function core: wizard answers → rendered `.env` contents (golden files), rename manifest application, module-generator output (golden files). Fetch stubbed — no network in tests. |

### Part B — Deploy (per service)

- **De-template `whatsapp/config/deploy.yml`** — today it still says
  `psicolab-whatsapp` + a real registry user + a **real server IP**, and
  references Flow IDs. **Must be sanitized before the repo goes public.**
- **De-template `admin/config/deploy.yml`** — says `saas-template-admin` +
  the same real IP. Both adopt core's placeholder style (`your-username`,
  `203.0.113.10`, `*.example.com`).
- `.kamal/secrets.example` per service (real `.kamal/secrets` stays
  gitignored).
- Verify the three Dockerfiles build for prod (`docker build` each).
- `docs/DEPLOY.md` (parent): the 3-service walkthrough — core + pgvector
  accessory (`pgvector/pgvector:pg17`, ADR-001/002), gateway, admin static;
  hostname convention `api.` / `wsp.` / `admin.<domain>`; auto-TLS via Kamal
  proxy; managed-bucket note for `STORAGE_*` (ADR-003: never a self-hosted
  accessory).

### Part C — OSS polish

- `LICENSE` Apache-2.0 in every repo (core ✓; parent, whatsapp, admin, cli).
- **Parent README rewrite**: what Chasqui is, the `uvx chasqui new`
  quickstart, architecture overview + diagram link, screenshots (panel,
  inbox), local dev, deploy link, badges.
- `CONTRIBUTING.md` (parent) + module-authoring guide (`docs/MODULES.md`):
  how to write a Tool Module — the Sprint-3 contract, `chasqui generate
  module` as the starting point, faq/ as the reference implementation.
- Final pass over service READMEs + `.env.example`s.

### Part D — Release train (order matters)

1. ~~Sanitize deploy.ymls (Part B)~~ — **done 2026-06-11**. ⚠️ Discovery:
   the org repos were ALREADY public, so the leaked IP/registry user sat in
   public history (low risk — no secrets; rewriting history is Willy's
   call, probably not worth it).
2. Final docs pass; submodule bumps in the parent.
3. Tag `v0.1.0`: core, whatsapp, admin, parent.
4. CLI pins the `v0.1.0` stack tag → publish `chasqui` 0.1.0 to PyPI
   (claim the name early — see gotcha 1; trusted publisher setup on PyPI
   is Willy's account) → tag cli `v0.1.0`.
5. ~~Repos public~~ — already public; codeload works as soon as the tag
   exists.
6. e2e with Willy: `uvx chasqui new demo` on a clean setup.

### Success criteria

- [x] On a machine with `uv` + Node 22 + Postgres: `uvx chasqui new demo` →
      answer the wizard → start the three services → message the WhatsApp
      number → the agent replies; panel, handoff inbox and leads work.
      **Without hand-editing a single file.** (Willy's e2e, 2026-06-11:
      wizard clean, WhatsApp reply, FAQ-RAG and handoff confirmed.)
- [x] `chasqui new demo --defaults --skip-provision` is non-interactive
      (CI-able) and `chasqui --version` works via `uvx` (verified from the
      published PyPI package, 2026-06-11).
- [x] `chasqui generate module hello` inside the project → `make test`
      green with the module auto-discovered (verified 2026-06-11; surfaced
      a real bug: test deps were an optional extra `uv sync` never
      installed → moved to `[dependency-groups]` in core+gateway, v0.1.2).
- [ ] A fresh VM deploys the three services following `docs/DEPLOY.md` with
      `kamal deploy` — **deferred post-beta** (needs a real VM; the guide,
      de-templated deploy.ymls, secrets examples and prod Docker builds are
      all in place).
- [x] No `psicolab` / `saas-template` / real-IP strings in anything public.
- [x] `v0.1.0` tagged on all five repos; `chasqui` 0.1.0 on PyPI (trusted
      publishing, 2026-06-11).
- [x] CLI pytest green (19); existing core (124) / gateway (23) / admin
      suites stay green.

---

## All Needed Context

```yaml
- file: docs/design/adr-005-cli-generator.md                  # the decision — read first
- file: docs/sprints/sprint-08-generator-deploy.md            # task list (internal)
- file: core/.env.example                                     # wizard contract (the richest one)
- file: whatsapp/.env.example                                 # wizard contract (WA creds + INTERNAL_API_KEY)
- file: admin/.env.example                                    # wizard contract (VITE_*)
- file: core/scripts/create_admin.py                          # the admin seed the CLI shells out to
- file: core/config/deploy.yml                                # the placeholder style to replicate
- file: core/app/modules/faq/                                 # module anatomy `generate module` scaffolds
- file: docker-compose.yml                                    # collaborator path the README documents
- file: ~/proyectos/pet-projects/saas-maker/generate-project.sh  # superseded source ref (rename list ideas)
```

### Key decisions (on top of ADR-005)

1. **PyPI name `chasqui`** — verified free 2026-06-11 (`chasqui-cli` free
   as fallback). Claim it with the first publish; don't sit on it.
2. **Generated project is ONE plain repo** — submodules are a maintainer
   workflow; users get one history, one clone, one PR flow.
3. **Stack tag pinned per CLI release** (CLI `v0.1.0` ↔ services
   `v0.1.0`) — reproducible scaffolds; `--ref` is the dev escape hatch.
4. **The wizard never asks anything that isn't a `.env` var.** A question
   needing new service behavior means the service grows the variable first.
5. **Renames via explicit Python manifest** — macOS/Linux `sed -i`
   divergence killed the shell approach, and "chasqui" appears in
   identifiers that must not change.
6. **Provisioning never aborts the scaffold** — a failed step prints its
   manual command and continues where safe (`rails new` without a DB still
   leaves a usable project).
7. **WhatsApp credentials are skippable** — people evaluate the stack
   before they have a Meta app; the gateway just won't start until filled.
8. **Trusted publishing (GH Actions → PyPI)** from day one — no API tokens
   to leak.
9. **macOS/Linux only for v0.1** — Windows untested, documented as such.

### Known gotchas

1. **`EMBEDDING_DIM` is consumed at the FIRST `alembic upgrade head`**
   (ADR-001) — the `.env`s must be fully written before provisioning
   migrates. Hard ordering invariant in the CLI.
2. **`INTERNAL_API_KEY` must be byte-identical in core and gateway
   `.env`s** — generate once, write twice.
3. **The gateway/admin `deploy.yml`s currently leak** a previous project's
   name and a real server IP — sanitizing them is a release **blocker**,
   sequenced before the visibility flip.
4. **`create_admin.py` loads core settings** — run it from `core/` via
   `uv run`, after `.env` exists and migrations ran.
5. **Codeload tarballs require public repos** — the CLI can't be e2e-tested
   against the org until the flip; use `--ref` + a local-path override
   during development.
6. **`uvx` runs the CLI in an ephemeral venv** — it must not import
   service code or assume the services' dependencies exist.
7. **`WA_VERIFY_TOKEN` is user-defined at Meta config time** — generate it,
   write it, and PRINT it in the epilogue so the user can paste it into the
   Meta console.
8. **Node 22 needed by the admin** (Vite/React 19) — preflight reports it;
   `npm install` failure degrades gracefully like every provision step.
9. **macOS vs Linux `createdb`/`psql` availability** differs — detect, and
   fall back to printing the SQL (`CREATE DATABASE …`) when absent.
10. **Don't run generated-project verification through `rtk`-style
    wrappers in CI** — assert on files and exit codes, not parsed stdout.

## Tasks

- [x] ADR-005 + this PRP (parent commit).
- [x] New repo `chasqui-stack/cli`: skeleton (pyproject, typer app,
      pytest, CI, LICENSE, README, AGENTS.md + CLAUDE.md symlink).
- [x] Fetch + layout + rename manifest (`--ref`, local-path dev override).
- [x] Wizard + `.env` rendering + auto-secrets (golden-file tests).
- [x] Provision pipeline + epilogue + `--defaults` / `--skip-provision`
      (full-provision e2e against the local stack: all six steps green,
      admin seeded, 2026-06-11).
- [x] `chasqui generate module <name>` (+ `--with-models`, `--with-admin`)
      + golden-file tests.
- [x] Deploy: de-template gateway/admin `deploy.yml`s,
      `.kamal/secrets.example` ×3, Dockerfile prod-build check (all three
      build clean, 2026-06-11), `docs/DEPLOY.md`.
- [x] OSS polish: LICENSEs (already in all repos), parent README rewrite
      (uvx quickstart + badges), CONTRIBUTING (already existed),
      `docs/MODULES.md`, service README pass (internal sprint refs dropped).
- [x] Release train: sanitize → tags `v0.1.0` ×5 → `chasqui` 0.1.0
      **published on PyPI** via trusted publishing (Willy set up the
      publisher; workflow green) → GitHub Releases on cli + parent →
      verified end-to-end: `uvx chasqui new demo` scaffolds from PyPI +
      the real tags (2026-06-11).
- [x] Manual e2e with Willy: `uvx chasqui new demo` from scratch →
      WhatsApp message answered → panel/FAQ/handoff OK (**accepted
      2026-06-11**).

## Mid-beta additions (Willy's feedback, 2026-06-11)

- **v0.1.1** — wizard asks the three service ports (defaults 8090/8000/5191;
  several stacks coexist — core/gateway `make dev` honor `PORT`, admin
  honors `VITE_PORT`); `docs/WHATSAPP-SETUP.md` (Meta credentials
  step-by-step) linked from the wizard, epilogue and READMEs; admin swapped
  to `@vitejs/plugin-react` (Rolldown-Vite warning).
- **v0.1.2** — test deps moved from an optional extra to
  `[dependency-groups]` in core+gateway: plain `uv sync` (what `chasqui
  new` runs) now installs them, so `make test` works in generated projects.
- **CLI 0.1.3** — initial commit moved AFTER provisioning: npm/uv lockfile
  updates land in it, generated projects start with a clean tree.
- **Release ceremony documented** in the cli repo's AGENTS.md (+ pointer in
  the parent's): services tag first → CLI pins `STACK_TAG` + bumps version
  in two places → tag push = trusted-publishing → verify via `uvx`.
