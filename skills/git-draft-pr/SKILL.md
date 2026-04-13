---
name: git-draft-pr
description: Create or update a GitHub pull request. Use when running gh pr create, gh pr edit, or when the user asks to create or update a PR.
---

# Draft PR

Create or update a GitHub pull request with structured formatting, scope detection, and human approval gates.

**Scope:** This is a **git workflow** — create a new PR from branch commits, or update an existing PR with new content while preserving hand edits made on GitHub.

## Entry: Parse Input

- If the user's message contains a PR number, go to **Update Flow**
- If no number is present, ask: "Create a new PR, or enter a PR number to update an existing one?"
- Wait for the user's response before proceeding

## Create Flow

### Preflight

1. Confirm the current branch is not `main` (or `master`).
2. Confirm `git status` is clean — no uncommitted changes.
3. Confirm the branch has commits ahead of `main` (`git rev-list --count main..HEAD`).
4. If any check fails, stop and tell me what to fix.

### Check for Existing PR

Run `gh pr list --head <current-branch> --json number,title,url`. If a PR already exists for this branch, show it and ask: "A PR already exists for this branch (#N). Update it instead?"

If the user says yes, switch to the **Update Flow** with that number.

### Scope Detection

Before building the PR body, determine which commits are in scope for this PR:

1. Gather authored first-parent commits (skips commits merged in from other branches):

   ```bash
   git log --first-parent --author="<user-email>" --no-merges --oneline main..HEAD
   ```

2. From that list, flag likely out-of-scope commits:

   - **Squash merges from other PRs** — message ends with `(#N)`
   - **Cross-ticket commits** — message references a different ticket ID than the branch name (e.g., branch is `DESN-1196` but commit says `DESN-1193`)

3. Present two lists:

   - **In scope** — commits that will feed into the PR body
   - **Excluded** — flagged commits with the reason (squash merge, cross-ticket, etc.)

4. Wait for user confirmation before proceeding. The user may override exclusions.

Only in-scope commits and their file changes feed into the Summary, Notes, and Changes tables.

### Build and Create

1. Collect branch-level diff status for all in-scope files (determines New/Updated/Removed/omitted):

   ```bash
   git diff --name-status main..HEAD -- <in-scope files>
   ```

   Files with no output have zero net diff against main — omit them from the PR body entirely. See the "Determine Change Type" section below for classification rules.

2. Follow the PR body format below to build the title and body from **in-scope commits only**.
3. Present the proposed title and full body to me for review.
4. Wait for my approval before running `gh pr create`.
5. Create with `gh pr create --draft --assignee @me` using a HEREDOC for the body.
6. Show the PR URL when done.

## Update Flow

### Fetch Current State

Run these in parallel:

- `gh pr view <number> --json title,body` — get the existing PR body as written on GitHub
- `git log --first-parent --author="<user-email>" --no-merges --oneline main..HEAD` — get in-scope commits (first-parent only)
- `git log --first-parent --author="<user-email>" --no-merges --name-only --pretty=format: main..HEAD | sort -u` — get in-scope file list
- `git diff --name-status main..HEAD` — get branch-level diff status for file classification (New/Updated/Removed)

Then apply the same scope filtering as in Create Flow → Scope Detection: exclude squash merges from other PRs (`(#N)` suffix) and cross-ticket commits. When adding new content to the PR body, only in-scope commits and files should be used.

### Additive Merge

The existing GitHub PR body is the **source of truth**. Do not regenerate it from scratch. Merge new content in:

1. **Summary bullets** — append new bullets for work not already described. Do not rewrite or remove existing bullets, even if you'd phrase them differently.
2. **Notes** — preserve all existing notes. Append new notes only if there's genuinely new context (e.g., newly skipped tests, new caveats). Never remove a note.
3. **Changes tables** — add new file rows for files not yet listed. Update the description of existing rows only if the file changed substantially since the last update. Do not remove rows unless a file was reverted entirely.
4. **Anything else** — if the body contains content that doesn't match the standard structure (custom sections, inline comments, reviewer instructions), preserve it exactly. The human put it there on purpose.

**When uncertain:** If you can't tell whether a difference between your generated content and the existing body is a hand edit or just stale content, ask me before changing it.

### Present and Apply

1. Show me the existing body and the proposed updated body side by side (or as a clear diff).
2. Wait for my approval.
3. Apply with `gh pr edit <number> --body` using a HEREDOC.
4. Confirm the update was applied.

---

## PR Body Format

The body has four sections in this order. No other sections (e.g., no "Test plan").

### 0. Linear Issue Link

The very first line of the PR body must be the Linear issue URL, formatted as a plain link on its own line followed by a blank line before the summary bullets:

```
https://linear.app/modularml/issue/DESN-1285/sliders-labels-values
```

**How to find it:** Extract the issue identifier from the branch name (e.g., branch `DESN-1285-sliders-labels-values` → issue ID `DESN-1285`). Then use the Linear MCP (`mcp__claude_ai_Linear__get_issue`) to look up the issue by identifier and construct the URL from the response. The URL format is `https://linear.app/modularml/issue/<ID>/<slug>`.

If the branch name does not contain a recognizable issue ID, ask the user for the Linear issue URL.

### 1. Summary (no heading)

Bare bullet list at the top of the body — no `## Summary` heading. Each bullet starts with a plural verb: "Adds", "Updates", "Fixes", "Removes", "Cherry-picks", etc.

Keep bullets concise. Focus on what the PR delivers, not implementation details.

**Ordering is intentional.** Bullets must be ordered by descending user-facing importance:

