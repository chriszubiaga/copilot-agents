# Workspace Instructions

## General

This workspace contains reusable Copilot agents and skills for software development tasks. Agents are defined in `.github/agents/` and language/domain skills live in `.github/skills/`.

When a request involves writing, reviewing, or running code, use the `orchestrator` agent to coordinate the full workflow. The orchestrator automatically delegates to the appropriate specialist agents (`planner`, `coder-jr`, `coder-sr`, `reviewer`, `debugger`, `documentor`, `git`) as subagents. Worker agents can also be invoked directly from the agent dropdown for focused, single-stage tasks.

## Environment

- **OS:** WSL (Windows Subsystem for Linux) with a Windows PowerShell bridge available.
- **PowerShell execution:** Always invoke PowerShell via the WSL→Windows bridge:
  ```
  powershell.exe -ep bypass <command or script path>
  ```
  Do **not** assume `pwsh` or `powershell` are available as native Linux binaries unless confirmed.
- **Shell default:** Bash (Linux). Bash scripts run natively in WSL.
- **Python:** Available via the WSL environment. Confirm with `python3 --version` before generating version-specific code.

---

## Language & Runtime Targets

| Language   | Minimum version | Notes                                   |
| ---------- | --------------- | --------------------------------------- |
| Python     | 3.11            | Use `python3` for standalone scripts. For multi-file projects, use `uv` — see `.github/skills/uv/SKILL.md`. Add `from __future__ import annotations` for forward-reference compatibility. |
| Bash       | 5.0             | `#!/usr/bin/env bash` + `set -euo pipefail` |
| PowerShell | 7.4 (Core)      | Invoked via `powershell.exe -ep bypass` from WSL |

---

## Formatter & Linter Baseline

| Language   | Formatter  | Linter / Type checker         |
| ---------- | ---------- | ----------------------------- |
| Python     | Black      | Ruff + mypy (or pyright)      |
| Bash       | —          | ShellCheck                    |
| PowerShell | —          | PSScriptAnalyzer              |

Generated code must pass the relevant linter without warnings before being considered complete.

---

## Script Language Decision Guide

Pick the language whose runtime is closest to the target environment:

- **Bash** — simple glue scripts, file manipulation, CI steps running on Linux.
- **PowerShell** — tasks that interact with Windows APIs, COM, or the Windows event log.
- **Python** — anything requiring data structures, HTTP, JSON parsing, or logic > ~40 lines.

When unsure, default to **Python** — it is available in all environments and easiest to test.

---

## Commit Convention

Use **Conventional Commits** for all commits. Invoke the `git` agent to stage and commit — do not run `git commit` directly. See `.github/skills/git-commit/SKILL.md` for message format rules.

---

## CLI Output Standard

All generated scripts must use the unified log format defined in `.github/skills/cli-output/SKILL.md`. Load that skill alongside any language skill when writing scripts or CLI tools.
