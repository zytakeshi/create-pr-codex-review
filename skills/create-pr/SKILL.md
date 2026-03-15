---
name: create-pr
description: |
  Create a PR from uncommitted changes, then monitor for Codex review comments, verify and fix issues, and merge.
  Triggers: "create pr", "create a pr", "make a pr", "open pr", "submit pr", "push and create pr", "pr for these changes"
  This skill should be used proactively whenever the user asks to create a pull request. It handles the full lifecycle: commit, push, create PR, wait for Codex review, address feedback, and merge.
---

You are automating the full pull request lifecycle. Follow each phase in order.

## User Request

$ARGUMENTS

## Phase 1: Prepare the Commit

1. Run these in parallel:
   - `git status` (never use `-uall`)
   - `git diff` and `git diff --staged` to see all changes
   - `git log --oneline -5` for commit message style
   - `git remote -v` to identify the remote {owner}/{repo}

2. Analyze all changes and draft a commit message:
   - Summarize the nature (feature, fix, refactor, etc.)
   - Keep it concise (1-2 sentence summary line, optional bullet details)
   - Do NOT commit `.env`, credentials, or secrets — warn the user if found

3. Stage specific files (not `git add -A`), commit, verify with `git status`.

## Phase 2: Push and Create PR

1. Create a feature branch if still on `main`/`master` (ask user for branch name if unclear).
2. Push with `-u` flag.
3. Create the PR:

```bash
gh pr create --title "<short title under 70 chars>" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points>

## Changes
<grouped by area/team if multi-file>

## Verification
- lint / analyze results
- test results
- any manual checks done

## Test plan
- [ ] <checklist items>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

4. Capture the PR number from the output URL.

## Phase 3: Monitor for Codex Review

This is the critical automation step. After creating the PR, poll for review activity from Codex.

The polling script checks three sources where Codex results may appear:
1. **Inline comments** (`/pulls/{n}/comments`) — line-level code review
2. **PR reviews** (`/pulls/{n}/reviews`) — approval/request-changes
3. **Check runs** (`gh pr checks`) — Codex may report via check runs

```bash
# Poll every 60 seconds, up to 20 minutes
for i in $(seq 1 20); do
  COMMENT_COUNT=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/comments --jq 'length' 2>/dev/null || echo "0")
  REVIEW_COUNT=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews --jq '[.[] | select(.state != "PENDING")] | length' 2>/dev/null || echo "0")
  CHECKS_DONE=$(gh pr checks {pr_number} 2>/dev/null | grep -cE '(pass|fail)' || echo "0")
  echo "Poll $i: $COMMENT_COUNT comments, $REVIEW_COUNT reviews, $CHECKS_DONE checks completed"
  if [ "$COMMENT_COUNT" != "0" ] || [ "$REVIEW_COUNT" != "0" ] || [ "$CHECKS_DONE" != "0" ]; then
    echo "---COMMENTS---"
    gh api repos/{owner}/{repo}/pulls/{pr_number}/comments --jq '.[] | {path, line, body}' 2>/dev/null
    echo "---REVIEWS---"
    gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews --jq '.[] | {state, body, user: .user.login}' 2>/dev/null
    echo "---CHECKS---"
    gh pr checks {pr_number} 2>/dev/null
    break
  fi
  sleep 60
done
```

Run this with `run_in_background: true` so the user isn't blocked. Tell the user:
> "PR created: <url>. Monitoring for Codex review comments — I'll notify you when they arrive."

## Phase 4: Verify and Fix Review Comments

When comments arrive:

1. Parse each comment — extract the file path, line range, and issue description.
2. For each comment, assess whether you agree:
   - **Agree**: Read the relevant code, implement the fix, and note what was changed.
   - **Disagree**: Explain why to the user and skip (don't fix without user confirmation).
   - **Partially agree**: Explain the nuance, propose an alternative, fix if the user approves.
3. After all fixes:
   - Run the project's lint/analyze command to verify no regressions.
   - Stage only the changed files, create a NEW commit (never amend):
     ```
     fix: address Codex review feedback

     <bullet list of what was fixed and why>
     ```
   - Push to the same branch.

## Phase 5: Merge

After pushing fixes:

1. Confirm with the user: "Review feedback addressed and pushed. Merge now?"
2. If user confirms (or if the original request included "and merge"):
   ```bash
   gh pr merge {pr_number} --squash
   ```
3. Verify merge succeeded:
   ```bash
   gh pr view {pr_number} --json state --jq '.state'
   ```

## Important Rules

- Always use `git remote -v` to verify the correct remote before pushing.
- Never force-push or amend commits.
- Never auto-merge without user confirmation unless they explicitly asked for it in the original request.
- If Codex review has no comments after 20 minutes, tell the user and ask whether to merge or keep waiting.
- If the PR has merge conflicts, notify the user rather than resolving automatically.
- Extract {owner}/{repo} from `git remote -v` output, don't hardcode.
- Parse the PR number from the `gh pr create` output URL.

## Examples

```
/create-pr
/create-pr for all uncommitted changes
/create-pr and merge after codex review
/create-pr fix: update auth service
```
