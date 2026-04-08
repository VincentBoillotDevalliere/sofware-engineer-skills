---
name: pr-review-fix
description: >
  Use this skill whenever the user wants to address, resolve, or fix PR (pull request) review comments.
  Trigger on phrases like: "fix my PR comments", "address the review feedback", "resolve PR review",
  "fix what the reviewer said", "handle PR comments", "go through my PR review", "fix review feedback",
  "apply suggested changes from PR", or any mention of acting on GitHub PR review comments.
  This skill fetches all unresolved review comments from the open PR on the current branch,
  understands each one in context, and applies the appropriate code fixes directly to the local files.
  Use this aggressively — if the user is in a repo and mentions review comments, PR feedback, or
  reviewer suggestions in any form, this skill should run.
---

# PR Review Fix

This skill fetches all unresolved review comments from the current branch's GitHub PR and automatically applies the fixes to local files.

## Workflow

### 1. Identify the PR

Run this to find the open PR for the current branch:

```bash
gh pr view --json number,title,url,baseRefName,headRefName
```

If no PR is found, tell the user and stop. If found, confirm with: "Found PR #N: [title] — fetching review comments…"

### 2. Fetch all unresolved review comments

Fetch all three comment types in parallel:

**Inline code review comments** (attached to specific lines):
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --paginate \
  --jq '[.[] | select(.in_reply_to_id == null) | {id, path, line, original_line, body, diff_hunk, side, commit_id}]'
```

**Review-level comments** (general feedback per review):
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --paginate \
  --jq '[.[] | select(.state != "APPROVED" and .body != "") | {id, state, body, submitted_at}]'
```

**Suggested changes** (GitHub's inline suggestions, already embedded in inline comments as markdown):
These appear in inline comments whose `body` contains ` ```suggestion ` blocks. Parse them from the inline comments above.

To get `{owner}/{repo}`, run:
```bash
gh repo view --json nameWithOwner --jq '.nameWithOwner'
```

**Filtering resolved comments:**
A thread is resolved if there's a reply in the thread that includes the word "resolved" or if the thread has a resolution event. As a practical heuristic: fetch replies to each comment and skip threads that have a reply from the PR author or where the latest reply seems to be a "thanks, done" acknowledgment.

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --paginate \
  --jq '[.[] | {id, in_reply_to_id, body, user_login: .user.login}]'
```

Group reply threads by `in_reply_to_id`. A thread is likely resolved if the PR author has replied to it. Be conservative — if unsure, treat the comment as unresolved.

### 3. Understand and plan fixes

Before touching any files:

1. **Read each comment in context** — for inline comments, read the file at the referenced path and look at the surrounding lines (use the `diff_hunk` to understand what code the reviewer was looking at).

2. **Group related comments** — multiple comments on the same file or feature should be fixed together.

3. **Categorize each comment:**
   - **Suggested change**: Apply the exact suggestion block if present — this is a mechanical change the reviewer already wrote out.
   - **Clear actionable feedback**: e.g., "rename this variable", "add error handling here", "this function is missing a return type" — these have a definite fix.
   - **Ambiguous/subjective feedback**: e.g., "this feels off", "consider refactoring" — flag these to the user and ask for clarification rather than guessing.

4. **Print a plan** before making changes:
   ```
   Found N unresolved comments. Here's my plan:
   ✓ [path/to/file.ts:42] Apply suggested change (rename `foo` → `bar`)
   ✓ [path/to/file.ts:87] Add null check before accessing `.data`
   ✓ [src/utils.py] General feedback: "consider extracting this into a helper" → will refactor
   ⚠ [src/auth.ts:12] Ambiguous: "this seems wrong" → will skip and flag to you
   ```

   Proceed without waiting for confirmation unless there are ambiguous comments — in that case, ask about those specifically before proceeding.

### 4. Apply the fixes

Work through the plan file by file:

- **For suggested changes**: Replace the exact lines indicated by the diff hunk with the suggestion content.
- **For inline comments**: Read the file, locate the referenced line(s), understand the issue, and make the targeted fix. Don't rewrite unrelated code.
- **For general review comments**: These often apply across the file or PR. Read the relevant file(s), understand the broader concern, and apply the change.

Be surgical — fix what the reviewer asked for, don't make opportunistic refactors or style changes they didn't ask for. The reviewer should see their specific concerns addressed.

If a fix would require changes across many files or is risky/large in scope, flag it to the user before proceeding.

### 5. Report what was done

After all fixes are applied, print a clean summary:

```
Fixed N of M comments:

✅ path/to/file.ts — Renamed `processData` → `processUserData` (line 42)
✅ path/to/file.ts — Added null guard before accessing `.response.data` (line 87)
✅ src/utils.py — Extracted repeated logic into `format_date()` helper
⚠️  src/auth.ts:12 — Skipped: comment was ambiguous ("this seems wrong"). Please review manually.

Files modified: path/to/file.ts, src/utils.py
No changes were committed. Review the diffs with `git diff` before pushing.
```

Always remind the user to review the diffs with `git diff` before committing.

## Tips for reliable fixes

- **Always read the file before editing it** — don't rely solely on the diff hunk, which may be outdated relative to the local working copy.
- **Use the `diff_hunk` for context**, not as ground truth for current line numbers — the working copy may have diverged from the commit the comment was made on.
- **For suggested changes**, the suggestion block format is:
  ````
  ```suggestion
  replacement line(s) here
  ```
  ````
  Replace the lines in the diff hunk's `-` side with the suggestion content.
- **If the referenced file doesn't exist locally** (e.g., it was deleted or renamed), flag it to the user.
- **If you can't figure out where to apply a fix** due to significant divergence between the PR diff and current working tree, say so clearly rather than making a guess that could corrupt the file.
