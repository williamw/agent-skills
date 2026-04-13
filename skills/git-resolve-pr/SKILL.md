---
name: git-resolve-pr
description: Update a PR plan document with resolution status, author responses, and git diffs for each review comment. Use when resolving PR review feedback.
---

# Resolve PR

Update an existing PR plan document with resolution status, author responses, and git diffs for each review comment.

## Step 1: Fetch Review Comments via GraphQL

**Important:** You MUST use the GitHub CLI's GraphQL API feature. The REST API endpoints do not return inline review comment bodies.

```bash
gh api graphql -f query='
{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr_number}) {
      title
      reviews(first: 20) {
        nodes {
          author { login }
          body
          state
          submittedAt
          comments(first: 50) {
            nodes {
              body
              path
              line
              createdAt
            }
          }
        }
      }
    }
  }
}'
```

This command requires:

- Network access (use `required_permissions: ["all"]` if in sandbox)
- The user must be authenticated with `gh auth login`

## Step 2: Get Recent Git Commits

Fetch the recent commit history to match commits to review items:

```bash
git log --oneline -15
```

## Step 3: Get Git Diffs for Each Commit

For each relevant commit, get the diff:

```bash
git show {commit_sha} --format=""
```

## Step 4: Update the Plan Document

The plan title should be in the format "PR {number}: {PR title}" (e.g., "PR 191: Add intro video").

For each numbered section, present metadata as a bulleted list with the git diff on a new line after:

### Updated Section Format

```markdown
### N. [Title]

- **File:** `[file path]`
- **Request:** "[Reviewer's original comment]"
- **Response:** [Quote from the author's reply comment, if any]
- **Commit:** `{short_sha}`
- **Summary:** [One-line description of the change made]

\`\`\`diff

- [old line]

* [new line]
  \`\`\`
```

## Guidelines for Git Diffs

**Include enough context to understand the change, but not the full `git show` output.**

- Do NOT include diff headers (`diff --git`, `index`, `---`, `+++`, `@@` lines)
- DO include 1-2 lines of surrounding context when it helps clarify the change
- For multi-line additions, show the full block

**Too verbose (don't do this):**

```diff
diff --git a/file.css b/file.css
index abc123..def456 100644
--- a/file.css
+++ b/file.css
@@ -120,7 +120,7 @@ .selector {
   property: value;
-  margin: 4rem auto;
+  margin: 64px auto;
}
```

**Too minimal (missing context):**

```diff
+  margin-bottom: 24px;
```

**Just right (has context):**

```diff
 @media (max-width: 996px) {
   .content .social-buttons {
+    margin-bottom: 24px;
   }
 }
```

```diff
-  margin: 4rem auto;
+  margin: 64px auto;
```

## Step 5: Update YAML Frontmatter

Change all todo statuses from `pending` to `completed`:

```yaml
todos:
  - id: example-item
    content: Description of the fix
    status: completed # was: pending
```

## Guidelines for Encoding

Use only ASCII-safe characters to ensure compatibility with all systems:

- Use `(none)` instead of em-dash (`--`) for empty responses
- Avoid emoji characters like checkmarks or X marks
- Use standard straight quotes `"` not curly quotes

## Guidelines for Matching Comments to Commits

1. Read the commit message to understand what it addresses
2. Match the file path in the review comment to files changed in the commit
3. If a commit addresses multiple review items, reference it in each relevant section
4. If a review item was addressed by discussion (not code change), note "Resolved via comment" instead of a commit

## Guidelines for Author Responses

1. Look for comments from the PR author that reply to reviewer comments
2. The GraphQL response groups comments by review - author replies appear as separate review nodes
3. Match replies to original comments by file path and timing
