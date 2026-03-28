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
EOF
)"
```

4. Capture the PR number from the output URL.

## Phase 3: Monitor for Codex Review

> **Note (March 31, 2026):** Codex Code Review now counts toward regular Codex usage limits instead of having a separate allowance. Heavy usage may cause users to hit their overall Codex limit sooner.

This is the critical automation step. After creating the PR, poll GitHub API for review activity from the Codex bot (`chatgpt-codex-connector[bot]`).

### How Codex reviews appear on GitHub

When Codex picks up a PR for review, it reacts with an **eyes emoji** (👀) on the PR. This is the trigger confirmation signal — it means the webhook fired and Codex is processing the PR.

When Codex finishes reviewing, it acts as `chatgpt-codex-connector[bot]`. Two possible outcomes:

1. **Issues found**: Posts a PR review (state: `COMMENTED`) + inline comments on specific lines. The review's `commit_id` field matches the reviewed commit SHA.
2. **No issues found**: May react with a thumbs-up on the PR. That reaction is scoped to the PR, not to a specific commit.

### Polling strategy: two stages

The polling has two stages:

1. **Trigger detection** (fast) — Check for the 👀 eyes reaction OR an already-completed review from Codex (covers the case where Codex finishes before the first poll). If neither is seen after 2 polls, the webhook likely didn't fire — recreate the PR to re-trigger (automatically, but only once).
2. **Review completion** (longer) — Once triggered, poll for the actual review results (review comments or thumbs-up).

### Polling script

Before running the polling script, capture the HEAD commit full SHA:
```bash
HEAD_SHA=$(gh api repos/{owner}/{repo}/pulls/{pr_number} --jq '.head.sha')
```

Run this with `run_in_background: true` so the user isn't blocked:

```bash
CODEX_BOT="chatgpt-codex-connector[bot]"
HEAD_SHA="{head_sha}"

# Get the HEAD commit's push time — used to attribute reactions to this commit.
PUSH_TIME=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/commits \
  --jq 'last | .commit.committer.date' 2>/dev/null)

echo "Waiting for Codex to pick up PR (looking for eyes reaction or completed review)..."

# --- Stage 1: Trigger detection (eyes emoji OR already-completed review) ---
# Also check for an existing review in Stage 1, so a fast Codex turnaround
# (review lands before our first poll) is caught immediately.
# Initial delay before first check
sleep 60

TRIGGERED=false
FAST_PATH=false
for i in $(seq 1 2); do
  EYES=$(gh api repos/{owner}/{repo}/issues/{pr_number}/reactions \
    --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .content == \"eyes\")] | length" \
    2>/dev/null || echo "0")

  # Also check if a HEAD-scoped review already landed (eyes emoji may already be gone).
  # Uses commit_id field (full SHA) — reliable regardless of body markdown formatting.
  EARLY_REVIEW=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
    --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .state != \"PENDING\" and .state != \"DISMISSED\" and .commit_id == \"$HEAD_SHA\")] | length" \
    2>/dev/null || echo "0")

  echo "Trigger check $i/2: eyes_reaction=$EYES existing_review_head=$EARLY_REVIEW"

  if [ "$EARLY_REVIEW" != "0" ]; then
    TRIGGERED=true
    FAST_PATH=true
    echo "CODEX_TRIGGERED"
    echo "Codex review already completed for HEAD commit. Skipping to results..."
    break
  fi

  if [ "$EYES" != "0" ]; then
    TRIGGERED=true
    echo "CODEX_TRIGGERED"
    echo "Codex picked up the PR (eyes reaction detected). Waiting for review results..."
    break
  fi

  [ "$i" -lt 2 ] && sleep 30
done

if [ "$TRIGGERED" = false ]; then
  echo "CODEX_NOT_TRIGGERED"
  exit 0
fi

# --- Fast path: review already landed during Stage 1 ---
if [ "$FAST_PATH" = true ]; then
  echo "CODEX_REVIEW_FOUND"
  echo "---REVIEWS---"
  gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
    --jq ".[] | select(.user.login == \"$CODEX_BOT\" and .commit_id == \"$HEAD_SHA\") | {state, body}" 2>/dev/null
  echo "---COMMENTS---"
  REVIEW_TIME=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
    --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .commit_id == \"$HEAD_SHA\")] | last | .submitted_at" \
    2>/dev/null)
  gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
    --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .created_at >= \"$REVIEW_TIME\")] | .[] | {path, line, body}" 2>/dev/null
  exit 0
fi

# --- Stage 2: Review completion ---
# Now wait for the actual review results
sleep 30

