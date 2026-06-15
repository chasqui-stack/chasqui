# PRP: Sprint 11 — Chasqui extension skills (`chasqui-stack/skills`)

> **Epic:** chasqui#18 · **Decision:** ADR-009 (`docs/design/adr-009-extension-skills.md`)
> **Scope of this PRP:** stand up the new repo + the **first trio** of skills
> (`chasqui-primer`, `chasqui-cli`, `chasqui-create-channel`) + the bidirectional
> CLI hook. Skills 4–7 (module/tool/adr/deploy) are follow-ups, not this sprint.

## Goal

Ship a new org repo **`chasqui-stack/skills`** (local sibling `chasqui-skills/`,
alongside `chasqui-cli/` and `chasqui-website/`) packaging
[Agent Skills](https://agentskills.io) that teach *any* skills-compatible coding
agent (Claude Code, Cursor, Codex, Gemini CLI, …) how to work with Chasqui. End
state: `npx skills add chasqui-stack/skills --skill '*'` installs three working
skills, each a **thin pointer** to Chasqui's canonical docs by `STACK_TAG`-pinned
URL, and `uvx chasqui new` tells the dev to install them.

## Why

- **Adoption play.** Conventions (the canonical contract, the `/send` seam, the
  ADR habit) only pay off if followed by *other* devs building on Chasqui for
  their clients. Skills transmit them to the dev's **agent**, not just their eyes.
- **One standard, every agent.** Agent Skills is an Anthropic-origin open format
  adopted by ~40 clients — one authoring effort reaches all of them.
- **Cheap now, expensive later.** Establishing the lockstep-with-`STACK_TAG`
  machinery while there are 3 skills beats retrofitting it across many.

## What

A standalone repo whose layout mirrors
[`langchain-ai/langchain-skills`](https://github.com/langchain-ai/langchain-skills)
(cloned for reference at `~/projects/open-source/langchain-skills`):

```
chasqui-skills/                         # remote: chasqui-stack/skills
├── .claude-plugin/
│   ├── marketplace.json                # one marketplace, one plugin
│   └── plugin.json                     # "skills": "./config/skills/"
├── config/
│   └── skills/
│       ├── chasqui-primer/SKILL.md     # router — INVOKE FIRST
│       ├── chasqui-cli/SKILL.md        # CLI reference
│       └── chasqui-create-channel/SKILL.md
├── README.md                           # install (npx skills + Claude plugin)
└── LICENSE                             # Apache-2.0 (match the stack)
```

Plus, in `chasqui-cli`: a post-scaffold next-step line suggesting
`npx skills add chasqui-stack/skills --skill '*'`.

### Success Criteria

- [ ] `chasqui-stack/skills` repo exists, public, Apache-2.0, with the layout above.
- [ ] `npx skills add chasqui-stack/skills --skill '*' --yes` installs all three
      skills into a throwaway project (and `--global` works).
- [ ] `/plugin marketplace add chasqui-stack/skills` + `/plugin install` works in
      Claude Code (manifests valid).
- [ ] Each `SKILL.md` has valid frontmatter (`name`, `description`) and a
      `STACK_TAG`-pinned raw-URL pointer that **resolves** (HTTP 200).
- [ ] `chasqui-primer` routes to the other two by name (the router pattern).
- [ ] `chasqui new` prints the `npx skills add …` hint after a successful scaffold.
- [ ] Release ceremony (cli/AGENTS.md) documents bumping skills in lockstep.
- [ ] No canonical contract text is **copied** into a skill — only pointed to.

## All Needed Context

### Documentation & References

```yaml
- file: docs/design/adr-009-extension-skills.md
  why: THE decision. Repo-not-submodule, Decision A (pinned-URL pointer, no copy),
       lockstep with STACK_TAG, taxonomy, CLI↔skills integration. Implement to this.

- url: https://agentskills.io  (+ /specification, /skill-creation/quickstart)
  why: SKILL.md format, frontmatter (name+description required), progressive
       disclosure (discovery → activation → execution). The skill is description +
       minimal steps + on-demand pointer — NOT a full doc copy.

- repo: ~/projects/open-source/langchain-skills  (cloned)
  why: The structural model. Copy the SHAPE: .claude-plugin/{marketplace,plugin}.json,
       config/skills/<name>/SKILL.md, README install section, install.sh (optional).
       See config/skills/ecosystem-primer/SKILL.md (router) and
       config/skills/langgraph-cli/SKILL.md (CLI-as-skill) as direct templates.

- file: docs/ARCHITECTURE.md
  why: The canonical-doc TARGET the skills point to. §5 canonical contract
       (gateway↔core), §5.1 POST /send outbound (ADR-004), §8.2 module contract.

- file: docs/design/adr-004-conversation-mode-outbound-send.md
  why: create-channel must wire the /send seam — point the skill here.

- file: docs/design/adr-006-telegram-channel.md  +  docs/TELEGRAM-SETUP.md
  why: Telegram is the WORKED EXAMPLE for create-channel (sprint 9, ~40% copied
       from whatsapp). The skill walks the same path.

- dir: telegram/  (app/main.py, app/handlers, app/services, app/core)
  why: The reference gateway structure a new channel mirrors. Point, don't inline.

- file: ../chasqui-cli/src/chasqui/stack.py
  why: STACK_TAG lives here (currently "v0.2.4"). The skills' pinned URLs use this
       exact tag. cli.py already has `chasqui new` + `chasqui generate module`.

- file: ../chasqui-cli/src/chasqui/cli.py  (around line 116, "Next steps:")
  why: Where to add the post-scaffold `npx skills add` hint (the `new` command).
```

### The pointer pattern (Decision A — the heart of every skill)

A `SKILL.md` body NEVER pastes the contract. It says, e.g.:

```markdown
The canonical message contract is the single source of truth. Fetch it on demand:
→ https://raw.githubusercontent.com/chasqui-stack/chasqui/v0.2.4/docs/ARCHITECTURE.md  (§5)
Do not reproduce its fields here; read it when you need them.
```

The tag segment (`v0.2.4`) is what the release ceremony bumps. The agent fetches
it during the *execution* stage (progressive disclosure), so `docs/` stays the
one source of truth and the skill carries almost no driftable surface.

### Known gotchas & conventions

```text
# 1. POINTER, NOT COPY. The whole ADR-009 thesis. If you find yourself pasting
#    contract fields / endpoint schemas into a SKILL.md, stop — link the pinned
#    URL instead. The only prose in a skill is the PROCEDURE + when-to-use.

# 2. Generated projects do NOT ship docs/ (CLI contributes only docker-compose.yml,
#    ADR-005). So pointers MUST be absolute pinned raw URLs, never relative paths
#    like docs/ARCHITECTURE.md — that file isn't in the dev's scaffold.

# 3. English-only (CLAUDE.md). SKILL.md content, descriptions, and prompt-facing
#    strings are English. (The stack localizes via the DB system prompt.)

# 4. STACK_TAG is the version anchor. Every pinned URL uses the SAME tag the skills
#    release targets. First cut pins v0.2.4 (current). Hardcode it; the ceremony
#    bumps it — do NOT use `main` (would silently drift / break on contract change).

# 5. Frontmatter is minimal: `name` (kebab-case, matches dir) + `description`
#    (when-to-use, this is all the agent sees at discovery — make it trigger well).
#    Mirror langchain-skills descriptions: "INVOKE THIS SKILL when …".

# 6. The primer is a ROUTER. It must (a) state the omakase philosophy + the 3
#    services + the contract pointer, and (b) tell the agent which sibling skill to
#    load next for the task at hand. Model on ecosystem-primer/SKILL.md.

# 7. Repo is a SIBLING, not a submodule of the parent (ADR-009 §1). Do NOT add it
#    to .gitmodules. Create it as its own repo; clone locally next to chasqui-cli.

# 8. License + authorship match the stack: Apache-2.0, author "William Wong Garay
#    <willywg@gmail.com>" (public authorship per memory). Commit co-author trailer
#    matches repo history.
```

## Implementation Blueprint

### Tasks (in execution order)

```text
1. Create the repo.
   gh repo create chasqui-stack/skills --public \
     --description "Agent Skills for building & extending Chasqui stacks"
   Clone locally as ~/proyectos/pet-projects/chasqui-skills (sibling of chasqui-cli).
   Add LICENSE (Apache-2.0) + .gitignore (.DS_Store, node_modules/, .claude).

2. Manifests (.claude-plugin/), copied-and-adapted from langchain-skills:
   - marketplace.json: name "chasqui-skills", owner, one plugin pointing "./".
   - plugin.json: name/version "0.1.0"/description, "skills": "./config/skills/",
     keywords [whatsapp, telegram, ai-agent, langgraph, chasqui, agent-skills].

3. config/skills/chasqui-primer/SKILL.md  (ROUTER — build first)
   frontmatter description: "INVOKE FIRST for any work on a Chasqui stack …
     omakase philosophy, the 3 services (core/admin/channel gateways), the
     canonical contract, and which Chasqui skill to load next."
   body: overview (what Chasqui is, omakase, channel-agnostic), the contract
     pointer (pinned URL §5), a decision table → "creating a channel? load
     chasqui-create-channel. Using the CLI? load chasqui-cli." Keep it thin.

4. config/skills/chasqui-cli/SKILL.md  (model on langgraph-cli/SKILL.md)
   description: "INVOKE when using the chasqui CLI to scaffold or extend a stack:
     uvx chasqui new, the wizard, --ref, chasqui generate module."
   body: install (uvx chasqui new <name>), wizard steps, --ref override (ADR-005),
     `chasqui generate module` (already in cli.py). Pointer to cli repo AGENTS.md
     for the release ceremony. Commands + when-to-use, à la langgraph-cli.

5. config/skills/chasqui-create-channel/SKILL.md  (HIGHEST LEVERAGE)
   description: "INVOKE when adding a NEW channel gateway to a Chasqui stack
     (a new messaging platform). Walks the canonical-contract inbound /ingest +
     outbound /send seam, using the Telegram gateway as the worked example."
   body: the procedure — (a) read the contract (pinned §5 + §5.1) and ADR-004,
     (b) mirror telegram/ structure (stateless FastAPI: handlers → canonical
     /ingest; POST /send → platform API), (c) the core only needs
     CHANNEL_<CH>_SEND_URL wiring — it never learns the channel exists,
     (d) ERROR_REPLY/UNSUPPORTED_REPLY are gateway-local (CLAUDE.md). All as
     pointers to the pinned docs + telegram example, not pasted code.

6. README.md — install matrix (npx skills add chasqui-stack/skills --skill '*'
   [--global]; Claude Code plugin via /plugin marketplace add). skills.sh badge.
   One-paragraph "what these are" + link to ADR-009.

7. CLI hook (chasqui-cli, separate PR on that repo):
   In cli.py `new`, after the existing success output, print:
     "Teach your agent to extend this stack:
        npx skills add chasqui-stack/skills --skill '*'"
   (Behind nothing — always shown; it's a hint, not provisioning.)

8. Release ceremony: in chasqui-cli/AGENTS.md, add the skills repo to the tag
   order — bump skills' pinned URLs + tag chasqui-stack/skills vX.Y.Z in step
   with STACK_TAG. (Docs change; can ride task 7's PR or its own.)
```

## Validation Loop

### Level 1: Structure & manifests

```bash
# valid JSON + required keys
cat .claude-plugin/marketplace.json | python3 -m json.tool >/dev/null
cat .claude-plugin/plugin.json | python3 -m json.tool >/dev/null
# every skill has frontmatter name+description
for f in config/skills/*/SKILL.md; do
  head -5 "$f" | grep -q '^name:' && head -5 "$f" | grep -q '^description:' \
    || echo "MISSING frontmatter: $f"
done
```

### Level 2: Pointer URLs resolve (the Decision-A guarantee)

```bash
# every pinned raw URL in every skill returns 200 (and uses a tag, not main)
grep -rhoE 'https://raw.githubusercontent.com/chasqui-stack/[^ )]+' config/skills \
  | sort -u | while read u; do
    echo "$u" | grep -q '/main/' && echo "BAD (uses main): $u"
    code=$(curl -s -o /dev/null -w '%{http_code}' "$u"); echo "$code  $u"
  done
# expect: all 200, none containing /main/
```

### Level 3: Install + activation smoke test

```bash
# install into a throwaway dir; confirm the three skills land
mkdir -p /tmp/skills-smoke && cd /tmp/skills-smoke
npx skills add chasqui-stack/skills --skill '*' --yes
ls .claude/skills/ 2>/dev/null   # expect chasqui-primer, chasqui-cli, chasqui-create-channel
# Manual: in an agent, ask "add a Discord channel to my Chasqui stack" →
# chasqui-primer should trigger and route to chasqui-create-channel.
```

### Level 4: CLI hint

```bash
cd ../chasqui-cli && uvx --from . chasqui new demo --skip-provision \
  && echo "<look for the 'npx skills add' line in the output>"
```

## Final Checklist

- [ ] Repo created, public, Apache-2.0, sibling (NOT in parent `.gitmodules`).
- [ ] Manifests valid; three skills present with good `description` triggers.
- [ ] Every pointer is a `v0.2.4`-pinned raw URL that returns 200; no `/main/`.
- [ ] No contract/code text copied into a skill (pointer-only).
- [ ] `npx skills add` (local + global) and the Claude plugin path both work.
- [ ] primer routes; cli mirrors langgraph-cli; create-channel uses Telegram as example.
- [ ] `chasqui new` prints the install hint (cli PR).
- [ ] cli/AGENTS.md release ceremony updated for lockstep.
- [ ] README install matrix + ADR-009 link.

## Anti-Patterns to Avoid

- ❌ Don't paste the canonical contract / endpoint schemas into a SKILL.md — point
  to the pinned URL (ADR-009 Decision A). Copies drift; that's the whole reason.
- ❌ Don't pin pointers to `main` — pin to `STACK_TAG`, or a contract change breaks
  installed skills silently.
- ❌ Don't use relative `docs/…` paths — generated projects don't ship `docs/`.
- ❌ Don't add the repo to the parent's `.gitmodules` — it's distribution, not stack.
- ❌ Don't write Spanish in SKILL.md — English-only codebase (CLAUDE.md).
- ❌ Don't make the primer a dumping ground — it's a thin router (philosophy +
  contract pointer + "load skill X next"), not a manual.
- ❌ Don't gate the CLI hint behind a flag — it's a one-line suggestion, always shown.

## Notes

**Why first trio = primer + cli + create-channel.** The primer is the entry point
every other skill needs; the cli is the lowest-risk CLI-as-reference (already
modeled by langgraph-cli); create-channel is the highest-leverage and best
showcase of the channel-agnostic design — and Telegram (sprint 9) is a ready
worked example. Skills 4–7 (module/tool/adr/deploy) follow once the pattern proves
out. Note the CLI already ships `chasqui generate module`, so `chasqui-create-module`
will largely point at that command + `docs/MODULES.md`.

**Lockstep mechanics.** The only per-release work is bumping the tag segment in the
pinned URLs and tagging the skills repo `vX.Y.Z`. Cheap by design — that's why we
do it now. A future `make bump-skill-pins` could sed the tag across `config/skills`.

**Out of scope (future):** skills 4–7; the website Skills section + skills.sh badge
(epic acceptance, separate task on chasqui-website); skills.sh telemetry/opt-out
guidance if we publish to the leaderboard; an `install.sh` like langchain-skills'.

---

**Confidence: 8/10** for one-pass success. High: the structural model is concrete
(langchain-skills cloned), the decisions are settled (ADR-009), the doc anchors
exist. The −2: authoring SKILL.md *descriptions* that trigger reliably is iterative
(needs the Level-3 activation smoke test to tune), and the `npx skills`/plugin
install behavior should be verified against the live CLI rather than assumed.
