# CLAUDE.md

Context for Claude Code working on this repo. Read this before touching any stage.

## What this repo is

A structured-search harness that produces schema-validated, cited reports on demand, feeding
a persistent LLM-wiki (raw/wiki/schema three-layer pattern) instead of re-deriving answers
per query. Full staged plan: `docs/ai_harness_wiki_plan.md` — read it before starting or
resuming any stage; this file only covers conventions, not the architecture.

Stages, in dependency order: 0 schema/scope → 1 single-topic search agent → 2 citation
verification → 3 perspective/parallel search → 4 wiki store → 5 query/synthesis interface →
6 maintenance/lint loop → 7 packaging. Don't start a stage whose dependencies in the plan
aren't done — each one assumes the prior stage's output format is stable.

## Current stage

**Stage 0 — schema & scope.** Nothing implemented yet. First tasks: draft the JSON Schema(s)
for report types in `schema/`, and the wiki page-naming/cross-reference conventions that will
become the wiki's own schema file later (Stage 4, not this file).

Update this section as stages complete — one line, current stage only, don't keep a full history here (that belongs in git log / `docs/log.md` once Stage 4 exists).

## Stack

Prototype Stages 0–6 in Python: fastest iteration, best library coverage for schema
validation (Pydantic) and search/agent tooling. Stage 7 explicitly leaves open whether the
orchestration layer gets rewritten in Rust for the hot path (search/merge loop) — don't
pre-optimize into Rust before Stage 6 is working and evaluated. If you want Rust practice
sooner, the merge/dedup logic in Stage 4 is a reasonable place to prototype a Rust module
against the Python harness rather than rewriting the whole pipeline early.

## Repo layout

```
schema/       Stage 0 output — JSON Schemas per report/page type
harness/      Stages 1-3 — search agents, perspective coordinator, verification pipeline
wiki/         Stage 4+ output — raw/, wiki/, and the wiki's own schema file (not this one)
query/        Stage 5 — index reader, gap detection, synthesis
maintenance/  Stage 6 — lint/contradiction pass, gap queue
docs/         plan, decisions, this file's longer-lived companions
tests/        schema validation tests, eval sweeps
```

## Code conventions

- Python: type hints throughout, Pydantic models mirror `schema/` exactly — schema drift
  between the JSON Schema and the Pydantic model is a bug, not a style issue.
- Rust (once introduced): `rustfmt` + `clippy` clean, no `unwrap()` outside tests.
- Every agent role (writer, extractor, judge, coordinator) gets its own prompt file under
  `harness/prompts/`, not inlined in code. Keep roles' prompts separable — Stage 2's judge
  must never share a prompt with Stage 1's writer.

## Agent prompt conventions (apply per-stage, not globally — see plan for which stage needs which)

- **No redundant self-verification.** Writer/merger roles (Stage 1 writer, Stage 4 page
  merger) do not get "double-check your work" instructions — the model self-verifies
  already; that instruction only adds cost here. Only Stage 2's judge role does real
  verification, because it checks against retrieved external evidence, not its own prior output.
- **Scope constraint on writer roles.** Stage 1/3 writer prompts must include the
  deliver-what-was-asked pattern — flag a better approach in one sentence if one exists,
  don't silently widen the topic.
- **Subagent caps are explicit and, once past prototype, enforced in code.** Stage 3's
  perspective coordinator and Stage 2's extractor/judge pipeline are the two legitimate
  delegation points. Every other role should not delegate. Caps live in the prompt during
  prototyping and get moved into code before Stage 7.
- **Narration differs by stage, don't reuse one instruction everywhere.** Stage 1/3 (live,
  user-visible) get the outcome-first narration pattern. Stage 4/6 (background) stay silent
  except for structured log entries — no conversational narration at all.
- **Deliverable length calibration on anything written to the wiki or returned as a report.**
  No filler sections, no restating the schema back to the user. This applies to Stage 1
  reports and Stage 4 wiki pages specifically, not to Stage 5's conversational answers,
  which get their own concise-response instruction instead.
- **Effort defaults per role, not global.** Start every new role at `high` until you have
  evals; expect Stage 5 queries and Stage 2 extraction to hold quality at `low`/`medium`,
  and Stage 3 perspective generation / Stage 1 synthesis to need `high`/`xhigh`. Re-sweep
  before assuming a default.

## How to pick up work

1. Check "Current stage" above and the plan file's dependency graph.
2. Confirm that stage's pre-implementation requirements (listed in the plan) are actually
   satisfied — don't infer from code existing that a requirement is met.
3. Build against `schema/` as the source of truth; if a stage needs a schema change, that's
   a Stage 0 edit first, then propagate.
4. Update "Current stage" and, once it exists, `docs/log.md`.

## Out of scope for now

Don't build Stage 5/6/7 scaffolding while Stage 0-2 are unstable — the plan is sequential on
purpose because each stage's output format is a hard dependency for the next. Resist adding
a vector-search layer before Stage 5 actually needs one; the plan defaults to direct
index-loading at small/medium wiki scale.
