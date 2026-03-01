---
name: git-worktree
description: 'Manage Git worktrees for parallel, isolated branch-level workflows.'
---

# Git Worktree

**Domain:** Git Worktree — parallel working trees for isolated branch-level execution, debugging, and safe experimentation.

---

## Overview

Git worktree attaches multiple working trees to a single repository. Each worktree operates on its own branch independently, enabling true parallel development without stashing, switching, or risking merge conflicts in the working directory.

In this multi-agent workflow, worktrees are used **conditionally** — only when the standard file-ownership parallelization strategy is insufficient.

---

## When to Use This Skill

Use worktrees when:
- The Orchestrator needs to run parallel tasks that must modify the **same files** as independent features
- The Debugger needs an isolated reproduction environment without disturbing ongoing work
- CoderSr is performing a high-risk refactoring that needs rollback safety
- Multiple independent features are being developed simultaneously and share core files

## When NOT to Use

- Tasks touch non-overlapping files (standard file-ownership is sufficient)
- Work is purely sequential
- Single-feature, single-branch workflows
- Simple bug fixes or minor changes

---

## 1. Core Commands Reference

### Create a Worktree
```bash
# Create a new worktree with a new branch
git worktree add <path> -b <new-branch-name> [<start-point>]

# Example: create a worktree for feature work
git worktree add ../<project>-wt-feature-auth -b wt/feature-auth main

# Create a worktree on an existing branch
git worktree add <path> <existing-branch>
```

### List Worktrees
```bash
git worktree list
# Output:
# /path/to/main-repo         abc1234 [main]
# /path/to/project-wt-auth   def5678 [wt/feature-auth]
```

### Remove a Worktree
```bash
# Remove after work is done
git worktree remove <path>

# Force remove (if worktree has uncommitted changes)
git worktree remove --force <path>
```

### Prune Stale Worktrees
```bash
# Clean up references to manually deleted worktrees
git worktree prune
```

---

## 2. Worktree Lifecycle (Orchestrator-Managed)

The **Orchestrator is the sole owner** of worktree lifecycle. Coding agents work within worktrees but do NOT create or remove them.

### Phase 1: Create
Orchestrator creates the worktree before delegating:
```bash
# Convention: sibling directory, descriptive name
git worktree add ../<project>-wt-<purpose> -b wt/<purpose> <base-branch>
```

**Naming conventions:**
- Branch: `wt/<purpose>` (e.g., `wt/feature-auth`, `wt/debug-login-crash`, `wt/refactor-api-layer`)
- Path: sibling directory `../<project>-wt-<purpose>`

### Phase 2: Delegate
Orchestrator delegates the task to the coding agent with:
- The worktree path as the working directory
- The branch name for reference
- Clear scope of what to implement

### Phase 3: Agent Works & Commits
The coding agent:
1. Works exclusively within the delegated worktree path
2. Commits all changes before returning control
3. Does NOT push, merge, or modify other worktrees

### Phase 4: Merge
After the agent completes and Reviewer approves:
```bash
# Orchestrator merges from main worktree
git merge wt/<purpose>

# Or cherry-pick specific commits
git cherry-pick <commit-hash>
```

### Phase 5: Cleanup (MANDATORY)
```bash
git worktree remove ../<project>-wt-<purpose>
git branch -d wt/<purpose>
```

> **CRITICAL:** Never leave worktrees dangling. Every created worktree must be removed after its purpose is fulfilled.

---

## 3. Common Pitfalls

### Locked Worktrees
```bash
git worktree unlock <path>
git worktree remove <path>
```

### Shared Refs
All worktrees share the same `.git` directory:
- Refs are shared — branch names must be unique across all worktrees
- A branch can only be checked out in ONE worktree at a time
- Stash is shared across worktrees

### Dependencies
Each worktree has its own working tree — install dependencies independently:
```bash
cd <worktree-path>
npm install    # or pip install -r requirements.txt, etc.
```

### Submodules
If the project uses submodules, initialize them in each worktree:
```bash
cd <worktree-path>
git submodule update --init --recursive
```

---

## 4. Patterns for Multi-Agent Use

### Pattern A: Parallel Feature Development
When two independent features must modify the same files (e.g., both touch `app.py`):
```
Main worktree:      stays on main branch (Orchestrator control)
Worktree A:         wt/feature-auth      → CoderSr works on auth
Worktree B:         wt/feature-dashboard → CoderSr works on dashboard
```
After both complete → Orchestrator merges sequentially, resolving conflicts if any.

### Pattern B: Isolated Bug Reproduction
Debugger needs a clean tree to reproduce without in-progress changes:
```
Main worktree:      feature work in progress
Worktree debug:     wt/debug-issue-42  → Debugger reproduces and fixes
```
After fix → merge back, remove worktree.

### Pattern C: Safe Refactoring
High-risk structural change that might need to be rolled back:
```
Main worktree:      stable state preserved
Worktree refactor:  wt/refactor-api-v2 → CoderSr performs refactoring
```
If refactoring passes review → merge. If it fails → remove the worktree, no damage done.

---

## 5. Cleanup Checklist

After every worktree session, the Orchestrator must verify:

- [ ] All changes committed in worktree
- [ ] Changes merged or cherry-picked to target branch
- [ ] Worktree removed (`git worktree remove`)
- [ ] Worktree branch deleted (`git branch -d`)
- [ ] No stale entries (`git worktree prune`)
