# git-code-audit design

## Summary

`git-code-audit` is a skill for analyzing a specific part of a repository through git history first, then selectively reading the most relevant code. It answers: who works in this area, what they have been changing, where bugs cluster, what looks unstable, and which files are most worth reading next.

The skill is designed for natural-language requests such as:

- “Audit `src/auth` and tell me who works on it and what they’ve changed.”
- “Look at `app/models` since `v2.3.0`.”
- “Read `server/billing` for the last 6 months and save a report.”
- “Focus on `lib/parser.rb` and tell me who’s been changing it, what keeps breaking, and what code I should read first.”

## Goals

- Focus analysis on one or more user-specified files or directories.
- Support optional time-based or revision-based scoping.
- Use git history as the primary signal before reading code.
- Reuse the audit methods from the referenced blog post, especially the five git command patterns.
- Read only the highest-signal files by default, rather than scanning all scoped code.
- Produce a structured report in chat.
- Optionally save a timestamped Markdown report to `.agents/reports/`.

## Non-goals

- Auditing the entire repository by default when no path is provided.
- Producing an exhaustive authorship model for every file in a large tree.
- Replacing deeper architectural review or blame-level archaeology.
- Acting as a code-modification or refactoring skill.

## User experience

### Expected inputs

The skill should infer these parameters from natural language:

- `paths`: required; one or more files or directories to analyze
- `time_scope`: optional; e.g. “last 6 months”
- `revision_scope`: optional; e.g. `v2.3.0..HEAD`
- `save`: optional; save a Markdown report when explicitly requested
- `mode`: optional; controls whether code is read after git analysis

### Required constraint

If the user does not specify a path or paths, the skill should ask for one instead of analyzing the whole repository by accident.

### Default behavior

- Require path-based scope.
- Accept either a time window or revision range, or neither.
- Run git-first analysis on the scoped area.
- Use hotspot-driven code reading by default.
- Return a structured report in chat.
- Save only when explicitly requested.

### Save behavior

When saving is requested:

- Create `.agents/reports/` if needed.
- Write a Markdown report with a shot-scraper-style timestamped filename.
- Filename pattern: `YYYYMMDD-HHMMSS-<description>.md`
- Remind the user to add `.agents/` to `.gitignore` if it is not already ignored.

## Modes

The skill should support these reading modes:

- `git-only`: analyze git history without reading source files unless the user later asks
- `read-relevant`: after git analysis, read the most relevant scoped files
- `hotspot-read`: after git analysis, read only the top hotspot files identified by the signals

### Default mode

Use `hotspot-read` by default.

This gives the best balance between signal quality and noise. It preserves the git-first method while still grounding the report in actual code for the most important files.

## Analysis workflow

The skill should follow a rigid, repeatable workflow.

### Phase 1: resolve scope

Interpret the user request into:

- `paths`
- optional `time_scope`
- optional `revision_scope`
- optional `save`
- optional `mode`

If neither `time_scope` nor `revision_scope` is provided, use sensible defaults:

- contributor view can use all history for the scoped paths
- churn, bug, and firefighting views should default to the last year

### Phase 2: run scoped git analysis

The skill should embed and adapt the blog-post command patterns for path-scoped analysis.

#### Churn hotspots

```bash
git log --format=format: --name-only --since="1 year ago" -- <paths> | sort | uniq -c | sort -nr | head -20
```

Find the most frequently changed files in the selected area.

#### Contributors

```bash
git shortlog -sn --no-merges -- <paths>
```

If a revision range is provided, prefer a revision-aware form:

```bash
git shortlog -sn --no-merges <rev-range> -- <paths>
```

The skill should compare historical and recent contributors when the request implies recency or when recent-vs-historical ownership concentration is relevant.

#### Bug hotspots

```bash
git log -i -E --grep="fix|bug|broken" --name-only --format='' -- <paths> | sort | uniq -c | sort -nr | head -20
```

