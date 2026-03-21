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

This is the critical automation step. After creating the PR, poll GitHub API for review activity from the Codex bot (`chatgpt-codex-connector[bot]`).

### How Codex reviews appear on GitHub

When Codex finishes reviewing a PR, it acts as `chatgpt-codex-connector[bot]`. Two possible outcomes may appear on GitHub:

1. **Issues found**: Posts a PR review (state: `COMMENTED`) + inline comments on specific lines. The review body includes `Reviewed commit: \`<sha>\``.
2. **No issues found**: May react with 👍 on the PR. That reaction is scoped to the PR, not to a specific commit.

Because only the review body is commit-scoped, the polling script treats **SHA-matched reviews as the only authoritative signal** that the current HEAD has been reviewed. It still snapshots bot 👍 reactions for timeout diagnostics, but it never treats a new reaction as proof that the latest commit was reviewed.

### Polling script

The script uses **commit-SHA matching**: Codex review bodies contain `Reviewed commit: \`<sha>\``, so we check if the bot has reviewed the HEAD commit specifically. This naturally handles re-reviews after pushing fixes — the script waits for a review that references the latest commit, ignoring reviews of older commits.

Before running the polling script, capture the HEAD commit short SHA:
```bash
HEAD_SHA=$(gh api repos/{owner}/{repo}/pulls/{pr_number} --jq '.head.sha[:10]')
```

Run this with `run_in_background: true` so the user isn't blocked:

```bash
CODEX_BOT="chatgpt-codex-connector[bot]"
HEAD_SHA="{head_sha}"
REVIEW_MARKER=$(printf 'Reviewed commit: `%s`' "$HEAD_SHA")

echo "Waiting for Codex to review commit $HEAD_SHA"

BASELINE_REACTIONS=$(gh api repos/{owner}/{repo}/issues/{pr_number}/reactions \
  --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .content == \"+1\")] | length" \
  2>/dev/null || echo "0")
SAW_NEW_REACTION=0

# Initial delay: Codex typically needs 2-5 minutes to process
sleep 90

for i in $(seq 1 38); do
  # Check for a review from Codex bot that references this specific commit
  REVIEWED=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
    --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .state != \"PENDING\" and ((.body // \"\") | contains(\"$REVIEW_MARKER\")))] | length" \
    2>/dev/null || echo "0")

  # Track new 👍 reactions for diagnostics only.
  # Reactions are PR-scoped, so never use them as proof that HEAD was reviewed.
  CODEX_REACTION=$(gh api repos/{owner}/{repo}/issues/{pr_number}/reactions \
    --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .content == \"+1\")] | length" \
    2>/dev/null || echo "0")

  if [ "$CODEX_REACTION" -gt "$BASELINE_REACTIONS" ]; then
    SAW_NEW_REACTION=1
  fi

  echo "Poll $i/38: reviewed_head=$REVIEWED reactions=$CODEX_REACTION"

  if [ "$REVIEWED" != "0" ]; then
    echo "CODEX_REVIEW_FOUND"
    echo "---REVIEWS---"
    gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
      --jq ".[] | select(.user.login == \"$CODEX_BOT\" and ((.body // \"\") | contains(\"$REVIEW_MARKER\"))) | {state, body}" 2>/dev/null
    echo "---COMMENTS---"
    # Get comments from this review round (submitted at same time as the matching review)
    REVIEW_TIME=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
      --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and ((.body // \"\") | contains(\"$REVIEW_MARKER\")))] | last | .submitted_at" \
      2>/dev/null)
    gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
      --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .created_at >= \"$REVIEW_TIME\")] | .[] | {path, line, body}" 2>/dev/null
    exit 0
  fi

  sleep 30
done

if [ "$SAW_NEW_REACTION" -eq 1 ]; then
  echo "NOTE: Saw a new Codex 👍 reaction after polling started, but reactions are PR-scoped and can't confirm that commit $HEAD_SHA was reviewed."
fi

echo "CODEX_REVIEW_TIMEOUT"
```

Replace `{head_sha}` with the actual short SHA captured before launching. Polls every 30 seconds after a 90-second delay, for up to ~30 minutes. Detects two outcomes:
- `CODEX_REVIEW_FOUND` — review references HEAD commit, outputs findings
- `CODEX_REVIEW_TIMEOUT` — no commit-scoped review within the polling window

Tell the user:
> "PR created: <url>. Waiting for Codex review — I'll notify you when a commit-scoped result arrives."

### Handling the result

When the background task completes, read its output:

- If output contains `CODEX_REVIEW_FOUND`: Parse the `---REVIEWS---` and `---COMMENTS---` sections. Proceed to Phase 4.
- If output contains `CODEX_REVIEW_TIMEOUT`: Codex did not produce a commit-scoped review signal within the polling window. If the output also includes the `NOTE:` line, tell the user a new Codex 👍 reaction appeared but could not be safely attributed to the current HEAD. Then present options:
  1. **Recreate the PR** *(recommended if first attempt)* — close and re-open to re-trigger the webhook
  2. **Keep waiting** — user can re-invoke later
  3. **Merge as-is** — skip review, proceed to Phase 5
- If user chooses option 1:
  - `gh pr comment {pr_number} --body "Closing to re-trigger Codex review."`
  - `gh pr close {pr_number}`
  - Re-run `gh pr create` with same branch/title/body
  - Re-enter Phase 3 with the new PR number
  - **Only offer recreate once.** On second timeout, present only options 2 or 3.

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
- If Codex review times out, follow the timeout handler in Phase 3.
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
