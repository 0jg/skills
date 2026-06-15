---
name: project-md
description: >
  Generates a project-specific CLAUDE.md file at project initialisation time.
  Use this skill whenever starting a new project, forking a repo, or when the
  user asks to set up Claude Code for a project. Always invoke this skill as
  part of the research-project-init workflow — do not write CLAUDE.md from
  memory. The generated file encodes project type, physics context, toolchain
  rules, directory conventions, agent configuration, and skill triggers so that
  all context persists across sessions and survives context compaction.
---

# CLAUDE.md Generation Skill

## Purpose

`CLAUDE.md` is the single source of truth Claude Code loads at the start of every
session. It must encode everything Claude needs to work correctly on this project
without re-explanation. Write it once at init time; update it when the project
changes materially.

---

## Step 1 — Gather inputs at init time

Ask the human the following questions in sequence. Do not proceed until all are
answered.

### 1a — Language

```
What language(s) does this project use?

1. Python only
2. Rust only
3. Both (Python + Rust, e.g. PyO3 bindings or separate components)
```

Store as `LANGUAGE ∈ {python, rust, both}`.

### 1c — Project type

```
What type of project is this?

1. ML          — machine learning models, training loops, W&B, data pipelines
2. Physics     — numerical simulation, PDEs, stochastic equations, physical units
3. Both        — ML applied to physical systems (e.g. surrogate models, BEC, atomtronics)
4. General     — software, tooling, analysis; no domain-specific physics or ML
```

Store the answer as `PROJECT_TYPE ∈ {ml, physics, both, general}`.

### 1d — Physics context (skip if PROJECT_TYPE = ml or general)

Ask:

```
Describe the physical system briefly:
- What equations govern the system? (e.g. SGPE, GPE, Maxwell-Bloch)
- What dimensionality? (1D / 2D / 3D)
- What unit system? (SI / natural / dimensionless)
- Key physical constraints to enforce? (e.g. normalisation, conservation laws,
  boundary conditions, symmetry)
- Known numerical considerations? (e.g. timestep stability, grid resolution,
  stiffness)
```

Store verbatim as `PHYSICS_CONTEXT`.

### 1e — ML context (skip if PROJECT_TYPE = physics or general)

Ask:

```
Describe the ML setup briefly:
- Model architecture family? (e.g. diffusion model, UNet, transformer, reservoir)
- Training framework? (PyTorch assumed unless stated otherwise)
- Key correctness constraints? (e.g. output normalisation, loss function choices,
  evaluation metrics)
- W&B in use? (yes/no — if yes, entity and project name)
```

Store verbatim as `ML_CONTEXT`.

### 1f — Project description

```
One sentence: what does this project do?
```

---

## Step 2 — Generate CLAUDE.md

Write to `CLAUDE.md` in the project root. Use the template below, substituting
all bracketed fields.

---

## CLAUDE.md template

