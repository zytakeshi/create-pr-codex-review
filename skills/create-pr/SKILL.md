---
name: create-pr
description: |
  Create a PR from uncommitted changes, then monitor for Codex review comments, verify and fix issues, and merge.
  Triggers: "create pr", "create a pr", "make a pr", "open pr", "submit pr", "push and create pr", "pr for these changes"
  This skill should be used proactively whenever the user asks to create a pull request. It handles the full lifecycle: auto-bump version (if not already bumped), commit, push, create PR, wait for Codex review, address feedback, and merge.
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

3. **Auto-bump the version (if not already bumped).** Every PR should carry a version bump unless one is already present in the changes. Detect → decide → bump:

   a. **Locate the version file** at the repo root — first match wins:

      | File | Version field |
      |------|---------------|
      | `pubspec.yaml` | `version: X.Y.Z` or `version: X.Y.Z+BUILD` (Flutter/Dart) |
      | `package.json` | `"version": "X.Y.Z"` |
      | `Cargo.toml` | `version = "X.Y.Z"` under `[package]` |
      | `pyproject.toml` | `version = "X.Y.Z"` under `[project]` or `[tool.poetry]` |
      | `build.gradle` / `build.gradle.kts` | `versionName "X.Y.Z"` (and `versionCode N`) |
      | `VERSION` / `version.txt` | bare `X.Y.Z` |

      If none is found, **skip the bump silently**, note it to the user, and continue — never fail the PR over a missing version file.

   b. **Check whether it's already bumped.** Resolve the PR base branch:
      ```bash
      BASE=$(git rev-parse --abbrev-ref origin/HEAD 2>/dev/null | sed 's@^origin/@@')
      [ -z "$BASE" ] && BASE=$(git remote show origin 2>/dev/null | sed -n 's/.*HEAD branch: //p')
      [ -z "$BASE" ] && BASE=main
      ```
      Compare the working-tree version against the base branch's copy:
      ```bash
      git show "origin/$BASE:<version_file>" 2>/dev/null
      ```
      - If the working-tree version **differs** from the base version → already bumped. **Skip**, and tell the user the existing new version.
      - If `origin/$BASE` isn't available, run `git fetch origin "$BASE"` once. If still unavailable, compare against `HEAD:<version_file>` instead; when genuinely in doubt, **bump** (a duplicate-looking bump is safer than a PR with no bump).

   c. **Decide the bump level** (only if not already bumped):
      - `$ARGUMENTS` overrides everything: `major` / `minor` / `patch` → use that; `no version bump` / `skip version` → skip.
      - Otherwise infer from the step-2 change analysis: breaking change → **major**; new feature (`feat:`) → **minor**; everything else (fix/refactor/chore/docs) → **patch**.
      - For `pubspec.yaml` with a `+BUILD` suffix, always also increment `BUILD` by 1. For `build.gradle`, also increment `versionCode` by 1.

   d. **Apply the bump** by editing only the version line of the detected file with the Edit tool (not `sed -i`). Then re-read that line to confirm the new value.

   e. Report one line: `Version bumped: X.Y.Z → X.Y.Z' (<level>)`, or `Version bump skipped — <reason>`.

   The bumped version file MUST be staged in the same commit as the rest of the changes (next step). The version bump happens once per PR here in Phase 1 — Phase 4 fix rounds must NOT re-bump (the step-3b check already guarantees this).

4. Stage specific files (not `git add -A`) — including the bumped version file — commit, verify with `git status`.

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

When Codex picks up a PR for review, it reacts with an **eyes emoji** (👀) on the PR. This signals the webhook fired and Codex is actively processing. **The eyes emoji is removed once Codex finishes**, so it is only visible while the review is in progress.

When Codex finishes reviewing, it removes the eyes emoji and acts as `chatgpt-codex-connector[bot]`. Two possible outcomes:

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

  # Also check if a HEAD-scoped review already landed (eyes emoji is removed on completion).
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
- **`CODEX_REVIEW_CLEAN`**: Codex reviewed and found no issues. Tell the user: "Codex reviewed the PR and found no issues." Proceed to Phase 5.
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

