---
name: research-project-init
description: >
  Scaffolds a new Python or Rust research project with the full standard toolchain.
  Use this skill whenever the user wants to start a new project, create a new repo,
  initialise a codebase, or set up a package. Triggers on phrases like "new project",
  "set up a repo", "initialise a project", "scaffold", or "start a codebase". Always
  use this skill — do not scaffold projects from memory, as the toolchain and structure
  are precisely specified here.
---

# Research Project Init Skill

## Toolchain

### Python

| Tool | Role |
|---|---|
| `uv` | Package and environment management — always; never pip, conda, or poetry |
| `ruff` | Linting and formatting |
| `ty` | Type checking |
| `pytest` | Testing |
| `pre-commit` | Local enforcement of ruff and ty before commit |
| GitHub Actions | CI: ruff, ty, pytest on every push and PR |
| Git LFS | Large file storage for data files |
| W&B | Experiment tracking (optional — ask at init time) |

### Rust

| Tool | Role |
|---|---|
| `cargo` | Build, test, and dependency management — always |
| `rustfmt` | Formatting (via `cargo fmt`) |
| `clippy` | Linting (via `cargo clippy -- -D warnings`) |
| `cargo test` | Testing |
| GitHub Actions | CI: fmt, clippy, test on every push and PR |
| Git LFS | Large file storage for data files |

### Mixed projects (Python + Rust)

Both toolchains are active. Python bindings to Rust are handled via `maturin` and
`PyO3`. The Python package wraps the Rust crate; `uv` manages the Python environment.

---

## Step-by-step init procedure

### 1 — Gather inputs

Ask the human for:

1. **Project language** — Python, Rust, or both?
1. **Project name** — this becomes the repo name and the primary package/crate name
   (use underscores for Python packages, hyphens for Rust crates and repos,
   e.g. repo `quantum-sensing`, Python package `quantum_sensing`, Rust crate `quantum-sensing`).
2. **One-sentence description** — used in `pyproject.toml`/`Cargo.toml` and `README.md`.
3. **W&B?** — Python and mixed projects only: "Will this project use Weights & Biases?" (yes/no).
   - If yes: "What is your W&B entity?" and "What is your W&B project name?"

Do not proceed until these are confirmed.

---

### 2 — Create directory structure

Adapt based on `LANGUAGE`. Create only the directories applicable to the language.

#### Python projects (LANGUAGE = python)

```
<repo-name>/
├── <package_name>/          # Python package (named after the project)
│   └── __init__.py
├── scripts/                 # Standalone entrypoint scripts (not part of package)
│   └── .gitkeep
├── data/                    # Data files — tracked via Git LFS
│   └── .gitkeep
├── plots/                   # Output figures — gitignored
│   └── .gitkeep
├── notes/                   # Private notes — gitignored
│   └── .gitkeep
├── tests/
│   └── __init__.py
├── configs/                 # Config files (YAML, TOML)
│   └── .gitkeep
├── .github/
│   ├── workflows/
│   │   └── ci.yml
│   └── PULL_REQUEST_TEMPLATE.md
├── .claude/
│   ├── agents/
│   ├── hooks/
│   └── settings.json
├── .gitignore
├── .gitattributes           # Git LFS tracking rules
├── .pre-commit-config.yaml
├── pyproject.toml
├── README.md
└── .env.example             # Always; .env itself is gitignored
```

#### Rust projects (LANGUAGE = rust)

```
<repo-name>/
├── src/
│   └── lib.rs               # or main.rs for a binary crate
├── tests/                   # Rust integration tests
│   └── .gitkeep
├── data/                    # Data files — tracked via Git LFS
│   └── .gitkeep
├── plots/                   # Output figures — gitignored
│   └── .gitkeep
├── notes/                   # Private notes — gitignored
│   └── .gitkeep
├── configs/                 # Config files (YAML, TOML)
│   └── .gitkeep
├── .github/
│   ├── workflows/
│   │   └── ci.yml
│   └── PULL_REQUEST_TEMPLATE.md
├── .claude/
│   ├── agents/
│   ├── hooks/
│   └── settings.json
├── .gitignore
├── .gitattributes           # Git LFS tracking rules
├── Cargo.toml
└── README.md
```

Note: no `scripts/`, no `<package_name>/`, no `.env.example` for pure Rust projects
unless W&B or other secrets are in use (unlikely for Rust-only).

#### Mixed projects (LANGUAGE = both)

```
<repo-name>/
├── <package_name>/          # Python package
│   └── __init__.py
├── src/                     # Rust crate (e.g. PyO3 extension module)
│   └── lib.rs
├── scripts/                 # Python entrypoint scripts
│   └── .gitkeep
├── data/
│   └── .gitkeep
├── plots/
│   └── .gitkeep
├── notes/
│   └── .gitkeep
├── tests/
│   └── __init__.py          # Python tests; Rust unit tests are inline in src/
├── configs/
│   └── .gitkeep
├── .github/
│   ├── workflows/
│   │   └── ci.yml
│   └── PULL_REQUEST_TEMPLATE.md
├── .claude/
│   ├── agents/
│   ├── hooks/
│   └── settings.json
├── .gitignore
├── .gitattributes
├── .pre-commit-config.yaml
├── pyproject.toml
├── Cargo.toml
├── README.md
└── .env.example
```