```markdown
# CLAUDE.md — <repo-name>

> <one-sentence project description>

This file is loaded by Claude Code at the start of every session. It encodes all
project conventions, context, and instructions. Do not override these without
explicit instruction from the human.

---

## Project type

<One of: ML | Physics simulation | ML + Physics | General>

---

## Physics context

<!-- Populated for PROJECT_TYPE = physics or both. Omit section entirely for ml/general. -->

<PHYSICS_CONTEXT verbatim>

**Invariants to check on every numerical implementation:**
- <list key conservation laws, normalisation conditions, symmetry requirements>
- Raise an explicit comment if an implementation may violate these.

---

## ML context

<!-- Populated for PROJECT_TYPE = ml or both. Omit section entirely for physics/general. -->

<ML_CONTEXT verbatim>

**Correctness requirements:**
- <list key ML constraints, e.g. output range, loss function validity>
- Flag any architecture choice that deviates from established practice for this
  problem class.

---

## Language

<One of: Python | Rust | Python + Rust>

---

## Repository structure

<!-- Python and mixed projects -->
<!-- If LANGUAGE = python or both: -->
| Path | Contents |
|---|---|
| `<package_name>/` | Core Python library — importable package |
| `src/` | Rust source (`src/lib.rs` or `src/main.rs`) — present for Rust/mixed projects |
| `scripts/` | Standalone entrypoint scripts; not part of the package |
| `data/` | Data files tracked via Git LFS; never gitignored |
| `plots/` | Generated figures; always PDF at 300 DPI |
| `literature/` | PDFs, papers, and other useful info |
| `notes/` | Working notes |
| `tests/` | Python test suite (pytest) and/or Rust integration tests (`tests/*.rs`) |
| `configs/` | YAML/TOML configuration files |
| `.claude/` | Claude Code configuration: agents, hooks, settings |

**Python projects**: core library is `<package_name>/`; tests are in `tests/test_*.py`.
**Rust projects**: source is in `src/`; unit tests are inline (`#[cfg(test)]`); integration tests are in `tests/*.rs`. No `<package_name>/` directory.
**Mixed projects**: both `<package_name>/` (Python) and `src/` (Rust) are present.

---

## Toolchain

<!-- Emit only the rows applicable to LANGUAGE -->

### Python
| Tool | Role | Command |
|---|---|---|
| `uv` | Environment and package management | `uv sync`, `uv add` |
| `ruff` | Lint and format | `uv run ruff check .` / `uv run ruff format .` |
| `ty` | Type checking | `uv run ty check` |
| `pytest` | Testing | `uv run pytest` |
| `pre-commit` | Pre-commit hooks | `uv run pre-commit install` |

Never use `pip`, `conda`, `poetry`, or `pipenv`. Always use `uv`.

### Rust
| Tool | Role | Command |
|---|---|---|
| `cargo` | Build, test, dependency management | `cargo build`, `cargo test` |
| `rustfmt` | Formatting | `cargo fmt` |
| `clippy` | Linting | `cargo clippy -- -D warnings` |

All checks must pass before any commit. The pre-commit gate enforces this as a hard block.

---

## Code conventions

<!-- Python conventions — omit if LANGUAGE = rust -->
**Python**
- Python 3.14+; type annotations on all public functions.
- Declarative, assertive naming; no abbreviations except established domain terms
  (e.g. `sgpe`, `bec`, `unet`).
- Single responsibility per function; no side effects unless explicitly documented.
- Magic numbers must be named constants with units in the name or a comment.
- Simulation/analysis code never produces figures directly. Save results to
  `data/<name>.csv`; plot via `scripts/plot_<name>.py`.
- All plot scripts follow the `research-figures` skill house style.

<!-- Rust conventions — omit if LANGUAGE = python -->
**Rust**
- All public items must have `///` doc comments; use LaTeX notation for mathematics.
- Prefer explicit error types; use `anyhow` for binaries, `thiserror` for libraries.
- Named `const` for all magic numbers; include units in the name.
- No `unwrap()` in production code; propagate errors explicitly.
- Simulation/analysis code never produces figures directly. Save results to
  `data/<name>.csv`.

**Both languages**
- Notes and working documents go in `notes/`; never commit them.
- No ASCII approximations of mathematical notation in comments or docstrings.
  Use LaTeX throughout: `\alpha`, `x^2` not `x**2`, `\partial x / \partial t`. Do not use superscript or subscript numbers in ASCII, use e.g., ^87 Rb or rubidium-87. Use 10^{-25} instead.

---

## Writing conventions

- British English throughout (colour, behaviour, optimise, etc.).
- Terse declarative prose; semicolons over conjunctions where appropriate.
- No filler phrases ("it is important to note", "in order to", "various").
- No ASCII approximations of mathematical notation in code comments or docstrings.
  Use LaTeX throughout: `\alpha`, `x^2` not `x**2`, `\partial x / \partial t`,
  `x \in \mathbb{R}`, etc. This applies to inline comments as well as docstrings.

---

## Active skills

The following skills govern this project. Claude Code loads them automatically.

| Skill | Trigger |
|---|---|
| `research-project-init` | New repo or fork setup |
| `github-projects` | Session start/end, task planning, board updates |
| `code-review` | PR creation, post-merge cleanup |
| `research-figures` | Any matplotlib or plotting code |
| `experiment-log` | W&B run summary |
| `agent-team` | Any non-trivial implementation task |
| `hooks` | Pre-commit gate configuration |

---

## Agent team

<!-- Section varies by PROJECT_TYPE. See agent-team skill for full agent definitions. -->

<INSERT AGENT BLOCK — see below>

---

## Session protocol

### Session start
1. Run `github-projects` Workflow A: fetch board, propose tasks, wait for approval.
2. Confirm all checks pass:
   - Python: `uv run ruff check . && uv run ty check && uv run pytest`
   - Rust: `cargo fmt --check && cargo clippy -- -D warnings && cargo test`
   - Mixed: run both sets above.
3. Move the first issue to `In Progress` on the board.

### During session
- Commit regularly; each commit should be small and purposeful.
- Branch per issue: `git checkout -b issue-<number>-<short-description>`
- Never commit directly to `main`.
- Invoke `agent-team` for any implementation task involving more than one file or
  any physics/ML logic.

### Session end
1. Open PR via `code-review` skill; AI review posted to GitHub.
2. Move issue to `In Review`; block until human confirms merge.
3. On merge: move issue to `Done`; run `github-projects` Workflow C.

---

## GitHub

- Organisation: `QuantumSensing`
- Repo: `QuantumSensing/<repo-name>`
- Project board: `https://github.com/orgs/QuantumSensing/projects/<number>`
- PR target: `main` on this repo (not upstream, if forked)
- Max PR size: 200 lines changed

---

## Environment

Secrets live in `.env` (gitignored). Required keys:

```
GITHUB_TOKEN=        # repo + project scopes
WANDB_API_KEY=       # if W&B is active
WANDB_ENTITY=        # if W&B is active
WANDB_PROJECT=       # if W&B is active
```

Load with: `from dotenv import load_dotenv; load_dotenv()`
```

---

## Step 3 — Insert agent block

Substitute `<INSERT AGENT BLOCK>` based on `PROJECT_TYPE`. In all variants,
update the `test-writer` line to reflect `LANGUAGE`:
- Python only: "writes and runs pytest tests"
- Rust only: "writes and runs cargo tests"
- Both: "writes and runs pytest and cargo tests"

### If PROJECT_TYPE = ml

```markdown
This project uses a two-agent team: `implementer` and `ml-reviewer`.
The `test-writer` agent is always active regardless of project type.
See `.claude/agents/` for full agent definitions.

Active agents:
- `implementer` — writes and edits code, runs bash
- `ml-reviewer` — read-only ML correctness review, web search enabled
- `test-writer` — writes and runs tests (pytest and/or cargo test)
```

### If PROJECT_TYPE = physics

```markdown
This project uses a two-agent team: `implementer` and `physics-reviewer`.
The `test-writer` agent is always active regardless of project type.
See `.claude/agents/` for full agent definitions.

Active agents:
- `implementer` — writes and edits code, runs bash
- `physics-reviewer` — read-only numerical/physics review, web search enabled
- `test-writer` — writes and runs tests (pytest and/or cargo test)
```

### If PROJECT_TYPE = both

```markdown
This project uses a three-agent team: `implementer`, `physics-reviewer`,
and `ml-reviewer`. The `test-writer` agent is always active.
See `.claude/agents/` for full agent definitions.

Active agents:
- `implementer` — writes and edits code, runs bash
- `physics-reviewer` — read-only numerical/physics review, web search enabled
- `ml-reviewer` — read-only ML correctness review, web search enabled
- `test-writer` — writes and runs tests (pytest and/or cargo test)
```

### If PROJECT_TYPE = general

```markdown
This project uses a minimal agent setup: `implementer` and `test-writer`.
See `.claude/agents/` for full agent definitions.

Active agents:
- `implementer` — writes and edits code, runs bash
- `test-writer` — writes and runs tests (pytest and/or cargo test)
```

---

## Step 4 — Confirm

After writing `CLAUDE.md`, tell the human:

```
CLAUDE.md written for <repo-name>.

Language:     <python|rust|both>
Project type: <type>
Active agents: <list>

Review CLAUDE.md now and update the physics/ML context if anything is missing
or imprecise — this file governs every future session.
```
