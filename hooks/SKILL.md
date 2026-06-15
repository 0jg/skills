---
name: hooks
description: >
  Configures Claude Code hooks for a project. Use this skill at project
  initialisation time to write .claude/settings.json with pre-commit gates.
  Also use when the user asks to set up hooks, enforce commit standards, or
  add pre-commit checks. The hook blocks any git commit that fails ruff, ty,
  or pytest — no exceptions.
---

# Hooks Skill

## Purpose

Claude Code hooks fire on tool events and can block execution. This skill
installs a single `PreToolUse` hook that intercepts every `git commit` and
runs the full quality gate. If any check fails, the commit is refused and
Claude Code reports the failure. No bad code reaches the repository.

---

## Installation (at project init time)

Create `.claude/` if it does not exist:

```bash
mkdir -p .claude
```

Write `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/pre-commit-gate.sh \"$CLAUDE_TOOL_INPUT\""
          }
        ]
      }
    ]
  }
}
```

Write `.claude/hooks/pre-commit-gate.sh`:

```bash
#!/usr/bin/env bash
# Pre-commit gate — hard blocks git commit if checks fail.
# Invoked by Claude Code PreToolUse hook on every Bash tool call.
# Only activates when the command contains "git commit".
# Detects Python (uv/pyproject.toml) and Rust (Cargo.toml) projects automatically.

set -euo pipefail

TOOL_INPUT="$1"

# Only intercept git commit calls
if ! echo "$TOOL_INPUT" | grep -q "git commit"; then
  exit 0
fi

echo "=== Pre-commit gate ==="
FAILED=0

# ── Python checks (if pyproject.toml present) ─────────────────────────────────
if [ -f "pyproject.toml" ]; then
  echo "--- ruff check ---"
  if ! uv run ruff check . 2>&1; then
    echo "FAIL: ruff check"
    FAILED=1
  fi

  echo "--- ruff format ---"
  if ! uv run ruff format --check . 2>&1; then
    echo "FAIL: ruff format (run 'uv run ruff format .' to fix)"
    FAILED=1
  fi

  echo "--- ty check ---"
  if ! uv run ty check 2>&1; then
    echo "FAIL: ty check"
    FAILED=1
  fi

  echo "--- pytest ---"
  if ! uv run pytest --tb=short -q 2>&1; then
    echo "FAIL: pytest"
    FAILED=1
  fi
fi

# ── Rust checks (if Cargo.toml present) ───────────────────────────────────────
if [ -f "Cargo.toml" ]; then
  echo "--- cargo fmt ---"
  if ! cargo fmt --check 2>&1; then
    echo "FAIL: cargo fmt (run 'cargo fmt' to fix)"
    FAILED=1
  fi

  echo "--- cargo clippy ---"
  if ! cargo clippy -- -D warnings 2>&1; then
    echo "FAIL: cargo clippy"
    FAILED=1
  fi

  echo "--- cargo test ---"
  if ! cargo test 2>&1; then
    echo "FAIL: cargo test"
    FAILED=1
  fi
fi

# ── notes/ guard ──────────────────────────────────────────────────────────────
echo "--- notes/ check ---"
if git diff --cached --name-only | grep -q "^notes/"; then
  echo "FAIL: notes/ files are staged for commit."
  echo "notes/ is a private working directory and must not be committed."
  echo "Run: git restore --staged notes/"
  FAILED=1
fi

# ── Branch guard ────────────────────────────────────────────────────────────────
echo "--- branch check ---"
CURRENT_BRANCH=$(git branch --show-current)
if [ "$CURRENT_BRANCH" = "main" ]; then
  echo "FAIL: commits to main are blocked."
  echo "Create a feature branch: git checkout -b issue-<N>-<description>"
  FAILED=1
fi

if [ "$FAILED" -eq 1 ]; then
  echo ""
  echo "=== COMMIT BLOCKED ==="
  echo "Fix all failures above before committing."
  exit 1
fi

echo ""
echo "=== All checks passed — commit allowed ==="
exit 0
```

Make the hook executable:

```bash
chmod +x .claude/hooks/pre-commit-gate.sh
```

---

## Behaviour

| Situation | Result |
|---|---|
| All checks pass | Commit proceeds normally |
| `git commit` on `main` branch | Commit blocked; branch instruction shown |
| ruff check fails | Commit blocked; ruff output shown |
| ruff format fails | Commit blocked; fix command shown |
| ty check fails | Commit blocked; type errors shown |
| pytest fails | Commit blocked; traceback shown |
| Bash command is not `git commit` | Hook exits immediately; no checks run |

The hook is a hard block. There is no bypass flag, no `--no-verify` equivalent
through this hook, and no warning-only mode. If a test suite is intentionally
incomplete for a given task, the test files must still pass (i.e., existing
tests must not be broken). Write tests before committing.

---

## Interaction with pre-commit

This hook and the `pre-commit` framework (configured in `.pre-commit-config.yaml`
by `research-project-init`) are complementary:

- **pre-commit** runs ruff and ty on `git commit` via the Git hook mechanism.
  This catches issues when committing outside Claude Code.
- **Claude Code hook** (`PreToolUse`) intercepts Claude's own Bash calls before
  they execute. This catches issues when Claude Code is doing the committing.

Both layers are active. The Claude Code hook is the primary gate during agentic
sessions; pre-commit is the fallback for manual commits.

---

## Updating the gate

To add or remove checks from the gate, edit `.claude/hooks/pre-commit-gate.sh`
directly. The structure is: run check, set `FAILED=1` on failure, report at the
end. Do not remove the pytest block without explicit instruction from the human.

---

## Updating the gate

To add or remove checks from the gate, edit `.claude/hooks/pre-commit-gate.sh`
directly. The structure is: detect language via manifest file presence, run checks,
set `FAILED=1` on failure, report at the end. Do not remove the pytest or cargo test
blocks without explicit instruction from the human.