1. **Primary deliverable first** — the main feature or fix that matches the branch name / ticket. This is the reason the PR exists.
2. **Supporting UX/behavior details** — secondary changes that serve the primary deliverable (e.g., visual polish, layout preservation).
3. **Developer tooling** — debug flags, dev-only helpers, test utilities.
4. **Infrastructure / dependency changes last** — version pins, config fixes, or other technical enablers that made the work possible but aren't the point.

A reviewer scanning only the first bullet should immediately understand what the PR is about.

### 2. Notes

`## Notes` heading. Bullet list of context, caveats, or things reviewers should know — skipped tests and why, cherry-picks that will be dropped on rebase, feature flags, known limitations.

### 3. Changes

`## Changes` heading. One or more file tables grouped by category (e.g., "Component Tests", "E2E Tests", "Config"). Each group gets a `###` subheading.

Tables use this format:

```markdown
| File                  | Change                                       |
| --------------------- | -------------------------------------------- |
| `path/to/file.tsx`    | **New** — brief description                  |
| `path/to/other.tsx`   | **Updated** — what changed                   |
| `path/to/replaced.ts` | **Replaced** — what it was vs what it is now |
```

Use **New**, **Updated**, **Replaced**, or **Removed** as the bold prefix.

### Determine Change Type

After collecting the in-scope file list from commits, cross-reference each file with the branch diff to get its actual status:

```bash
git diff --name-status main..HEAD -- <file>
```

- `A` → candidate for **New** (check authorship below)
- `M` → **Updated**
- `D` → **Removed**
- No output → file changed within the branch but has zero net diff against main — omit it from the tables

A commit can touch a file by deleting it, not just adding or modifying it. The branch-level diff is the ground truth for what the PR actually delivers.

**Check authorship for added files.** A file with branch status `A` may have been introduced by another author via rebase or merge. Check the original author:

```bash
git log --first-parent --diff-filter=A --format='%ae' --reverse main..HEAD -- <file> | head -1
```

If the original author is someone else and the PR author modified it later, mark it **Updated**, not **New**.

### Authorship

Changes tables must only contain files the PR author actually touched **as part of this PR's work**. Other authors' work can land on the branch via rebase or merge — do not take credit for it. The author's own work from other PRs can also land on the branch via merge or cherry-pick — do not include it either.

**Scope tables to in-scope commits only.** Use `--first-parent` in all git log commands to skip commits that entered via merge from other branches:

```bash
git log --first-parent --author="<user-email>" --no-merges --name-only --pretty=format: main..HEAD | sort -u
```

Only files in this output belong in the tables.

**Exclude rebased/merged-in work.** Changes that entered the branch via `git merge main` or rebase do not appear in the GitHub PR diff. Do not include them in summary bullets, notes, or tables. When uncertain, cross-reference with `git log --first-parent --author="<user-email>"` to confirm the work belongs to the PR author.

### Excluding Imported Work

Work from other PRs that was merged, cherry-picked, or squash-merged into this branch is out of scope — even if authored by the PR author. Apply these filters **before** building summary bullets, notes, or file tables:

1. **Use `--first-parent`** in all `git log` commands. This skips commits that entered via `git merge <other-branch>` or `Merge pull request #N`.

2. **Filter squash merges from other PRs.** Commits whose message ends with `(#N)` are GitHub squash merges. Exclude them unless `N` is the current PR's number.

3. **Filter cross-ticket commits.** If the branch name contains a ticket ID (e.g., `DESN-1196`), commits referencing a different ticket ID (e.g., `DESN-1193`, `DESN-1199`) are likely from other work. Flag them for exclusion.

When any of these filters flag commits, present the in-scope and excluded lists to the user for confirmation before proceeding.

### Example

```markdown
https://linear.app/modularml/issue/DESN-1215/add-testing-for-login-flows

- Adds 45 component tests covering all login-related components and SSO flows
- Adds E2E testing infrastructure using Playwright route mocking
- Updates CI workflow with dashboard-e2e-test job

## Notes

- 3 E2E tests are skipped pending app logic from another branch. Each has a comment explaining when to un-skip.
- Component tests follow the `rtl-component-testing` skill conventions
- Cherry-picked modal migration will be dropped automatically on future rebase from main

## Changes

### Component Tests

| File                                             | Change                                  |
| ------------------------------------------------ | --------------------------------------- |
| `src/components/LoginEmailPasswordForm.test.tsx` | **New** — 8 tests                       |
| `src/components/LoginSSOSection.test.tsx`        | **New** — 10 tests                      |
| `src/components/LoginEmailPasswordForm.tsx`      | **Updated** — add data-testid to inputs |

### E2E Tests

| File                   | Change                                                        |
| ---------------------- | ------------------------------------------------------------- |
| `e2e/loginMocks.ts`    | **New** — API route mocks for login tests                     |
| `tests/login.spec.ts`  | **Replaced** — route-mocked tests replacing old skipped tests |
| `playwright.config.ts` | **Updated** — webServer auto-start, project entries           |
```

---

## Execution Rules

- Do not run `gh pr create` or `gh pr edit` without my explicit approval
- Always use `--draft` when creating (unless I explicitly ask to mark it ready)
- Always use `--assignee @me` when creating to self-assign the PR
- Always use a HEREDOC when passing the body to `gh` commands
- No "Test plan" section — ever
- No `## Summary` heading — the summary bullets stand alone at the top
- Plural verbs in summary bullets: "Adds", not "Add"
- No internal tool references — don't mention Figma, design system internals, or other internal tooling in the PR body unless directly relevant to reviewers
- Do not force-push or modify commits — this command only touches the PR metadata
