# AI Harness + LLM-Wiki: Staged Implementation Plan

Structured-search harness (schema-validated, verified reports) feeding a Karpathy-style
three-layer wiki (raw / wiki / schema). Model: Claude Opus 5. Each stage lists its goal,
what has to exist before you start it, what it depends on, and how the Opus 5 prompting
guide changes that stage's system prompt specifically — not generic advice, only what
applies to that stage's failure mode.

## Dependency graph

```
Stage 0 (schema/scope)
   └─> Stage 1 (single-topic search agent)
          ├─> Stage 2 (citation verification)
          └─> Stage 3 (multi-perspective/parallel search) ──depends on Stage 2 too
                 └─> Stage 4 (wiki: raw/wiki/schema store)
                        ├─> Stage 5 (query/synthesis interface)
                        └─> Stage 6 (maintenance/lint loop) ──feeds new tasks back to Stage 1/3
                               └─> Stage 7 (packaging: CLI/MCP/Rust)
```

---

## Stage 0 — Schema & Scope Definition

**Goal:** Fix, on paper, what a "report" and a "wiki page" contain before any agent code
exists — field names, required vs. optional, confidence/provenance fields, page-type
taxonomy (concept / entity / comparison / source).

**Pre-implementation requirements:**
- A JSON Schema (or Pydantic model) per report type you'll support.
- Naming/cross-reference conventions for wiki pages (this becomes the schema file, e.g. `AGENTS.md`).

**Dependencies:** None — this is the foundation everything else validates against.

**Prompting-guide alignment:** Minimal here — this is mostly human design. If you use
Claude to draft the schema docs, apply the *written deliverable length* instruction
(cover the substance, no filler sections) since schema docs are exactly the kind of thing
Opus 5 will otherwise pad.

---

## Stage 1 — Single-Topic Structured Search Agent (core loop)

**Goal:** Given one topic/definition, run a bounded search loop (query → retrieve → assess
→ refine) and emit one schema-validated report. No multi-agent, no wiki yet — get this
loop reliable first.

**Pre-implementation requirements:**
- Search tool/connector wired up.
- Schema validator from Stage 0 in the loop (reject/repair non-conformant output).
- Base system prompt for the search-and-write role.

**Dependencies:** Stage 0.

**Prompting-guide alignment:**
- **Task scope constraint** — this is where scope creep first shows up: Opus 5 will
  happily "also" cover adjacent topics unasked. Use the exact scope-constraint pattern
  from the guide: deliver what was asked, flag a better approach in one sentence if one
  exists, don't quietly widen the topic.
- **Written deliverable length** — apply the no-padding instruction to the report body itself.
- **Remove legacy verification scaffolding** — do *not* add "double-check your report before
  finishing" to this agent's prompt. Opus 5 self-verifies already; that instruction only
  burns tokens here. (Real verification against external evidence is Stage 2's job, not
  this agent re-reading its own output.)
- **Narration cadence** — set the one-sentence-before-tool-call / lead-with-outcome pattern
  if this agent's steps are shown to the user live.
- **Effort** — start at `high`, sweep down to `medium`/`low` once you have evals; simple
  definitional lookups likely hold quality at lower effort.

---

## Stage 2 — External Citation Verification

**Goal:** A separate pass — extractor → memory lookup → web/scholar retrieval → judge —
that checks Stage 1's citations against live sources. This is *evidence-grounded*
verification, not self-reflection, which is why it doesn't conflict with the "don't
add redundant verification" guidance above.

**Pre-implementation requirements:**
- Distinct prompts per role (extractor, judge) — don't reuse Stage 1's prompt for this.
- A `confidence` / `verified: bool` field in the Stage 0 schema to write results into.

**Dependencies:** Stage 1 (needs a report + citations to check).

**Prompting-guide alignment:**
- **Subagent spawning control** — if extractor/retriever/judge run as separate agent
  calls, this is one of the cases the guide calls a *good* delegation target (genuinely
  independent sub-tasks), but still cap it explicitly (e.g. one extractor + one judge per
  report, not one per citation) or cost scales badly.
- **Effort** — extraction can run at `low`/`medium`; judgment (the actual real/fake call)
  should stay at `high`.

---

## Stage 3 — Perspective-Guided Parallel Search (STORM-style)

**Goal:** Extend Stage 1 so a topic spawns N perspective sub-agents (different angles on
the same question), which get merged into one outline-first report. This is where the
"defined and structured" search you wanted actually comes from — perspectives are
generated *before* writing, not discovered ad hoc.

**Pre-implementation requirements:**
- Perspective-generation prompt (surveys existing coverage, proposes N angles).
- Merge/aggregation logic that reconciles overlapping sub-agent findings before Stage 2 runs.

**Dependencies:** Stage 1, Stage 2 (verification can run per-perspective or once on the merged report — decide before building, it changes cost a lot).

**Prompting-guide alignment:**
- **Subagent spawning control is critical here** — this is exactly the "large,
  genuinely independent, parallelizable" scenario the guide flags as worth delegating.
  Set a hard cap on perspective count (e.g. 3–5) and state the delegation criterion
  explicitly in the coordinator's prompt; otherwise Opus 5's higher delegation propensity
  will spawn more than you want per topic.