---

### 3 — Initialise uv and pyproject.toml

```bash
uv init <repo-name>
cd <repo-name>
```

`pyproject.toml`:

```toml
[project]
name = "<repo-name>"
version = "0.1.0"
description = "<one-sentence description>"
requires-python = ">=3.14"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["<package_name>"]

[tool.ruff]
line-length = 88
target-version = "py314"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]
ignore = []

[tool.ruff.format]
quote-style = "double"

[tool.ty]
# ty configuration — add per-project overrides here

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Add dev dependencies:

```bash
uv add --dev ruff ty pytest pre-commit
```

---

### 4 — .gitignore

Emit all sections applicable to the project language.

```gitignore
# Environment
.env

# Output (regenerable)
plots/
notes/

# Editors
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# ── Python (include if LANGUAGE = python or both) ──────────────────────────────
.venv/
__pycache__/
*.py[cod]
.uv/

# W&B (if applicable)
wandb/

# ── Rust (include if LANGUAGE = rust or both) ──────────────────────────────────
target/
# Note: commit Cargo.lock for binaries; do not commit it for libraries.
# Ask the human which applies, and gitignore Cargo.lock only for libraries.
```

---

### 5 — Git LFS

```bash
git lfs install
```

`.gitattributes`:

```
*.npy    filter=lfs diff=lfs merge=lfs -text
*.npz    filter=lfs diff=lfs merge=lfs -text
*.hdf5   filter=lfs diff=lfs merge=lfs -text
*.h5     filter=lfs diff=lfs merge=lfs -text
*.pt     filter=lfs diff=lfs merge=lfs -text
*.pkl    filter=lfs diff=lfs merge=lfs -text
*.csv    filter=lfs diff=lfs merge=lfs -text
```

Note: `data/` is **not** gitignored. Files within it are committed via LFS. This
ensures reproducibility. Warn the human if LFS is not available on their remote.

---

### 6 — pre-commit

#### Python projects (LANGUAGE = python or both)

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/astral-sh/ty
    rev: 0.0.0-alpha.6    # update to latest stable
    hooks:
      - id: ty
```

#### Rust projects (LANGUAGE = rust or both)

Rust does not use pre-commit for `cargo fmt` and `cargo clippy` — these are
handled by the Claude Code pre-commit gate hook (see `hooks` skill) and the
GitHub Actions CI workflow. This is because pre-commit's Rust support requires
the system Rust toolchain rather than the project-pinned version, which can
produce inconsistent results.

For mixed projects, use the Python `.pre-commit-config.yaml` above for ruff/ty,
and rely on the Claude Code hook and CI for Rust checks.

Install hooks (Python and mixed projects only):

```bash
uv run pre-commit install
```

---

### 7 — GitHub Actions CI

#### Python projects — `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Install dependencies
        run: uv sync --dev

      - name: Lint (ruff)
        run: uv run ruff check .

      - name: Format check (ruff)
        run: uv run ruff format --check .

      - name: Type check (ty)
        run: uv run ty check

      - name: Tests (pytest)
        run: uv run pytest
```

#### Rust projects — `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

env:
  CARGO_TERM_COLOR: always

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - name: Install Rust stable
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - name: Cache cargo
        uses: Swatinem/rust-cache@v2

      - name: Format check
        run: cargo fmt --check

      - name: Clippy
        run: cargo clippy -- -D warnings

      - name: Tests
        run: cargo test
```

#### Mixed projects — `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

env:
  CARGO_TERM_COLOR: always

jobs:
  rust:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - name: Install Rust stable
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - name: Cache cargo
        uses: Swatinem/rust-cache@v2

      - name: Format check
        run: cargo fmt --check

      - name: Clippy
        run: cargo clippy -- -D warnings

      - name: Tests
        run: cargo test

  python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Install dependencies
        run: uv sync --dev

      - name: Lint (ruff)
        run: uv run ruff check .

      - name: Format check (ruff)
        run: uv run ruff format --check .

      - name: Type check (ty)
        run: uv run ty check

      - name: Tests (pytest)
        run: uv run pytest
```

---

### 8 — PR template

`.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Why
<!-- What problem does this solve? What decision was made and why? -->

## How
<!-- What approach was taken? Key design decisions. -->

## What
<!-- Concrete list of changes: files added/modified, functions changed, behaviour altered. -->

## Tests
<!-- What is tested? How? If no tests: justify why. -->

## Physics & numerical considerations
<!-- Discretisation choices, stability, units, boundary conditions, known limitations.
     Write "N/A" only if the code has zero physical or numerical content. -->

