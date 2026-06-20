# pybuilder

Build a Python CLI, library, or LLM agent from a PRD, under a loop that won't call the work done until the evidence says so.

## Why it exists

A passing test suite tells you the code didn't crash on the inputs you thought of. It doesn't tell you the program does what you asked — and for code that calls a language model, it doesn't even tell you the same input gives the same answer twice. The usual build loop optimizes for green, and green is the wrong target.

pybuilder optimizes for evidence instead. You describe what you want in a PRD; it derives unambiguous acceptance criteria, scaffolds a harness it can't quietly weaken, builds against a quality vector rather than a single pass/fail, and runs the same inputs N times to separate acceptable model noise from a prompt that's underspecified. A project is "ready" only when an independent reviewer — a model that didn't write the code — agrees the criteria are met. It is the Python counterpart to [autobuilder](https://github.com/joeyen-atscale/autobuilder), which does the same for Rust.

The `agent` target is the reason the tool exists. An agent is modeled as an explicit, traced state machine — the flowchart is the code — so a production trace can become a replayable test, and a class of LLM-application bugs becomes mechanically catchable.

## What it builds

| Target | What it is | Use it for |
|---|---|---|
| `cli` | a terminal command | automation, file processing, scheduled tasks |
| `lib` | importable code | shared logic, data models, API wrappers |
| `agent` | a traced state machine that reasons | language-model workflows, multi-step adaptive behavior |

## Install

Requires **Python 3.11+**, **[uv](https://docs.astral.sh/uv/)**, **git**, and **[Claude Code](https://claude.com/claude-code)**. The CLI install path is `uv tool install`; there's no pip/Docker/cloud path.

One step installs both Claude Code skills (`/prd-writer`, `/pybuilder`) and the `pybuilder` CLI:

```bash
curl -fsSL https://raw.githubusercontent.com/joeyen-atscale/pybuilder/main/install.sh | bash
```

Or from a checkout:

```bash
git clone https://github.com/joeyen-atscale/pybuilder.git
cd pybuilder && ./install.sh
```

The installer self-clones when piped from curl, symlinks the two skills into `~/.claude/skills/`, and installs the CLI via `uv tool install`. If `uv` isn't present it skips the CLI and tells you the command to run once you have it.

## Quickstart

```bash
pybuilder demo
```

`demo` runs the whole pipeline on a tiny two-state agent end to end — state machine and trace, multi-dimensional eval, stability classification, a receipt-driven gate verdict — with no network call and no optional dependency. It's the fastest way to see what "ready" is made of.

For a real project, start from a PRD inside Claude Code. If you don't have one, `/prd-writer` turns an idea into a buildable spec by asking questions, not by handing you a form. Then:

```
/pybuilder path/to/your-prd.md
```

The CLI itself exposes the four mechanical steps the skill orchestrates:

```bash
pybuilder scaffold <intent-card.json> <dest>   # generate a locked-harness project
pybuilder audit    <path>                       # scan for semantic footguns ruff can't see
pybuilder gate     <project> --target cli|lib|agent
pybuilder demo
```

Intake — turning a PRD into the structured `intent-card.json` that `scaffold` consumes — is a reasoning step the skill drives with a model, not a CLI subcommand.

## How it works

Five stages, each run by the model that fits its job — strong models where judgment is needed, fast ones where the work is mechanical.

1. **Intake (5-Whys)** — read the PRD, extract acceptance criteria, refuse to proceed while any MUST-criterion is ambiguous. Emits `intent-card.json`.
2. **Scaffold** — generate a tree where `tests/`, `scripts/`, and config are read-only and the model edits only `src/`. The empty tree is already green under ruff + mypy + pytest. An `agent` target also gets a state-machine skeleton.
3. **Iterate-and-prove** — edit `src/`, run the metrics harness, score the artifact across the quality vector, and for model-bearing artifacts run the stability check. Advance only on a net improvement with no MUST-criterion regression; otherwise revert.
4. **Risk gate** — a project is `ready` only when every required receipt passes. `cli`/`lib` need intake, scaffold-integrity, proof (ruff/mypy/pytest/coverage), audit, reviewer, and CI; `agent` additionally needs eval-vector, stability, and dataset-provenance. The reviewer receipt is an independent model reading the criteria in plain English — no self-approval. Missing, failed, or stale receipts block, with a machine-readable diagnostic.
5. **Postmortem** — summarize the run and queue an append-only evolution proposal.

The four ideas underneath, drawn from test-driven development for LLM applications:

- **State-machine modeling** (`pybuilder.flowgraph`) — agents are explicit, traced state machines; traces replay as tests.
- **Multi-dimensional eval** (`pybuilder.eval`) — exact, fuzzy, static, human, and judge dimensions collapsed to one quality vector, not a single pass/fail.
- **Log-don't-assert** (`pybuilder.eval.results_bag`) — every dimension is recorded per run; a miss doesn't abort the suite, it becomes a row.
- **Stability and provenance** (`pybuilder.stability`, `pybuilder.dataset`) — the same input N times tells noise apart from underspecification; every eval row is pinned to the commit that produced it.

### Models

Defaults live in `src/pybuilder/models.py` and override via environment variables. Splitting models keeps the tool usable on lower API rate limits while spending the strongest model only where it changes the outcome.

| Role | Default | Override |
|---|---|---|
| Build loop — intake, code, eval | Sonnet | `PYBUILDER_BUILD_MODEL` |
| Mechanical — scaffold, audit, gate checks, postmortem | Haiku | `PYBUILDER_FAST_MODEL` |
| Independent reviewer | Opus | `PYBUILDER_REVIEW_MODEL` |

The core pipeline — scaffold, audit, gate receipt-checks — runs locally with no network call. The judge dimension and the reviewer need an `ANTHROPIC_API_KEY`; without one, the judge dimension reports `unavailable` rather than failing.

## Layout

```
src/pybuilder/        the pipeline
skill/                the /pybuilder Claude Code skill
skills/prd-writer/    the /prd-writer skill (bundled)
tests/                the test suite
examples/selfgate.py  pybuilder running its own gate on this repo
install.sh            installs both skills and the CLI
```

## Status

Version 0.1.0, and honest about it. pybuilder was hand-built — there was no Python builder to build it with — so it hasn't yet been driven end to end by its own loop. `examples/selfgate.py` runs the real proof and audit checks against this repo and marks the orchestration receipts with bootstrap detail rather than pretending an in-loop reviewer ran. The pipeline, the CLI, and the eval machinery are real and tested; the closed loop running unattended on an external PRD is the work ahead.

## License

MIT OR Apache-2.0.