- **Narration cadence** — suppress per-sub-agent chatter in user-facing output; only
  surface the coordinator's summary, not each perspective agent's play-by-play.
- **Self-correction narration** — when merging perspectives produces a correction to an
  earlier sub-agent's claim, only surface it if it changes the report's conclusion; silent
  fix otherwise.

---

## Stage 4 — Wiki Layer (raw / wiki / schema)

**Goal:** Persistent store. Verified reports from Stage 3 get compiled into wiki pages —
new page, or merged into an existing page on the same concept — plus `index.md` and
`log.md` updates.

**Pre-implementation requirements:**
- Directory layout: `raw/` (immutable sources), `wiki/` (generated pages), schema file.
- Dedup/merge logic: does a page for this concept already exist? Read-then-revise, not
  read-then-duplicate.
- Version control (git) for provenance and rollback.

**Dependencies:** Stages 1–3 as content sources.

**Prompting-guide alignment:**
- **Written deliverable length** — wiki pages are the clearest case for the no-padding
  instruction; a compounding wiki that grows verbose per edit becomes unusable fast.
- **Self-correction narration** — this is exactly what `log.md` is for: log substantive
  revisions, don't narrate every minor rewrite in the page itself.
- **Long-context** — Opus 5's 1M window with consistent instruction-following across it
  means merge-conflict resolution across many existing pages can be done by loading them
  directly rather than chunked retrieval, at small-to-medium wiki sizes.

---

## Stage 5 — Query & Synthesis Interface

**Goal:** User asks a question; agent reads `index.md`, picks relevant pages, answers from
the compiled wiki. Falls back to Stage 1/3 (live search) only when the wiki doesn't cover it.

**Pre-implementation requirements:**
- Index reader + relevance-selection logic.
- Gap-detection: a clear signal for "wiki doesn't have this" that routes to the harness
  instead of hallucinating an answer.

**Dependencies:** Stage 4.

**Prompting-guide alignment:**
- **Long-context** — load `index.md` plus the selected pages directly instead of
  vector-chunking; only add embedding-based retrieval once the wiki exceeds the practical
  single-context ceiling.
- **Response length/verbosity** — this is a different output mode from Stage 1/4 (a
  conversational answer, not a written page), so it needs its own concise-response
  instruction rather than reusing the report-writing prompt's length rules.

---

## Stage 6 — Maintenance / Lint Loop

**Goal:** Periodic pass over the wiki that flags contradictions, stale claims, orphaned
pages, and gaps — and queues the gaps as new search tasks for Stage 1/3.

**Pre-implementation requirements:**
- Trigger mechanism (cron, git hook, manual command).
- Contradiction-detection prompt and a gap-queue format the harness can consume.

**Dependencies:** Stage 4, Stage 5 (needs the query layer's gap-detection signal reused here).

**Prompting-guide alignment:**
- **Self-correction** — apply the "only correct/narrate when it changes a conclusion"
  rule here directly: routine fixes (typos, dead links) get silently applied and logged;
  contradictions that change a page's substance get flagged for review, not auto-resolved.
- **Task scope constraint** — the lint pass should flag and queue, not silently rewrite
  large sections of the wiki on its own initiative; that's scope creep in exactly the form
  the guide warns about.

---

## Stage 7 — Interface & Packaging

**Goal:** Expose the whole thing as a CLI and/or MCP server; decide whether the
orchestration layer stays in Python or gets rewritten in Rust for the search/merge hot path.

**Pre-implementation requirements:**
- Working, evaluated prototype from Stages 1–6.
- MCP server spec if you want Claude Code/other agents to use the wiki directly.
- Config/auth handling for the search backend and model API.

**Dependencies:** All prior stages validated.

**Prompting-guide alignment:**
- **Effort** — production defaults: `low`/`medium` for routine wiki queries (Stage 5),
  reserved `high`/`xhigh` for new multi-perspective investigations (Stage 3). Re-sweep once
  you have real usage evals rather than carrying Stage-1 defaults forward.
- **Subagent caps become deterministic, not just prompted** — once this moves to a Rust
  orchestration layer, enforce the perspective/sub-agent limits from Stage 3 in code, not
  only in the prompt.

---

## Cross-cutting notes

- **Don't duplicate self-verification.** Any agent role that reflects on its *own* prior
  output without new evidence (Stage 1's writer, Stage 4's page-merger) should **not** get
  a "double-check yourself" instruction — Opus 5 already does this. Only Stage 2's
  judge role, which checks against *external* retrieved evidence, is genuine verification
  and belongs in the pipeline.
- **Narration is a per-surface decision, not global.** Stage 1/3 (live search) benefits
  from the outcome-first narration pattern if shown to a user; Stage 4/6 (background
  maintenance) should stay silent except for `log.md` entries.
- **If any stage runs with thinking disabled** for latency (e.g. a cheap routing/gap-check
  classifier), add the guide's combined mitigation instruction (permission to speak before
  a tool call, fallback when no tool fits, no internal XML tags) rather than naming
  "thinking" directly in the prompt.