## Checklist
- [ ] Types pass (`ty check`)
- [ ] Lint passes (`ruff check`)
- [ ] Tests pass (`pytest`)
- [ ] Data files not committed accidentally (only via LFS)
- [ ] No hardcoded paths or secrets
```

---

### 9 — W&B (if yes)

Create `.env`:

```bash
WANDB_API_KEY=
WANDB_ENTITY=<entity>
WANDB_PROJECT=<project-name>
```

Create `.env.example` (committed to repo):

```bash
WANDB_API_KEY=
WANDB_ENTITY=<entity>
WANDB_PROJECT=<project-name>
```

Create `configs/wandb.yaml`:

```yaml
entity: <entity>
project: <project-name>
```

Create `<package_name>/logging.py`:

```python
"""Experiment logging utilities."""

import os
from pathlib import Path

import wandb
from dotenv import load_dotenv

load_dotenv()


def init_run(config: dict, name: str | None = None) -> wandb.sdk.wandb_run.Run:
    """Initialise a W&B run with the given config."""
    return wandb.init(
        entity=os.environ["WANDB_ENTITY"],
        project=os.environ["WANDB_PROJECT"],
        name=name,
        config=config,
    )
```

Add W&B and python-dotenv as dependencies:

```bash
uv add wandb python-dotenv
```

Add `wandb/` to `.gitignore` (already included in template above).

---

### 10 — README.md

```markdown
# <repo-name>

<one-sentence description>

## Installation

Requires [uv](https://github.com/astral-sh/uv).

```bash
git clone https://github.com/<owner>/<repo-name>
cd <repo-name>
uv sync
```

## Development

```bash
uv run pytest          # run tests
uv run ruff check .    # lint
uv run ty check        # type check
```

Pre-commit hooks are installed automatically on first `git commit` after `uv sync`.

## Structure

| Directory | Contents |
|---|---|
| `<package_name>/` | Core library code |
| `scripts/` | Standalone entrypoint scripts |
| `data/` | Data files (tracked via Git LFS) |
| `plots/` | Generated figures (gitignored) |
| `notes/` | Private notes (gitignored) |
| `tests/` | Test suite |
| `configs/` | Configuration files |
```

---

### 11 — Initialise git and push

All repositories are created under the `QuantumSensing` organisation. The remote
URL is always:

```bash
git init
git add .
git commit -m "chore: initialise project scaffold"
git branch -M main
git remote add origin https://github.com/QuantumSensing/<repo-name>.git
git push -u origin main
```

Remind the human to create the GitHub repo under `QuantumSensing` before pushing
if it does not yet exist: https://github.com/organizations/QuantumSensing/repositories/new

---

### 12 — Invoke `project-md` skill

After the repo is pushed, invoke the `project-md` skill to generate `CLAUDE.md`.
This requires answers to the project type and context questions defined in that
skill. Commit the result:

```bash
git add CLAUDE.md
git commit -m "chore: add CLAUDE.md project context"
git push
```

---

### 13 — Invoke `agent-team` skill

Based on the `PROJECT_TYPE` declared during `CLAUDE.md` generation, invoke the
`agent-team` skill to install agent definition files into `.claude/agents/`.
Commit:

```bash
git add .claude/agents/
git commit -m "chore: add Claude Code agent definitions"
git push
```

---

### 14 — Invoke `hooks` skill

Invoke the `hooks` skill to write `.claude/settings.json` and
`.claude/hooks/pre-commit-gate.sh`. Commit:

```bash
git add .claude/
git commit -m "chore: add Claude Code pre-commit gate hook"
git push
```

---

### 15 — Invoke `github-projects` skill

Hand off to the `github-projects` skill to complete project setup:

1. Delete default GitHub labels and create: `Writing`, `Coding`, `Experiments`,
   `Research`, `Won't Fix`.
2. Create a GitHub Project named after the repository under `QuantumSensing`.
3. Configure the board with columns: `Backlog → Todo → In Progress → In Review → Done`.
4. Link the repository to the project.

Follow the **Project initialisation** section of the `github-projects` skill.

---

## Post-init checklist (present to human)

```
Project scaffold complete. Before starting work:

[ ] Create the GitHub repo at https://github.com/organizations/QuantumSensing/repositories/new
[ ] Run: git push -u origin main
[ ] Add WANDB_API_KEY to .env (if using W&B)
[ ] Add GITHUB_TOKEN to .env — must have repo + project scopes
[ ] Confirm Git LFS is enabled on the remote (Settings → Git LFS)
[ ] Install pre-commit hooks: uv run pre-commit install
[ ] Confirm GitHub Project board columns are set correctly (or set manually if API unavailable)
[ ] Board URL: https://github.com/orgs/QuantumSensing/projects/{project_number}
[ ] Review CLAUDE.md and update physics/ML context if needed
```
