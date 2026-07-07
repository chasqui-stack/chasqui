# PRP — `chasqui add channel <name>`: retrofit a channel into an existing project

**Issue:** [cli#5](https://github.com/chasqui-stack/cli/issues/5) · **Repo:** `chasqui-stack/cli` · **Sprint:** 14

## Goal

Projects scaffolded with `chasqui new` are degit snapshots — one plain repo, no
upstream. Users who skipped a channel in the wizard (or scaffolded before a
channel existed, e.g. web/ADR-011) have no command to add one later; today the
path is manual (`docs/WEB-SETUP.md`, "Adding it to an existing project").

`chasqui add channel <whatsapp|telegram|web>`, run from a project root, does
that retrofit: fetch the gateway dir at the project's stack tag, render its
`.env` reusing the **existing** `INTERNAL_API_KEY`, ask only that channel's
wizard questions, wire `CHANNEL_<CH>_SEND_URL` into `core/.env`, provision
best-effort.

**Out of scope:** `chasqui upgrade` (three-way merge against user-edited code).

## Context — everything already exists; this is composition

| Piece | Where | Reused how |
|-------|-------|-----------|
| Channel dir ↔ repo map | `stack.CHANNEL_SERVICES` | validates `<name>`, resolves the repo |
| Degit fetch | `fetch._download_tarball` / `_extract_into` / `_copy_local` | new thin `fetch_channel(project_dir, channel, ref, source)` |
| `.env` renderers | `envfiles.render_{whatsapp,telegram,web}_env(a, s)` | called with `GeneratedSecrets(internal_api_key=<existing>)` — the dataclass accepts overrides, so the shared secret stays **byte-identical** with `core/.env` |
| Channel questions | `wizard._ask_{whatsapp,telegram,web}` + port prompts | new `wizard.run_channel_wizard(channel, answers)`; `--defaults` skips it |
| Provision | `provision.Step` / `provision.run` | one step: `uv sync` (python gateways) or `npm install` (web) |
| Next-steps text | `epilogue.build` channel sections | extracted into per-channel helpers shared by `build` and a new `build_add` |

New module: `src/chasqui/add_channel.py` (detection + orchestration).

## Key decisions

- **Project detection:** `core/.env` present in cwd ⇒ project root. Anything
  else refuses with a pointer to run it from the generated directory.
- **Stack ref:** parsed from the generated `README.md` (`chasqui new` writes
  "…stack vX.Y.Z"); `--ref` overrides; fall back to the CLI's pinned
  `STACK_TAG` with a warning. Fetching the *project's* tag keeps the new
  gateway contemporaneous with the rest of the snapshot.
- **`INTERNAL_API_KEY` is read, never regenerated** — if it's missing from
  `core/.env` the command fails fast with guidance (a wrong key would 401
  silently at runtime; worst kind of bug to hand a user).
- **`CORE_URL` follows the project:** the core port is read from `core/.env`
  (`PORT`, default 8090), not assumed.
- **Idempotence:** refuse politely if `<channel>/` exists; appending
  `CHANNEL_<CH>_SEND_URL` is skipped if the key is already present.
- **No git commit** — unlike `chasqui new` (which births the repo), `add`
  mutates the *user's* repo; changes are left staged-for-review.

## Tasks

1. `fetch.fetch_channel` — one-dir fetch (tarball or `--source` copy).
2. `add_channel.py` — `detect_project`, `detect_stack_ref`, `read_env_value`,
   `wire_core_send_url`, `run_add` orchestration.
3. `wizard.run_channel_wizard` — port + channel-specific prompts only.
4. `epilogue` — extract channel sections; `build_add` (run command, "restart
   the core", webhook/embed steps).
5. `cli.py` — `add` sub-app, `channel` command (`--defaults`,
   `--skip-provision`, `--ref`, `--source`).
6. Tests (`tests/test_add_channel.py`) + `mini_stack` grows `telegram/` and
   `web/` dirs.
7. `AGENTS.md` command inventory.

## Validation

```bash
uv run pytest                      # all green, new suite included
# manual: scaffold with --channels whatsapp, then:
uv run chasqui add channel telegram --defaults --skip-provision --source ~/…/chasqui
```

Acceptance: gateway dir laid at the project's tag · gateway `.env` carries the
core's exact `INTERNAL_API_KEY` · `CHANNEL_<CH>_SEND_URL` appended once ·
second run refuses (`telegram/ already exists`) · run outside a project
refuses (no `core/.env`) · epilogue says to restart the core.

**Confidence:** 9/10 — pure composition of tested pieces; the only novel logic
is detection (README tag, env parsing), both trivially unit-testable.