## Phase 4: Verify and Fix Review Comments (loop)

This phase loops with Phase 3 until Codex is satisfied. Each iteration is called a **round** (round 1, round 2, …).

When comments arrive:

1. Parse each comment — extract the file path, line range, and issue description.
2. **Independently assess each comment** — read the relevant code and evaluate whether the suggestion is correct and improves the code. You are the gatekeeper; do NOT blindly fix everything Codex says. For each comment:
   - **Agree** (the suggestion is valid and beneficial): Implement the fix and note what was changed.
   - **Disagree** (the suggestion is incorrect, unnecessary, or would degrade the code): **Do NOT fix it.** Instead, reply to the comment on the PR explaining why you disagree:
     ```bash
     gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies \
       --method POST -f body="<your reasoning for disagreeing>"
     ```
     This lets Codex see your counterargument in the next review round and either accept it or push back.
   - **Partially agree** (the concern is valid but the suggested fix isn't ideal): Reply to the comment explaining why you're taking a different approach, then implement your better alternative.
3. After processing all comments (fixes + reply-comments):
   - If any code was changed: run the project's lint/analyze command to verify no regressions, stage only the changed files, create a NEW commit (never amend):
     ```
     fix: address Codex review feedback (round N)

     <bullet list of what was fixed and why>
     <bullet list of what was disputed with reasoning>
     ```
   - If no code was changed (all comments disputed): still push the reply comments so Codex can read them, then create a no-op commit with a message like:
     ```
     chore: respond to Codex review (round N) — no code changes

     All findings disputed — see PR comment replies for reasoning.
     ```
   - Push to the same branch.
4. **Loop back to Phase 3** — re-capture the new HEAD SHA and re-enter the polling script to wait for Codex to review the new commit (and read your reply comments). Tell the user:
   > "Round N complete. Waiting for Codex to re-review…"
5. When Phase 3 returns a result for the new commit:
   - **`CODEX_REVIEW_FOUND`**: Start the next round — go to step 1 of this phase.
   - **`CODEX_REVIEW_CLEAN`**: Codex is satisfied. Proceed to Phase 5.
   - **`CODEX_REVIEW_TIMEOUT`**: Tell the user Codex timed out on re-review and offer options (keep waiting / merge as-is).

**Safety cap**: After **5 rounds** of fixes without a clean review, stop looping and ask the user how to proceed (keep going, merge as-is, or abandon). This prevents unbounded iteration.

## Phase 5: Merge

After Codex gives a clean review (or user elects to merge):

1. Confirm with the user: "Codex is satisfied (or: user chose to merge). Merge now?"
2. If user confirms (or if the original request included "and merge"):
   ```bash
   gh pr merge {pr_number} --squash
   ```
3. Verify merge succeeded:
   ```bash
   gh pr view {pr_number} --json state --jq '.state'
   ```
4. Switch back to the base branch and pull:
   ```bash
   BASE_BRANCH=$(gh pr view {pr_number} --json baseRefName --jq '.baseRefName')
   git checkout "$BASE_BRANCH" && git pull
   ```

## Important Rules

- Always use `git remote -v` to verify the correct remote before pushing.
- Never force-push or amend commits.
- Never auto-merge without user confirmation unless they explicitly asked for it in the original request.
- If Codex review times out, follow the timeout handler in Phase 3.
- If the PR has merge conflicts, notify the user rather than resolving automatically.
- Extract {owner}/{repo} from `git remote -v` output, don't hardcode.
- Parse the PR number from the `gh pr create` output URL.
- Auto-bump the version exactly once per PR, in Phase 1, and only if it isn't already bumped vs the base branch. Never bump in Phase 4 fix rounds. Never bump if no version file exists or the user said `no version bump` — skip gracefully, never fail the PR over versioning.
- Bump the version line with the Edit tool, never `sed -i`. Default to a **patch** bump; use **minor** for `feat:`, **major** for breaking changes, or whatever level the user specified in `$ARGUMENTS`.

## Examples

```
/create-pr
/create-pr for all uncommitted changes
/create-pr and merge after codex review
/create-pr fix: update auth service
/create-pr bump minor
/create-pr no version bump
```
