---
name: git-commit
description: Iterative commit-then-verify workflow for making changes. Use when writing code, creating tests, or making any modifications that need user verification. Defines the commit loop, message format, and rules for handling failures.
---

# Git Commit Protocol

You are making changes to a codebase in collaboration with a human who verifies each step. Follow this protocol whenever you are writing code, creating tests, or making any modifications.

## Before Committing

1. Run `git status` and `git diff` to understand the full scope of changes.
2. Assess whether the changes represent **one logical unit** or **multiple unrelated units**.
3. If the changes should be split, propose the grouping to the human for approval before committing.
4. For each commit (whether one or several), follow the commit protocol below for message format and the verify loop.

## Commit Planning

When a plan or task involves multiple changes, propose a commit breakdown **before writing any code**. Separate concerns by type:

1. **Core fix or feature** — the minimal change that addresses the issue
2. **Cleanup** — stale references, renames, or follow-on changes enabled by the core fix
3. **Tests** — test coverage for the new behavior

Present the proposed breakdown to the human for approval. Don't default to grouping everything into one commit — err toward isolating the core change so it's independently reviewable and revertable.

## Commit-Then-Verify Loop

For each unit of work (a test file, a config change, a component modification):

1. **Make the change** — implementation, tests, or both, as dictated by the development workflow.
2. **Commit** using the format below.
3. **Ask** the human to run the relevant verification command (e.g., `cd dashboard && pnpm test`).
4. **If verification passes** — move on to the next step.
5. **If verification fails** — fix the issue, make a **new** commit (never amend), and ask the human to re-run.
6. **Repeat** until the human confirms the step is complete.

Do not batch multiple unrelated changes into a single commit. Each commit should represent one logical unit of work that can be independently verified.

## Commit Message Format

```
<Imperative verb> <short description>

- <what changed>
- <why or technical detail>
- <additional context if needed>
```

### Rules

- **Imperative mood** in the subject line: "Add", "Fix", "Update", "Remove" — not "Added", "Fixes", "Updated"
- **Subject line under 72 characters**
- **Do NOT mention tools** (Figma, Cursor, etc.) in commit titles — OK to reference in the body for context
- **Body uses hyphens** for bullet points
- **Explain both what and why** in the body — the subject says what, the body says why
- **Never amend** a commit after asking the human to verify — if a fix is needed, make a new commit
- **Never force push** unless the human explicitly asks (e.g., after a rebase)
- **No AI co-authoring statements** — do not add "Co-authored by" or similar attribution to any AI agent

### Examples

```
Add component tests for LoginEmailPasswordForm

- Test form rendering, input labels, autoFocus, and submit button text
- Verify onFinish callback receives correct form data on submit
- Verify onBack fires when "sign in with SSO instead" is clicked
- Covers loading state on submit button
```

```
Fix userEvent calls to use v12 API

- Project uses @testing-library/user-event v12 which lacks .setup()
- Switch to direct userEvent.type() and userEvent.click() calls
- Matches the pattern used in existing tests (e.g., BaseSidebar.test.tsx)
```

```
Skip E2E tests that depend on unreleased app logic

- Skip invalid credentials test: Login.tsx lacks error display UI
- Skip logout test: needs data-testid="user-menu-trigger" from BaseSidebar changes
- Skip org selector test: needs data-testid="org-selector-trigger"
- 3 login tests pass (valid login, invite code, base domain redirect)
- Each skip has a comment explaining what to un-skip when
```
