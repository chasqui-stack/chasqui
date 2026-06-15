# ADR-009 — Extension skills (`chasqui-stack/skills`)

> **Status:** Accepted — 2026-06-15
> **Sprint:** 11 (extension skills) — epic chasqui#18
> **Related:** ADR-002 (Postgres-only / omakase, "the menu has an owner"), ADR-004 (`POST /send` outbound seam), ADR-005 (CLI generator, `STACK_TAG` pinning), ARCHITECTURE §5 (canonical contract). Models: [`langchain-ai/langchain-skills`](https://github.com/langchain-ai/langchain-skills), the [Agent Skills](https://agentskills.io) open format.

## Context

Chasqui is omakase, the Rails way: an opinionated stack whose conventions
(the canonical contract, `app/modules/`, the `/send` seam, the ADR habit)
only pay off if they're *followed* — including by other devs extending the
stack for their own clients. Today those conventions live in `docs/` and are
discoverable only by a human who reads them.

The [Agent Skills](https://agentskills.io) format — an Anthropic-origin open
standard now adopted by ~40 agents (Claude Code, Cursor, Codex, Gemini CLI,
Copilot, …) — packages procedural knowledge into version-controlled folders an
agent loads **on demand**. A skill is a `SKILL.md` (frontmatter `name` +
`description`, then instructions) plus optional `scripts/`, `references/`,
`assets/`. Agents use **progressive disclosure**: at startup they see only each
skill's `name`/`description`; the full body loads when a task matches; referenced
files/scripts load only at execution. That makes a skill the natural carrier for
"how you extend Chasqui correctly" — transmitted to the user's *agent*, not just
their eyes.

The risk, by our own doctrine ("wikis drift; `docs/` is versioned and reviewed
with the code"): a skill that **copies** the contract becomes a second source of
truth that rots. Progressive disclosure resolves this — a skill can be a thin
*pointer* to the canonical doc rather than a copy.

## Decision

### 1. A dedicated org repo `chasqui-stack/skills` — sibling, not submodule

Skills ship from a new repo, a structural sibling of `cli` and `website`, and
are **not** a submodule of this parent. The parent orchestrates the **deployable
stack** (core/admin/whatsapp/telegram) — the things `uvx chasqui new` fetches
pinned per tag (ADR-005). Skills are a **distribution/extension** concern, like
the CLI (the generator) and the website (landing) — none of those belong in the
submodule set. Making skills a submodule would entangle authoring commits with
runtime pointer bumps for no benefit.

Repo layout mirrors `langchain-ai/langchain-skills`:

```
chasqui-stack/skills/
├── .claude-plugin/
│   ├── marketplace.json
│   └── plugin.json          # "skills": "./config/skills/"
└── config/skills/<name>/SKILL.md
```

Installable via `npx skills add chasqui-stack/skills --skill '*'` (any
Agent-Skills client) **and** as a Claude Code plugin
(`/plugin marketplace add chasqui-stack/skills`).

### 2. A skill is a thin pointer to the canonical doc, by versioned URL — *Decision A*

A `SKILL.md` carries: the `description` (when the skill applies), the **minimal
procedural steps**, and a **pointer to the canonical doc** — never a copy of it.
The pointer is a `STACK_TAG`-pinned raw URL:

```
https://raw.githubusercontent.com/chasqui-stack/chasqui/<STACK_TAG>/docs/ARCHITECTURE.md
```

The agent fetches it on demand (progressive disclosure, execution stage). This
keeps `docs/` the single source of truth (docs-as-code) and shrinks the
driftable surface to near zero.

**Why a URL and not a relative path:** generated projects do **not** ship
`docs/` (the CLI only contributes `docker-compose.yml` as a parent root file,
ADR-005), so `docs/ARCHITECTURE.md` does not exist in a scaffolded project. A
pinned raw URL always resolves and is version-matched to the stack the skill
targets.

**Rejected pointer alternatives:**
- **`references/` vendored into the skill** — works offline but reintroduces a
  copy to maintain; the drift we explicitly avoid.
- **CLI copies `docs/` into the scaffold** — bloats every generated project with
  docs it doesn't run, and still needs lockstep updates.

### 3. Versioned in lockstep with `STACK_TAG`

The skills repo is tagged in step with the stack and the CLI, and bumping its
pinned doc URLs becomes part of the release ceremony (cli/AGENTS.md). A skill at
the stack's `vX.Y.Z` points at `docs/` at `vX.Y.Z`. This is also the priority
rationale for doing it early: establishing the lockstep machinery now is cheaper
than retrofitting it across many skills once the roadmap has grown.

### 4. Bidirectional CLI ↔ skills integration

- After `uvx chasqui new`, the CLI prints a next-step hint:
  `npx skills add chasqui-stack/skills --skill '*'` — the generator hands the
  dev the skills that teach their agent to extend the freshly-scaffolded stack.
  The CLI is "zero to running"; skills are "extend it correctly afterward".
- One skill (`chasqui-cli`) **is** the CLI reference for the agent (install,
  `chasqui new`, the wizard, `--ref`) — the same pattern as
  `langchain-skills`' `langgraph-cli`.

### 5. Taxonomy — a primer/router plus focused skills

Modeled on `langchain-skills`' `ecosystem-primer`:

1. `chasqui-primer` — router, **INVOKE FIRST**: omakase philosophy, the three
   services, the canonical contract, which skill to load next.
2. `chasqui-cli` — CLI reference.
3. **`chasqui-create-channel`** — a new gateway over the canonical contract +
   the `/send` seam (ADR-004). Highest leverage and the best showcase of the
   channel-agnostic design; Telegram (sprint 9) is the worked example.
4. `chasqui-create-module` — a business module in `core/app/modules`.
5. `chasqui-add-tool` — the tool registry.
6. `chasqui-write-adr` — the ADR / docs-as-code convention.
7. `chasqui-deploy` — Kamal.

First sprint builds **1 + 2 + 3** to validate the pattern before escalating.

## Consequences

**Positive**
- Conventions become executable and portable to *any* Agent-Skills client, not
  just Claude Code — a real adoption multiplier for an opinionated framework.
- `docs/` stays the single source of truth; skills add no second copy to
  maintain (pointer, not copy).
- Clean separation of concerns: parent = deployable stack, `skills`/`cli`/
  `website` = distribution.

**Negative / trade-offs**
- Skill activation that needs the canonical doc requires **network** (to fetch
  the pinned raw URL). Acceptable: it's the execution stage, not discovery.
- The skills repo is **one more release surface** to bump in lockstep — mitigated
  by folding it into the existing ceremony.
- A skill's *procedural prose* (not the pointed-to doc) can still age; kept small
  and reviewed with each release to bound the risk.

## Alternatives considered

- **Skills co-located inside each service repo** — rejected: not installable as
  one unit via `npx skills add <repo>`, no single marketplace/leaderboard
  presence, and harder to version as a coherent set.
- **Skills as a submodule of the parent** — rejected: skills are distribution,
  not the deployable stack (§1); would couple authoring to runtime pointer bumps.
- **Skills that inline the contract** (the `langchain-skills` content style) —
  rejected for Chasqui: we already have versioned `docs/`; pointing to it beats
  duplicating it (§2).

## Follow-ups (out of scope)

- The remaining taxonomy skills (4–7) after the first trio proves out.
- A **website** Skills section (chasqui.chat) with the install command + the
  skills.sh install-count badge.
- Telemetry/opt-out guidance for the skills.sh leaderboard, if we publish there.
