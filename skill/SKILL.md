---
name: pybuilder
description: PRD-driven, evaluation-gated Python code generation. Use when the user wants to build a Python CLI, library, or LLM agent from a Product Requirements Document under an autonomous iterate-and-prove loop. The Python analog of /autobuilder, applying test-driven development for LLM applications — multi-dimensional log-don't-assert evaluation, N-run stability classification, git-commit-pinned eval datasets, and a receipt-based risk gate. Targets cli|lib|agent; agent (state-machine) is first-class.
---

# pybuilder

## What this skill does

Takes a PRD (file path or pasted text) and drives a 5-stage pipeline that yields a
Python project where every artifact is (a) generated from a structured
`intent-card.json` derived via 5-Whys, (b) scaffolded behind a **locked harness**
(ruff + mypy + pytest + coverage; the agent edits only `src/`), (c) evaluated across
a **quality vector** rather than a single pass/fail (the part autobuilder can't copy,
because LLM-bearing Python isn't deterministic), and (d) gated by structured receipts
before declaring "ready." It applies the four pillars of *TDD for LLM applications*:

1. **State-machine modeling** (`pybuilder.flowgraph`) — agents are explicit, traced
   state machines; the flowchart *is* the code, and production traces become
   replayable pytest cases.
2. **Multi-dimensional eval** (`pybuilder.eval`) — exact / fuzzy / static / human /
   judge, collapsed to a quality vector.
3. **Log-don't-assert** (`pybuilder.eval.results_bag`) — every dimension recorded per
   run without aborting on first miss; aggregated to CSV / DataFrame.
4. **Stability + provenance** (`pybuilder.stability`, `pybuilder.dataset`) — same input
   N times to separate acceptable LLM noise from an underspecified prompt; every eval
   row pinned to the producing git commit.

## When to invoke

- The user hands you a PRD and asks for a Python project (CLI, library, or agent).
- The user says "build me a Python X" with concrete acceptance criteria.
- Dogfooding pybuilder against one of its own components.

Do **not** invoke for: non-Python projects (use `/autobuilder` for Rust); surgical
edits to an existing Python file (use direct tools).

## Targets

`--target cli | lib | agent`. **`agent` is first-class** — the state-machine target is
the reason this tool exists. (Resolves vision OQ-1, decided 2026-06-12.)

## Pipeline

```
PRD ──► Stage 1: Intake (5-Whys)            ──► intent-card.json   (prompts/prd-intake-5whys.md)
        Stage 2: Scaffold (locked harness)  ──► <project>/ tree    (pybuilder scaffold)
        Stage 3: Iterate-and-Prove loop     ──► quality vector + stability per iteration
        Stage 4: Risk Gate (receipts)       ──► ready / blocked    (pybuilder gate)
        Stage 5: Postmortem + Self-Evolve    ──► gated proposal
```

### Stage 1 — Intake (5-Whys)
Run `prompts/prd-intake-5whys.md` against the PRD. Output validates against
`schemas/intent-card.schema.json` (`intent-card.v1`). Refuse to proceed if ambiguity
leaves any MUST-AC undefined — surface what's missing and ask.

### Stage 2 — Scaffold
`pybuilder scaffold intent-card.json <dest>`. Emits a tree where `tests/`, `scripts/`,
and config are read-only and the agent edits only `src/`. The empty tree is green
under ruff + mypy + pytest. For `--target agent`, also emits a `StateMachine` skeleton.

### Stage 3 — Iterate-and-Prove
Edit `src/` only. Each iteration: run `scripts/run-metrics.sh` (→ `metrics.json`),
evaluate the artifact across dimensions into a `ResultsBag`, compute the quality vector
(`pybuilder.eval.vector`), and for LLM-bearing artifacts run `pybuilder.stability`
(advance only if not `underspecified`). Advance on **net vector improvement AND no
MUST-AC regression**; else revert. Extract any failing real trace into a regression
case via `pybuilder.flowgraph.trace_to_testcase`.

### Stage 4 — Risk Gate
`pybuilder gate <project> --target <t>`. Reads receipts under `<project>/.pybuilder/`:
`intake`, `scaffold-integrity`, `proof` (ruff+mypy+pytest+coverage), `audit`
(BAD_PYTHON), `reviewer` (**always-Opus**, independent, against the AC English text),
`ci-checks`; and for agents `eval-vector` (no regression vs baseline), `stability`
(not `underspecified`), `dataset-provenance` (rows commit-pinned). Missing / stale /
blocked → block with a machine-readable diagnostic. No self-approval.

### Stage 5 — Postmortem & Self-Evolve
Summarize the run; queue an append-only `evolution-proposal.json`. Grow the BAD_PYTHON
catalog and the default dimension weights from accumulated runs.

## CLI

```bash
pybuilder scaffold <intent-card.json> <dest> [--force]
pybuilder audit    <path>                          # BAD_PYTHON AST scan
pybuilder gate     <project> --target cli|lib|agent
pybuilder demo                                      # whole pipeline on a tiny agent
```

## Layout

```
pybuilder/
├── skill/
│   ├── SKILL.md                       ← this file
│   ├── schemas/intent-card.schema.json
│   └── prompts/{prd-intake-5whys,judge,reviewer}.md
├── src/pybuilder/                     ← the pipeline (modules map 1:1 to the 8 PRDs)
├── tests/
└── install.sh
```

## Reused skills
- `/loop` — long-running iterate-and-prove cadence.
- `/verify` — final end-to-end app-run check (Stage 4).
- `/code-review` — the `reviewer` receipt (dispatch on Opus).
- `/claude-api` — confirm the judge's model id + call shape before judging.

## Status
Bootstrap v0.1: built and self-gated by hand (chicken-and-egg — no Python builder
existed to build the Python builder). Designed in 8 PRDs at `~/Documents/PRDs/`
(`visions/pybuilder.md`). Ships as ONE monorepo (deliberate departure from
autobuilder's per-slice publishing).
