# pybuilder

**PRD-driven, evaluation-gated Python code generation** — the Python analog of the
Rust [`/autobuilder`](https://github.com/joeyen-atscale/autobuilder-private) skill, applying the four
pillars of *test-driven development for LLM applications* as first-class pipeline
primitives.

Where autobuilder advances a Rust build on a single deterministic pass/fail signal,
pybuilder advances LLM-bearing Python on a **quality vector** and refuses to ship on
vibes. The thing that grades the AtScale semantic-layer thesis (text-to-SQL graders,
MCP trajectory audits) should itself be proven — that is what this builds.

## The four pillars (and where they live)

| Pillar (from the TDD-of-LLM article) | Module |
|---|---|
| **State-machine modeling** — the flowchart *is* the code; traces become tests | [`pybuilder.flowgraph`](src/pybuilder/flowgraph.py) |
| **Multi-dimensional eval** — a vector (exact/fuzzy/static/human/judge), not a boolean | [`pybuilder.eval`](src/pybuilder/eval/) |
| **Log-don't-assert** — record every dimension per run, aggregate to CSV/DataFrame | [`pybuilder.eval.results_bag`](src/pybuilder/eval/results_bag.py) |
| **Stability + provenance** — N-run variance → spec signal; every row git-commit-pinned | [`pybuilder.stability`](src/pybuilder/stability.py), [`pybuilder.dataset`](src/pybuilder/dataset.py) |

Plus the receipt-based [`pybuilder.gate`](src/pybuilder/gate.py) (ready/blocked, no
self-approval), the AST-based [`pybuilder.bad_python`](src/pybuilder/bad_python.py) audit,
the always-Opus calibrated [`pybuilder.judge`](src/pybuilder/judge.py), and the
locked-harness [`pybuilder.scaffold`](src/pybuilder/scaffold.py).

## Targets

`--target cli | lib | agent`. **`agent` is first-class** (the state-machine target is the
reason this tool exists, not a v2 add-on).

## Prerequisites

- **Python 3.11+**
- **[uv](https://docs.astral.sh/uv/)** — `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **git**
- **Claude Code** (for the `/pybuilder` skill)

## Quickstart

```bash
git clone https://github.com/joeyen-atscale/pybuilder.git
cd pybuilder
uv sync --dev
uv run pytest -q            # the suite
uv run pybuilder demo       # the whole pipeline on a tiny agent, no network
uv run pybuilder audit src  # BAD_PYTHON scan
```

Install the Claude Code skill (drops `/pybuilder` into `~/.claude/skills/`):

```bash
bash install.sh
```

Or without cloning first (curl-pipe):

```bash
curl -fsSL https://raw.githubusercontent.com/joeyen-atscale/pybuilder/main/install.sh | bash
```

## Layout

```
src/pybuilder/        the pipeline (one package; modules map 1:1 to the PRD components)
skill/                SKILL.md + intake/judge/reviewer prompts + intent-card schema
tests/                pytest suite (log-don't-assert, stability, flowgraph, gate, audit, …)
install.sh            curl|bash self-cloning installer
```

## Status

Bootstrap v0.1. Built and self-gated by hand (chicken-and-egg: no Python builder
existed to build the Python builder). Dual MIT/Apache-2.0.

**Network:** The `judge` optional dep calls Claude (`claude-sonnet-4-6` by default)
and requires `ANTHROPIC_API_KEY` in the environment. The core pipeline (scaffold,
audit, gate) is network-free; only the judge dimension degrades when the key is absent.