Find files that recur in bug-fix commits.

#### Momentum

```bash
git log --format='%ad' --date=format:'%Y-%m' -- <paths> | sort | uniq -c
```

Use this to infer whether activity touching the scoped area is steady, dropping, or bursty. The skill should explicitly note that path-scoped momentum is suggestive rather than definitive.

#### Firefighting signals

```bash
git log --oneline --since="1 year ago" -- <paths> | grep -iE 'revert|hotfix|emergency|rollback'
```

Use this to detect signs of instability or operational stress in the scoped area.

### Phase 3: correlate signals

The skill must interpret results instead of dumping raw command output.

It should specifically look for:

- files that appear in both churn and bug hotspot lists
- areas where one person dominates the contributor list
- differences between historical owners and recent owners
- recurring firefighting terms near the same files or area
- candidate files that should be read next

### Phase 4: selective code reading

In default `hotspot-read` mode:

- select a small number of top overlapping hotspot files, typically 3 to 5
- read those files
- summarize what each file appears to do
- summarize the recurring kinds of changes that history suggests are happening there
- call out signs of fragility, centralization, or surprising breadth of responsibility

The skill should avoid broad codebase reading unless the user explicitly asks for it.

### Phase 5: report

Produce a structured report in chat, and save to disk only when requested.

## Report structure

The report should use these sections.

### 1. Scope

- analyzed paths
- time or revision scope used
- read mode used

### 2. Who works here

- top contributors for the scoped area
- ownership concentration observations
- historical vs recent ownership notes when relevant

### 3. What changes here

- top churn files in scope
- short explanation of what those files likely do
- common themes from recent commits when available

### 4. Where bugs cluster

- bug-fix hotspot files
- overlap with churn hotspots
- explicit highest-risk files callout

### 5. Firefighting and stability signals

- matching revert/hotfix/emergency/rollback commits
- interpretation of stability signal strength

### 6. Code reading notes

For each file read:

- what the file does
- what kinds of edits seem to recur there
- whether the file appears fragile, central, or under concentrated ownership

### 7. Takeaways

A short conclusion covering:

- who likely owns the area
- what has been changing
- what looks risky
- what file should be read first for deeper investigation

### 8. Limitations

The skill should explicitly state limitations when signal quality is weak or interpretation could mislead.

## Natural-language interpretation

The skill should map ordinary prompts into structured parameters.

Examples:

- “read `src/auth`” → `paths=["src/auth"]`
- “since `v1.8.0`” → `revision_scope="v1.8.0..HEAD"`
- “for the last 3 months” → `time_scope='--since="3 months ago"'`
- “save it” → `save=true`
- “history only” → `mode=git-only`
- omitted mode → `mode=hotspot-read`

This lets the user stay conversational instead of learning a command interface.

## Edge cases and caveats

The skill should tell the agent to mention these when relevant:

- squash merges can distort contributor counts
- vague commit messages weaken bug and firefighting analysis
- rename-heavy history can weaken simple file-count signals
- path-scoped momentum can be informative without being conclusive
- wide directory scopes can generate noisy hotspot lists and may need narrowing
- low-commit areas may not produce strong enough signals for confident claims

## Recommended skill structure

The implementation should be a single user-facing skill named `git-code-audit`.

Internally, the skill should be organized into:

1. request parsing and scope resolution
2. scoped git command recipes
3. signal correlation rules
4. selective code-reading rules
5. report formatting and optional save behavior

This keeps the user experience simple while leaving room to reuse the command patterns later in related analysis skills.

## Success criteria

The skill is successful if it can reliably:

- analyze a user-specified path or file without auditing the whole repo by accident
- explain who works in that area
- explain what has been changing there
- highlight where churn, bugs, and firefighting overlap
- read only the most relevant files by default
- produce a report that helps the user know what to inspect next
- optionally save a clean Markdown artifact to `.agents/reports/`
