---
name: git-code-audit
description: "Audit a specific file or directory through git history first, then selectively read hotspot files to explain ownership, change patterns, bugs, and stability risks."
alwaysApply: false
metadata:
  author: Bill Welense
  version: "1.0.0"
  triggers:
    - Tell me who works on this code
    - Audit a path through git history
    - Find churn and bug hotspots in a directory
    - Explain what people have been changing here
    - Save a code audit report
  categories:
    - git-history-analysis
    - code-ownership
    - hotspot-analysis
    - selective-code-reading
---

# git-code-audit: Git-First Code Ownership and Hotspot Analysis

Use this skill when the user wants to understand a specific part of a repository: who works on it, what has changed, where bugs cluster, whether the area shows firefighting signals, and which files should be read first.

This skill is path-focused. Do not audit the whole repository by default. If the user does not specify a file or directory, ask for one.

## Inputs to infer from the request

- `paths` — one or more files or directories to analyze; required
- `time_scope` — optional, such as `--since="6 months ago"`
- `revision_scope` — optional, such as `v2.3.0..HEAD`
- `save` — optional; only save when the user explicitly asks
- `mode` — optional; one of `git-only`, `read-relevant`, or `hotspot-read`

## Default behavior

- Require a path or paths.
- Use git-first analysis before reading code.
- Default to `hotspot-read` mode.
- Return a structured report in chat.
- Save only when explicitly requested.

## Modes

### `git-only`
Run the git analysis and report the findings without reading source files unless the user later asks.

### `read-relevant`
Run the git analysis, then read the most relevant files in the scoped area.

### `hotspot-read` (default)
Run the git analysis, identify the highest-signal hotspot files, then read only a small number of those files.

## Core command recipes

### Churn hotspots

```bash
git log --format=format: --name-only --since="1 year ago" -- <paths> | sort | uniq -c | sort -nr | head -20
```

### Contributors

```bash
git shortlog -sn --no-merges -- <paths>
```

If a revision range is present, prefer:

```bash
git shortlog -sn --no-merges <rev-range> -- <paths>
```

### Bug hotspots

```bash
git log -i -E --grep="fix|bug|broken" --name-only --format='' -- <paths> | sort | uniq -c | sort -nr | head -20
```

### Momentum

```bash
git log --format='%ad' --date=format:'%Y-%m' -- <paths> | sort | uniq -c
```

### Firefighting signals

```bash
git log --oneline --since="1 year ago" -- <paths> | grep -iE 'revert|hotfix|emergency|rollback'
```

## Workflow

### Phase 1: Resolve scope

Interpret the request into `paths`, optional `time_scope`, optional `revision_scope`, optional `save`, and optional `mode`.

If neither `time_scope` nor `revision_scope` is provided:
- use all history for contributor analysis on the scoped paths
- use the last year for churn, bug, and firefighting views

### Phase 2: Run the git analysis

Run the core command recipes scoped to the requested paths. Prefer revision-aware variants when a revision range is provided.

### Phase 3: Correlate signals

Do not dump raw command output without interpretation. Look for:
- files that appear in both churn and bug hotspot lists
- ownership concentration around a single contributor
- differences between historical and recent owners
- repeated crisis terms in the same area
- files that deserve code reading next

### Phase 4: Read code selectively

In `hotspot-read` mode:
- choose the top 3 to 5 overlapping hotspot files
- read those files
- summarize what each one does
- summarize what kinds of edits recur there
- call out signs of fragility, centralization, or broad responsibility

### Phase 5: Report

Return a structured report in chat, and save to disk only if the user asked for it.

## Report structure

Use these sections in the response:

1. **Scope** — analyzed paths, time or revision scope, and read mode
2. **Who works here** — top contributors, concentration notes, historical vs recent ownership if relevant
3. **What changes here** — churn hotspots and common change themes
4. **Where bugs cluster** — bug hotspots, overlap with churn, highest-risk files
5. **Firefighting and stability signals** — matching revert/hotfix/emergency/rollback commits and interpretation
6. **Code reading notes** — what the hotspot files do and what recurring edits suggest
7. **Takeaways** — who likely owns the area, what has changed, what looks risky, what file to read next
8. **Limitations** — caveats that weaken confidence or signal quality

## Save behavior

If the user asks to save the report:

1. Create `.agents/reports/` if needed:
   ```bash
   mkdir -p .agents/reports
   ```
2. Write a timestamped Markdown file named:
   `YYYYMMDD-HHMMSS-<description>.md`
3. Remind the user to add `.agents/` to `.gitignore` if it is not already ignored.

## Limitations and caveats

Mention these when relevant:
- squash merges can distort contributor counts
- vague commit messages weaken bug and firefighting analysis
- rename-heavy histories can weaken simple file-count signals
- path-scoped momentum is suggestive, not definitive
- wide directory scopes can produce noisy hotspot lists
- low-commit areas may not produce enough signal for strong claims
