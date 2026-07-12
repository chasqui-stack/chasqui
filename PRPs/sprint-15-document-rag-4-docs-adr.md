# PRP: Sprint 15.4 — Document-RAG: ADR, docs & release readiness

> **Version:** 1.0
> **Created:** 2026-07-12
> **Status:** Ready
> **Executor:** Bryan (@BryanDev2023) with close review, or Willy · **Reviewer:** Willy (@willywg)
> **Series:** 4/4 — requires 15.1–15.3 merged. A sprint isn't closed until docs reflect it.

---

## Goal

The paper trail for Document-RAG: **ADR-012** recording the decisions taken in
15.1/15.3, the affected docs updated (parent ARCHITECTURE/MODULES, core README/AGENTS,
admin README/AGENTS), and a release-notes draft. Explicitly verified: **no CLI or
skills changes are needed** (and why).

**Non-goals:** the release itself (tagging/publishing is Willy's ceremony, documented
in the cli repo's AGENTS.md), website/landing updates.

## Why

- Chasqui is docs-as-code, no wiki: any non-obvious architectural decision gets an
  ADR **in the same repo, versioned** (see `docs/design/adr-*.md`). Document-RAG made
  several (text-in-Postgres, BackgroundTasks over a worker, chunking constants,
  two-retriever separation) that will look arbitrary in six months without the record.
- End-of-sprint rule (parent AGENTS.md): a sprint isn't closed until docs reflect it.

## What

### Success Criteria

- [ ] `docs/design/adr-012-document-rag.md` merged, matching the house ADR shape
      (Status/Sprint/Related header, Context, Decision, Consequences — honest
      negatives included, Alternatives considered, Follow-ups).
- [ ] `docs/ARCHITECTURE.md` §8 mentions the `knowledge` module next to `faq` as a
      built-in; module list/table (if present) updated.
- [ ] `docs/MODULES.md` gains a line pointing at `knowledge` as the second reference
      implementation (upload + background processing patterns).
- [ ] `core/README.md` + `core/AGENTS.md` mention the module (feature list + module
      inventory).
- [ ] `admin/README.md` + `admin/AGENTS.md` mention the Knowledge Base page.
- [ ] Release-notes draft (a markdown block in the epic issue) listing the feature
      for the next stack version.
- [ ] One paragraph in the epic confirming the no-CLI-change rationale (below),
      so the release ceremony has zero surprises.

---

## All Needed Context

```yaml
- skill: chasqui-write-adr                     # the ADR habit + anti-patterns (installed skill)
- file: docs/design/adr-003-media-storage.md   # a good medium-size ADR to mirror
- file: docs/design/adr-008-deferred-dispatch-coalescing.md  # worker-related sibling — Related: link it
- file: docs/design/adr-001-embeddings-provider-dims.md      # Related: dims strategy the module obeys
- file: PRPs/sprint-15-document-rag-1-core-module.md  # decisions 1–7 → ADR Decision section
- file: PRPs/sprint-15-document-rag-3-agent-tool.md   # decisions 1–4 → ADR (two-retriever separation)
- file: chasqui/AGENTS.md                      # docs-as-code conventions, end-of-sprint rule
```

### ADR-012 content map (write it from these, in your own words)

- **Context:** FAQ-RAG answers curated Q&A; businesses hold knowledge in files.
  Post-MVP backlog item since Sprint 4. Constraint: must work on every install
  (no hard S3 dependency), Jr-friendly review surface.
- **Decision:** built-in `knowledge` module; extracted text persisted in Postgres
  (original file discarded, MVP); pdf/docx/txt/md/html via light deps (pypdf,
  python-docx, bs4); RecursiveCharacterTextSplitter 1200/180 overlap (markdown-aware
  for .md); one batched embed per document; FastAPI BackgroundTasks + status column
  (no generic job queue exists — ADR-008's worker is coalescing-specific); separate
  `document_chunks` table + `search_documents` tool with boundary-carrying
  docstrings vs `faq_search` (config_key `document_search`, defaults 4 / 0.35).
- **Consequences — negatives to be honest about:** originals unrecoverable (only
  extracted text); a crash mid-processing leaves `processing` until manual
  reprocess; no OCR (scanned PDFs rejected); two embeddings spaces (FAQ + chunks)
  double the per-query embed cost when both tools fire; description-based routing
  is probabilistic, not guaranteed.
- **Alternatives considered:** originals in the Sprint-6 bucket (deferred — extra
  moving part, storage not always configured); `unstructured` library (heavyweight
  dep tree); generic SKIP-LOCKED job worker (premature — one caller); merging
  FAQ + docs into one retriever (kills curation precision; harder thresholds).
- **Follow-ups (out of scope):** OCR, xlsx/pptx, original-file storage + download,
  worker-based processing, hybrid/keyword search, citations with page/section,
  per-document enable flags.

### Why NO CLI / skills changes (verify, then write it down in the epic)

```text
- Modules under core/app/modules/ are auto-discovered (registry.discover()) — a
  generated project gets `knowledge` by upgrading its stack tag; the wizard asks
  nothing new (no new REQUIRED .env vars: chunking is constants, retrieval knobs
  live in agent_config.tool_config).
- New python deps live in core/pyproject.toml → `uv sync` covers them.
- Skills repo re-pins docs URLs at release time as part of the ceremony (lockstep,
  ADR-009) — not a sprint task.
VERIFY by grepping the cli repo for module-specific wiring (there should be none)
and running `uvx chasqui new` against a local --source checkout if in doubt.
```

---

## Implementation Blueprint

```yaml
Task 1 - ADR:
  - CREATE docs/design/adr-012-document-rag.md (content map above; Status:
    Accepted — <merge date>; Sprint: 15; Related: ADR-001, ADR-003, ADR-008,
    ARCHITECTURE §8; Refs the epic issue #N)
Task 2 - Parent docs:
  - MODIFY docs/ARCHITECTURE.md §8 (knowledge module alongside faq)
  - MODIFY docs/MODULES.md (second reference implementation pointer)
Task 3 - Service docs:
  - MODIFY core/README.md + core/AGENTS.md (feature + module inventory)
  - MODIFY admin/README.md + admin/AGENTS.md (Knowledge Base page)
  NOTE: core/admin changes go as small PRs on those repos (or ride the 15.1/15.2
  PRs if still open); parent changes are one PR on chasqui-stack/chasqui.
Task 4 - Release notes draft + CLI rationale:
  - COMMENT on the epic: release-notes block + the no-CLI-change paragraph
Task 5 - Close the loop:
  - Epic checklist all green → Willy runs the release ceremony (out of scope here)
```

## Validation Loop

```bash
# Docs don't compile — review IS the validation:
# - ADR passes the chasqui-write-adr anti-pattern check (has Alternatives AND
#   negative consequences; ships with/right after the code, not weeks later)
# - grep the repos: every doc that lists modules/pages now includes knowledge
rg -l "faq" docs/ core/README.md core/AGENTS.md   # each hit: does it also need knowledge?
```

## Final Checklist

- [ ] ADR-012 merged (parent repo), linked from the epic
- [ ] ARCHITECTURE §8 + MODULES.md + core & admin READMEs/AGENTS updated
- [ ] Release-notes draft + no-CLI-change rationale posted on the epic
- [ ] PRs reference the epic (`Refs chasqui-stack/chasqui#N`)

## Anti-Patterns to Avoid

- ❌ An ADR that's a brochure — no alternatives, no negative consequences
- ❌ Writing the ADR weeks after the code merged (it travels with the sprint)
- ❌ Documenting in a wiki/gist instead of versioned docs/
- ❌ Touching CLI/skills "just in case" — the rationale above says why not; verify, don't churn
