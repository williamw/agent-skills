---
name: git-rebase
description: Interactive rebase-onto-main workflow with conflict resolution. Use when rebasing a feature branch onto main, resolving merge conflicts, or when the user asks to pull in the latest main.
---

# Git Rebase Protocol

You are helping a human rebase their feature branch onto the latest `origin/main`. This protocol walks through the rebase, presents conflicts one at a time, and lets the human choose the resolution.

**Scope:** This is a **git workflow** — fetch the latest main, rebase your commits on top, and walk through any conflicts one at a time with clear recommendations.

## When to Use

- User asks to rebase onto main
- User asks to "pull in latest main"
- User asks for help resolving rebase conflicts
- User is mid-rebase and stuck on conflicts

## Before Starting

1. Confirm the current branch is NOT `main`.
2. Confirm there are no uncommitted changes (`git status` must be clean). If dirty, ask the human to commit or stash first.
3. If either check fails, stop and tell me what to fix before proceeding.
4. Read and follow the `git-commit` skill — follow its commit message format if any fixup commits are needed post-rebase.

## Phase 1: Start the Rebase

```bash
git fetch origin main
git rebase origin/main
```

If the rebase completes cleanly (no conflicts), skip to **Phase 3: Verify**.

If conflicts occur, git will pause. Proceed to **Phase 2**.

## Phase 2: Resolve Conflicts (per-step loop)

A rebase replays your commits one at a time on top of main. Each commit that conflicts produces a separate rebase step. For each step:

### 2a. Identify the conflicting commit

```bash
# Show which commit is being replayed
git log --oneline -1 REBASE_HEAD

# List files with conflicts
git diff --name-only --diff-filter=U
```

Tell the human: "Replaying commit `<hash> <subject>`. The following files have conflicts: ..."

### 2b. Present each conflicting file

For each conflicting file, read the file and find the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`). For each conflict hunk:

1. **Show both sides** clearly labeled:

   - **Main (incoming):** the code from `origin/main` — what landed while you were working
   - **Ours (your branch):** the code from your commit being replayed

2. **Provide context:** Read 10–20 lines above and below the conflict to understand intent. If the conflict involves imports, check what symbols are actually used in the file. If it involves logic changes, explain what each side does differently.

3. **Recommend a resolution** with reasoning. Use the heuristics table below to guide your recommendation. Present one of:

   - **Pick main** — when main has a newer version of something your branch didn't intentionally change
   - **Pick ours** — when your branch intentionally changed this code
   - **Merge both** — when both sides added different things (e.g., both added new imports, both added new test cases)
   - **Rewrite** — when the two sides conflict semantically and need a manual blend

4. **Ask the human to choose.** Never auto-resolve without confirmation. Present it as a structured choice.

### 2c. Apply the resolution

After the human chooses:

1. Edit the file to apply the chosen resolution (remove conflict markers, keep the chosen code).
2. Verify no conflict markers remain in the file.
3. **Read the resolved file in full** (or at minimum the imports and the area around each conflict). Verify:
   - All imports match what the file actually uses — no missing imports, no unused imports
   - No orphaned references to symbols that were removed or renamed in the resolution
   - The file is syntactically complete — no dangling brackets, missing closing tags
4. Stage the file: `git add <file>`
5. If more conflicting files remain in this step, repeat 2b for the next file.
6. When all files in this step are resolved:

```bash
# Verify no remaining conflicts
git diff --name-only --diff-filter=U

# Continue the rebase
git rebase --continue
```

6. If the next commit also has conflicts, loop back to 2a. If the rebase completes, proceed to Phase 3.

## Phase 3: Verify

After the rebase completes:

```bash
# Confirm we're on the feature branch, not detached
git branch --show-current

# Show the branch's commits on top of main
git log --oneline origin/main..HEAD

# Confirm working tree is clean
git status
```

Tell the human the rebase is complete and show them the commit log. Highlight if any commits were dropped or squashed.

### 3a. Pattern sweep

If the branch performed a migration or rename (e.g., replacing `toaster` with `notifications`), grep the full source tree for the old pattern:

```bash
grep -r "OLD_PATTERN" src/ --include="*.ts" --include="*.tsx"
```

Any hits in active code (not comments) are bugs introduced by the rebase bringing in main's code that predates the migration. Fix them with a new commit before proceeding.

### 3b. Run tests

This is mandatory, not optional. Run the project's test suite and report results:

```bash
pnpm test
```

If new failures appeared that weren't present before the rebase, they are rebase artifacts and must be fixed before pushing.

### 3c. Smoke test the app

If the project has a dev server, start it and confirm the app loads without a white screen or console errors:

```bash
pnpm dev:mock  # or pnpm dev
```

This catches runtime crashes from missing imports or undefined references that tests may not cover.

**Note:** Uncommitted changes to tracked files (e.g., dev-only config tweaks in mock routes) are silently overwritten during rebase. If you had local modifications before starting, verify those files after the rebase completes.

## Phase 4: Push

The human must force-push since rebase rewrites history:

```bash
git push --force-with-lease
```

**Never run force-push without the human's explicit approval.** Present the command and wait for confirmation. `--force-with-lease` is safer than `--force` because it refuses to push if someone else pushed to the remote branch since your last fetch.

## Conflict Resolution Heuristics

When recommending a resolution, use these heuristics:

| Scenario                                                                                 | Recommendation                                                                                                                   |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Main updated a dependency version, your branch didn't touch it                           | Pick main                                                                                                                        |
| Both sides added new imports                                                             | Merge both (keep all imports)                                                                                                    |
| Main refactored a function your branch also modified                                     | Read both carefully — usually rewrite to apply your change on top of main's refactored version                                   |
| Both sides modified the same lines                                                       | Stop and ask — this requires human judgment                                                                                      |
| Conflict is in generated files (lock files, etc.)                                        | Regenerate after rebase completes                                                                                                |
| Conflict is in a file your commit didn't intentionally change                            | Pick main                                                                                                                        |
| Branch performed a codebase-wide migration and main added new code using the old pattern | Rewrite — apply the migration to main's new code. After the rebase completes, sweep the full tree for the old pattern (step 3a). |

## Abort

If the rebase gets into a bad state or the human wants to start over:

```bash
git rebase --abort
```

This restores the branch to its pre-rebase state with no side effects. Always mention this option if the human seems uncertain.

## Rules

- **Never auto-resolve conflicts** — always present choices to the human
- **Never force-push without explicit approval** — present the command, wait for confirmation
- **One conflict hunk at a time** — don't overwhelm with all conflicts at once
- **Show context** — conflict markers alone are not enough; read surrounding code to explain intent
- **Verify after each step** — confirm no conflict markers remain before `git rebase --continue`
- **If in doubt, abort** — `git rebase --abort` is always safe
- **Clean working tree required** — refuse to start if there are uncommitted changes
- **Follow commit protocol** — if any fixup commits are needed after rebase, follow `git-commit`