for i in $(seq 1 57); do
  # Check for a review from Codex bot targeting this specific commit
  REVIEWED=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
    --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .state != \"PENDING\" and .commit_id == \"$HEAD_SHA\")] | length" \
    2>/dev/null || echo "0")

  # Check for thumbs-up reaction created AFTER the HEAD commit was pushed.
  REACTION_AFTER_PUSH=$(gh api repos/{owner}/{repo}/issues/{pr_number}/reactions \
    --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .content == \"+1\" and .created_at > \"$PUSH_TIME\")] | length" \
    2>/dev/null || echo "0")

  echo "Poll $i/57: reviewed_head=$REVIEWED reaction_after_push=$REACTION_AFTER_PUSH"

  # Review with findings (commit-scoped via SHA in body)
  if [ "$REVIEWED" != "0" ]; then
    echo "CODEX_REVIEW_FOUND"
    echo "---REVIEWS---"
    gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
      --jq ".[] | select(.user.login == \"$CODEX_BOT\" and .commit_id == \"$HEAD_SHA\") | {state, body}" 2>/dev/null
    echo "---COMMENTS---"
    REVIEW_TIME=$(gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
      --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .commit_id == \"$HEAD_SHA\")] | last | .submitted_at" \
      2>/dev/null)
    gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
      --jq "[.[] | select(.user.login == \"$CODEX_BOT\" and .created_at >= \"$REVIEW_TIME\")] | .[] | {path, line, body}" 2>/dev/null
    exit 0
  fi

  # Clean review (no bugs — thumbs-up reaction created after push)
  if [ "$REACTION_AFTER_PUSH" != "0" ]; then
    echo "CODEX_REVIEW_CLEAN"
    echo "Codex reviewed commit $HEAD_SHA and found no issues (reacted with thumbs up after push)."
    exit 0
  fi

  [ "$i" -lt 57 ] && sleep 30
done

echo "CODEX_REVIEW_TIMEOUT"
```

Replace `{head_sha}` with the actual full SHA captured before launching. The script has two stages:
- **Stage 1**: 60s initial delay, then 2 polls (30s apart) for the 👀 eyes reaction OR an already-completed review (via `commit_id`). Exits with `CODEX_NOT_TRIGGERED` if neither is found.
- **Stage 2**: Once triggered, polls every 30s for up to ~30 minutes for review results.

Detects four outcomes:
- `CODEX_TRIGGERED` → `CODEX_REVIEW_FOUND` — review references HEAD commit, outputs findings (may also fire via Stage 1 fast-path if review landed before polling started)
- `CODEX_TRIGGERED` → `CODEX_REVIEW_CLEAN` — thumbs-up after HEAD push, no bugs
- `CODEX_TRIGGERED` → `CODEX_REVIEW_TIMEOUT` — triggered but no review within the polling window
- `CODEX_NOT_TRIGGERED` — neither eyes reaction nor HEAD-scoped review found, webhook likely didn't fire

Tell the user:
> "PR created: <url>. Waiting for Codex review — I'll notify you when results arrive."

### Handling the result

When the background task completes, read its output:

- **`CODEX_REVIEW_FOUND`**: Parse the `---REVIEWS---` and `---COMMENTS---` sections. Proceed to Phase 4.
- **`CODEX_REVIEW_CLEAN`**: Codex reviewed and found no issues. Tell the user: "Codex reviewed the PR and found no issues." Proceed to Phase 5 (ask to merge).
- **`CODEX_NOT_TRIGGERED`** (first attempt): Codex didn't pick up the PR. Automatically recreate:
  - `gh pr comment {pr_number} --body "Closing to re-trigger Codex review — eyes reaction not detected."`
  - `gh pr close {pr_number}`
  - Re-run `gh pr create` with same branch/title/body
  - Re-enter Phase 3 with the new PR number
  - **This auto-retry happens only once.** If the second PR also gets `CODEX_NOT_TRIGGERED`, ask the user what to do:
    1. **Keep waiting** — user can re-invoke later
    2. **Merge as-is** — skip review, proceed to Phase 5
- **`CODEX_NOT_TRIGGERED`** (second attempt): Do NOT auto-recreate. Ask the user:
  1. **Keep waiting** — user can re-invoke later
  2. **Merge as-is** — skip review, proceed to Phase 5
- **`CODEX_REVIEW_TIMEOUT`**: Codex picked up the PR but didn't finish reviewing. Present options:
  1. **Keep waiting** — user can re-invoke later
  2. **Merge as-is** — skip review, proceed to Phase 5

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
