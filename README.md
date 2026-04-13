# agent-skills

Custom skills for AI coding agents, compatible with the [Agent Skills](https://agentskills.io) open standard.

## Skills

| Skill | Description |
|-------|-------------|
| `composed-method-pattern` | Refactor TypeScript in a Kent Beck style: make code read like a story for humans |
| `git-commit` | Iterative commit-then-verify workflow with message format and failure handling |
| `git-draft-pr` | Create or update a GitHub pull request |
| `git-rebase` | Interactive rebase-onto-main workflow with conflict resolution |
| `git-resolve-pr` | Resolve PR review feedback with status tracking and git diffs |
| `git-worktree` | Create isolated worktrees from a bare repo for feature branches |

## Installation

```bash
npx skills add ./agent-skills
```

Or install specific skills:

```bash
npx skills add ./agent-skills -s git-commit
```
