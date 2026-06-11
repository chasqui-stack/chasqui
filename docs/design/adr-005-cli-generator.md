# ADR-005 — The project generator is a thin CLI (`uvx chasqui new`)

> **Status:** Proposed — 2026-06-11
> **Sprint:** 8 (CLI, deploy & OSS release)
> **Related:** ADR-001 (EMBEDDING_DIM is provision-time), ADR-002 (omakase
> posture), `docs/sprints/sprint-08-generator-deploy.md` (the pivot)

## Context

The product is complete (Sprints 0–7), but *using* it requires cloning four
repos, hand-writing three `.env` files, sharing one secret between two of
them, and knowing what a BSUID is. The omakase promise is a `rails new`
equivalent: **one command from zero to a configured, running WhatsApp agent.**

A `generate-project.sh` existed as a source reference (saas-maker). Shell
generators don't survive contact with users: no real prompts, `sed -i`
divergence between macOS and Linux, no testability, no `--defaults` for CI.

The enabling design principle was laid down across Sprints 4–7, on purpose:
**everything a generator would need to ask is already a `.env` variable** in
the services — LLM provider/model, embeddings + `EMBEDDING_DIM`, Postgres
location, WhatsApp credentials, storage, send-URL, notifications, fallback
reply, admin-UI locale. The services are the template; no engine needed.

## Decision

### 1. A Python CLI in its own repo, published to PyPI as `chasqui`

New repo `chasqui-stack/cli`, package name `chasqui` (verified free on PyPI,
2026-06-11; `chasqui-cli` also free as fallback). Runnable with zero install
via `uvx chasqui new <name>` — `uv` is already a stack prerequisite, so the
runner comes for free. Stack: `typer` (commands) + `questionary` (wizard) +
`httpx` (fetch). The CLI never imports service code — it runs in uvx's
ephemeral environment, before any service dependency exists.

### 2. The wizard asks only what is a `.env` variable

Every prompt maps 1:1 to a variable that already exists in a service
`.env.example`. If a question would require new behavior in a service, the
variable lands in the service **first** (own PR, own sprint if needed) — the
generator stays thin forever: **download + rename + write `.env`s +
provision.** No template engine, no conditional scaffolding, no engines.

### 3. Degit-style fetch at a pinned stack tag

`chasqui new` downloads GitHub tarballs (codeload) of the three service repos
at a **stack tag pinned per CLI release** (CLI `v0.1.0` ↔ services
`v0.1.0`) — no git history, no submodules, reproducible output. An escape
hatch (`--ref`) exists for development and testing against unreleased
branches.

### 4. The generated project is ONE plain repo

`<name>/` contains `core/`, `whatsapp/`, `admin/` as plain directories plus a
root README and docker-compose, then `git init` + first commit. Submodules
are a *maintainer* workflow (pinning service versions in the parent); a user
project wants one history, one clone, one PR flow.

### 5. Renames via an explicit manifest, in Python

A fixed list of (file, replacement) pairs — service/image names in
`deploy.yml`, `APP_NAME`, default `POSTGRES_DB`, `package.json` name, README
titles. Never a blind `sed` over the tree (portability, and "chasqui"
appears in code identifiers that must not change).

### 6. Provisioning is best-effort and resumable

`uv sync` (core, gateway) + `npm install` (admin) + `createdb` (local
Postgres only) + `alembic upgrade head` + admin seed
(`scripts/create_admin.py`). Each step that fails prints the exact manual
command and the run **continues where safe** — `rails new` on a machine
without a database still leaves a usable project. Hard ordering invariant:
`.env`s are written **before** the first migrate, because `EMBEDDING_DIM` is
baked into the schema at that moment (ADR-001).

## Consequences

- One command (`uvx chasqui new`) is the public face of the stack — the
  parent README quickstart, the release announcement, the docs all point at
  it. The CLI repo becomes release-critical: services tag first, CLI pins
  the tag, then publishes.
- Service `.env.example` files are now a **public contract** consumed by the
  wizard — adding a provisioning-relevant variable means updating the CLI's
  prompt map in the same release.
- `chasqui generate module <name>` rides the same repo and philosophy: it
  scaffolds the Sprint-3 module anatomy (the contract is the template), and
  the admin form still comes free via `config_schema()`.
- macOS/Linux only for v0.1; Windows is untested and documented as such.

## Considered and rejected

- **Shell script (`generate-project.sh`)** — not cross-platform, not
  testable, no real wizard; superseded by this ADR.
- **cookiecutter/copier template repo** — a template is a *second copy* of
  the stack that drifts from the real repos. Here the repos ARE the
  template; the wizard only writes `.env`s.
- **Submodules in the generated project** — exports our maintainer workflow
  onto users; rejected (decision 4).
- **`npx create-chasqui`** — the stack is Python-first and `uv` is already
  required; a Node entry point adds a second toolchain for zero gain.
- **Cloning with full git history** — slower, bigger, and the user's project
  history should start at *their* first commit.
