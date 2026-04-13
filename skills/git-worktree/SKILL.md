---
name: git-worktree
description: Workflow for creating isolated worktrees from a bare repo. Use when starting a new feature branch, or when the agent needs to understand the bare-repo worktree layout it may already be operating inside.
---

# Git Worktree Protocol

This repo uses a **bare repo layout**. There is no working tree at the repo root — all development happens inside worktree subdirectories, each on its own branch.

## Repository Layout

```
<repo>/                              ← repo root (no working tree here)
├── .bare/                           ← bare git repository
├── .git                             ← file: "gitdir: ./.bare"
├── .claude/                         ← NOT git-tracked; repo-root only
├── main/                            ← worktree (branch: main)
│   ├── .git                         ← file: points back to ../.bare
│   ├── .cursor/                     ← git-tracked, part of checkout
│   │   ├── skills/
│   │   └── docs/
│   └── ...                          ← full repo checkout
├── DESN-1179-update-login-page/     ← worktree (branch: DESN-1179-...)
│   ├── .git
│   ├── .cursor/
│   └── ...
└── ...
```

Key points:

- `.cursor/` is **git-tracked** — it lives inside each worktree checkout
- `.claude/` is **not git-tracked** — it lives only at the repo root
- Each worktree folder at the root is a complete, independent checkout
- The repo root has **no working tree** — only `.bare/`, `.git`, `.claude/`, and worktree folders

## Detecting Your Context

Before creating a worktree, determine whether you're at the repo root or inside an existing worktree:

```bash
# Check if current directory is inside a worktree
git rev-parse --is-inside-work-tree   # true = inside a worktree

# Find the path to the shared bare repo
git rev-parse --git-common-dir        # returns path to .bare/
```

- If `--git-common-dir` returns something like `/path/to/repo/.bare`, then the **repo root** is the parent of `.bare/` — i.e., `/path/to/repo/`.
- If you're already inside a worktree (e.g., `cwd` is `/path/to/repo/DESN-1193-.../src/`), navigate up to the repo root before creating new worktrees.

To derive the repo root from anywhere:

```bash
REPO_ROOT="$(git rev-parse --git-common-dir | sed 's|/\.bare$||')"
```

## Already Inside a Worktree

When Cursor opens a worktree subfolder directly, you are already inside a fully checked-out branch. Key things to know:

- `git status`, `git log`, `git diff`, etc. all work normally — no need for `git -C`
- The current branch name matches the worktree folder name
- To create a _new_ worktree, you must navigate to the repo root first (see "Detecting Your Context")
- The `.cursor/` folder here is git-tracked and specific to this checkout

## Creating a New Worktree

There are two flows depending on whether you're creating a **new branch** or checking out an **existing remote branch**.

### Determine the flow

```bash
# Fetch all remote refs first
git fetch origin

# Check if the branch already exists on the remote
git branch -r --list "origin/<branch-name>"
```

- If the command returns a match → use **Flow B: Existing Remote Branch**
- If no match → use **Flow A: New Branch**

### Flow A: New Branch

Use this when starting fresh work. The branch starts at the latest `origin/main` (or a user-specified base), but tracks **its own remote branch** — not the base.

#### Input

- A **branch name**, e.g. `DESN-1193-add-astronaut-video-to-login-page`
- An optional **base branch** to start from (defaults to `origin/main`)

#### Steps

```bash
# 0. Resolve repo root (skip if already there)
REPO_ROOT="$(git rev-parse --git-common-dir | sed 's|/\.bare$||')"
cd "$REPO_ROOT"

# 1. Fetch the base branch
git fetch origin <base-branch>          # e.g. "main" or "skuo/DESN-1256/bug-ux-improvements"

# 2. Create the worktree and branch starting from the base
git worktree add <branch-name> -b <branch-name> origin/<base-branch>

# 3. Push the new branch to origin and set upstream to its own remote branch
git -C <branch-name> push -u origin <branch-name>

# 4. Install dependencies (detect package manager)
if [ -f <branch-name>/pnpm-lock.yaml ]; then
  pnpm -C <branch-name> install
elif [ -f <branch-name>/dashboard/pnpm-lock.yaml ]; then
  pnpm -C <branch-name>/dashboard install
elif [ -f <branch-name>/package-lock.json ]; then
  npm --prefix <branch-name> install
fi
```

**Important:** Step 3 pushes to `origin/<branch-name>` and sets upstream tracking to it. The new branch must track itself on origin, **not** the base branch it was created from.

#### Example (default base)

```bash
REPO_ROOT="$(git rev-parse --git-common-dir | sed 's|/\.bare$||')"
cd "$REPO_ROOT"
git fetch origin main
git worktree add DESN-1193-add-astronaut-video -b DESN-1193-add-astronaut-video origin/main
git -C DESN-1193-add-astronaut-video push -u origin DESN-1193-add-astronaut-video
# then install dependencies (see detect package manager step above)
```

