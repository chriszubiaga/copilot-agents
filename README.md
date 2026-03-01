[README.md](https://github.com/user-attachments/files/25659361/README.md)
# Copilot Agent Template

A ready-to-clone workspace template that wires up a full multi-agent development pipeline in GitHub Copilot. Drop the `.github/` folder into any project and get a coordinated set of specialist agents that take a task from first keystroke to committed, documented code.

## Requirements

- VS Code **1.106+**
- GitHub Copilot extension (Chat)

## Repository structure

```bash
.github/
├── agents/                    # Custom agent definitions
│   ├── orchestrator.agent.md  # ← start here for any task
│   ├── planner.agent.md       # clarification gate + implementation plan
│   ├── coder-jr.agent.md      # quick fixes, simple features, utilities
│   ├── coder-sr.agent.md      # complex architecture, integrations, security
│   ├── coder.agent.md         # general-purpose coding (all-rounder)
│   ├── reviewer.agent.md
│   ├── debugger.agent.md
│   ├── documentor.agent.md
│   └── git.agent.md
├── skills/                    # Reusable domain knowledge loaded by agents
│   ├── bash/SKILL.md
│   ├── powershell/SKILL.md
│   ├── python/SKILL.md
│   ├── git-commit/SKILL.md
│   └── git-worktree/SKILL.md  # parallel working trees for isolated execution
└── copilot-instructions.md    # Workspace-level defaults applied to every session
```

## Pipeline overview

Every task flows through the same pipeline. The **orchestrator** drives it automatically by invoking each worker as a **subagent** — each worker runs in an isolated context window, does its focused job, and returns results to the orchestrator.

```bash
orchestrator
     │
     ▼  Step 0: Clarify & Plan
  planner  (mandatory first step — clarifies intent, returns structured plan)
     │
     ▼  Step 1: Build Execution Plan (parallel phases, file-scoped)
     │
     ▼  Step 2: Start Coding (assigned per complexity)
 coder-jr ◀────────────────── simple tasks, quick fixes
 coder-sr ◀────────────────── complex tasks, architecture, security
     │                        │
     │  escalate if needed ───┘  (coder-jr → coder-sr)
     │
     ▼  Step 3: Review Code
  reviewer  ◀──────────────────────────────────────────┐
     │                                        Re-review │
     │  blockers found                                  │
     └──────────────▶  debugger  ──── fix ─────────────┘
     │  (no blockers)
     ▼  Step 4: Update Docs (if interfaces/behavior changed)
 documentor
     │
     ▼  Step 5: Commit Changes
    git
     │
     ▼
   Done ✓
```

**Planner is mandatory:** the orchestrator always calls the planner first. The planner will ask the user clarifying questions if needed, then produce a file-level implementation plan the orchestrator uses to parallelize work.

**Coder tier:** `coder-jr` handles simple work (quick fixes, utilities, minor changes); `coder-sr` handles complex work (architecture, integrations, performance, security). The orchestrator starts with `coder-jr` and escalates when needed.

**Git Worktree (conditional):** when parallel tasks must modify the same files, the orchestrator uses git worktrees for isolation. See `.github/skills/git-worktree/SKILL.md` for the full protocol.

## Environment notes

This template is tuned for **WSL (Windows Subsystem for Linux)** with a Windows PowerShell bridge:

```bash
# Run PowerShell scripts from WSL
powershell.exe -ep bypass <script.ps1>
```

- Default shell: **Bash** (Linux)
- Python: available in WSL — confirm version with `python3 --version`
- `pwsh` / `powershell` are **not** assumed as native Linux binaries

Adjust `.github/copilot-instructions.md` if your environment differs (pure Linux, macOS, etc.).

## Using this as a project base

1. Copy the `.github/` folder into your project root.
2. Edit `.github/copilot-instructions.md` to reflect your project's stack, OS, and coding conventions.
3. Add or remove skill files under `.github/skills/` as needed.
4. Add or remove agents under `.github/agents/` as needed.
5. Commit and push — agents are workspace-scoped, so anyone who clones the repo gets the same pipeline.

## Extending the template

### Add a new skill

1. Create `.github/skills/<name>/SKILL.md`.
2. Write the domain knowledge as plain Markdown (no frontmatter).
3. Register it in `.github/copilot-instructions.md` under the `## Skills` table.
4. In the relevant agent's body, instruct it to load the skill file when appropriate.

### Add a new agent

1. Create `.github/agents/<name>.agent.md` with valid frontmatter (`name`, `description`, `tools`).
2. Add its name to `orchestrator`'s `agents:` frontmatter list so the orchestrator is allowed to invoke it as a subagent.
3. In the orchestrator's body, describe when and how to invoke the new agent.
4. If the agent should only run as a subagent and not appear in the dropdown, add `user-invokable: false` to its frontmatter.

### Change the review/debug loop

The orchestrator uses a review → debug → re-review loop. To change iteration behavior, edit the **Step 3 — Review** and **Step 4 — Debug Loop** sections in [`.github/agents/orchestrator.agent.md`](.github/agents/orchestrator.agent.md).