#### Example (custom base branch)

```bash
REPO_ROOT="$(git rev-parse --git-common-dir | sed 's|/\.bare$||')"
cd "$REPO_ROOT"
git fetch origin skuo/DESN-1256/bug-ux-improvements
git worktree add my-fix -b my-fix origin/skuo/DESN-1256/bug-ux-improvements
git -C my-fix push -u origin my-fix
# then install dependencies (see detect package manager step above)
```

### Flow B: Existing Remote Branch

Use this when checking out a branch that already exists on the remote (e.g. a colleague's branch).

#### Input

- A **remote branch name**, e.g. `skuo/DESN-1256/bug-ux-improvements`
- An optional **folder name** (defaults to a sanitized version of the branch name — typically the last path segment or ticket ID, e.g. `DESN-1256-bug-ux-improvements`)

#### Steps

```bash
# 0. Resolve repo root (skip if already there)
REPO_ROOT="$(git rev-parse --git-common-dir | sed 's|/\.bare$||')"
cd "$REPO_ROOT"

# 1. Fetch the branch
git fetch origin <branch-name>

# 2. Create the worktree tracking the remote branch
git worktree add <folder-name> -b <branch-name> origin/<branch-name>

# 3. Set upstream tracking so git pull works
git -C <folder-name> branch --set-upstream-to=origin/<branch-name>

# 4. Install dependencies (detect package manager)
if [ -f <folder-name>/pnpm-lock.yaml ]; then
  pnpm -C <folder-name> install
elif [ -f <folder-name>/dashboard/pnpm-lock.yaml ]; then
  pnpm -C <folder-name>/dashboard install
elif [ -f <folder-name>/package-lock.json ]; then
  npm --prefix <folder-name> install
fi
```

#### Example

```bash
REPO_ROOT="$(git rev-parse --git-common-dir | sed 's|/\.bare$||')"
cd "$REPO_ROOT"
git fetch origin skuo/DESN-1256/bug-ux-improvements
git worktree add DESN-1256-bug-ux-improvements -b skuo/DESN-1256/bug-ux-improvements origin/skuo/DESN-1256/bug-ux-improvements
git -C DESN-1256-bug-ux-improvements branch --set-upstream-to=origin/skuo/DESN-1256/bug-ux-improvements
# then install dependencies (see detect package manager step above)
```

## Verification

After creating the worktree, confirm it is set up correctly:

```bash
# List all worktrees — the new one should appear
git worktree list

# Confirm the branch and latest commit
git -C <folder-name> log -1 --oneline

# Confirm tracking is set (should show upstream branch)
git -C <folder-name> branch -vv
```

## Post-Setup

After creating and verifying the worktree, install dependencies by detecting the package manager:

```bash
# Detect package manager and install
if [ -f <folder-name>/pnpm-lock.yaml ]; then
  pnpm -C <folder-name> install
elif [ -f <folder-name>/dashboard/pnpm-lock.yaml ]; then
  pnpm -C <folder-name>/dashboard install
elif [ -f <folder-name>/package-lock.json ]; then
  npm --prefix <folder-name> install
fi
```

This ensures the worktree is ready for development immediately.

## Cleanup

When a worktree is no longer needed:

```bash
git worktree remove <branch-name>
```

If the branch should also be deleted:

```bash
git branch -d <branch-name>
```

## Rules

- **New branches track themselves, not their base** — for Flow A, push the new branch to origin with `git push -u origin <branch-name>` so it tracks `origin/<branch-name>`. Never set upstream to the base branch (e.g. `origin/main`); the base is only used as the starting point
- **Existing branches track their remote counterpart** — for Flow B, set upstream to `origin/<branch-name>` with `git branch --set-upstream-to`
- **Respect the specified base branch** — if the user names a base branch, use it; only default to `origin/main` when no base is given
- **Check for existing remote branches first** — `git fetch origin` + `git branch -r --list` to decide between Flow A (new) and Flow B (existing)
- **Branch name = folder name** — for new branches use the same string for both; for existing remote branches the folder name may be a shorter alias (e.g. ticket ID)
- **Worktree folders live at the repo root** — as siblings to `.bare/` and `.claude/`, not nested inside other directories
- **Use `git -C <folder>`** to run git commands inside a worktree from the repo root
- **Never create worktrees inside `.claude/`** — that directory is reserved for Claude Code configuration
- **One branch per worktree** — git enforces this; you cannot check out a branch that is already checked out in another worktree
- **Detect context first** — before creating worktrees, check whether you're at the repo root or inside a worktree, and navigate accordingly
