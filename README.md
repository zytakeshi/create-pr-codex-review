# create-pr-codex-review

A Claude Code skill that automates the full pull request lifecycle with **OpenAI Codex** as your AI code reviewer.

One command: commit, push, create PR, wait for Codex review, fix issues, merge.

## Install

```bash
npx skills add zytakeshi/create-pr-codex-review
```

Or install globally:

```bash
npx skills add zytakeshi/create-pr-codex-review -g
```

## Usage

In Claude Code:

```
/create-pr                          # commit, push, create PR, wait for review
/create-pr and merge                # same + auto-merge after review passes
/create-pr fix: update auth service # with a custom commit hint
```

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
- [Codex review](#enable-codex-review) enabled on your repo

## How It Works

```
  You                  Claude Code              GitHub               Codex
  │                        │                      │                    │
  │  /create-pr            │                      │                    │
  │───────────────────────>│                      │                    │
  │                        │  git commit + push   │                    │
  │                        │─────────────────────>│                    │
  │                        │  gh pr create        │                    │
  │                        │─────────────────────>│                    │
  │                        │                      │  webhook           │
  │  "PR created,          │                      │───────────────────>│
  │   waiting review..."   │                      │                    │
  │<───────────────────────│                      │                    │
  │                        │                      │   Codex reviews    │
  │  (you keep working)    │  poll comments...    │   your code        │
  │                        │─────────────────────>│                    │
  │                        │  (nothing yet)       │                    │
  │                        │─────────────────────>│                    │
  │                        │                      │  review comments   │
  │                        │  found comments!     │<───────────────────│
  │                        │<─────────────────────│                    │
  │                        │                      │                    │
  │  "Codex found 2       │  read → fix → push  │                    │
  │   issues, fixed"       │─────────────────────>│                    │
  │<───────────────────────│                      │   re-review        │
  │                        │  poll again...       │───────────────────>│
  │                        │─────────────────────>│                    │
  │                        │  no new comments     │  all clear         │
  │                        │<─────────────────────│<───────────────────│
  │                        │                      │                    │
  │  "All clear. Merge?"   │                      │                    │
  │<───────────────────────│                      │                    │
  │  "y"                   │                      │                    │
  │───────────────────────>│  gh pr merge         │                    │
  │                        │─────────────────────>│                    │
  │  "Merged!"             │                      │                    │
  │<───────────────────────│                      │                    │
```

### Phase-by-Phase Flow

```
  ┌─────────────────────────────┐
  │  Phase 1: Prepare Commit    │
  │  - git status / diff        │
  │  - analyze changes          │
  │  - generate commit message  │
  │  - git add + commit         │
  └──────────────┬──────────────┘
                 │
                 v
  ┌─────────────────────────────┐
  │  Phase 2: Push + Create PR  │
  │  - git push -u origin       │
  │  - gh pr create             │
  │  - generate title + body    │
  └──────────────┬──────────────┘
                 │
                 v
  ┌─────────────────────────────┐
  │  Phase 3: Wait for Review   │
  │  (background, non-blocking) │
  │                             │
  │  Poll every 60s:            │
  │  - PR inline comments       │
  │  - PR reviews               │
  │  - CI check runs            │
  │                             │
  │  Timeout: 20 minutes        │
  └──────────────┬──────────────┘
                 │
                 v
  ┌─────────────────────────────┐
  │  Phase 4: Assess + Fix      │
  │                             │
  │  For each comment:          │
  │  ┌─────────┐               │
  │  │ Agree?  │               │
  │  └────┬────┘               │
  │   yes │  no                │
  │   v   v                    │
  │  fix  skip (explain why)   │
  │                             │
  │  Then: lint + commit + push │
  └──────────────┬──────────────┘
                 │
                 v
          ┌──────────────┐
          │ New comments? │──yes──> back to Phase 3
          └──────┬───────┘
                 │ no
                 v
  ┌─────────────────────────────┐
  │  Phase 5: Merge             │
  │  - confirm with user        │
  │  - gh pr merge --squash     │
  │  - verify merge status      │
  └─────────────────────────────┘
```

## Enable Codex Review

1. Go to [chatgpt.com/codex](https://chatgpt.com/codex)
2. Click **Settings** (bottom-left)
3. Under **GitHub**, connect your GitHub account
4. Click **Configure repositories** and select your repos
5. In **Settings → General**, enable:
   - **Auto-review pull requests**
   - Trigger on: _Open a pull request for review_
6. Save

Once enabled, Codex automatically reviews every PR with inline comments.

**Manual trigger:** Comment `@codex review` on any PR
**Let Codex fix:** Comment `@codex address that feedback`

> Requires ChatGPT Plus, Pro, Team, or Enterprise.

## FAQ

**Q: Does it work without Codex?**
A: Yes. Without Codex comments, it waits 20 minutes then asks if you want to merge directly.

**Q: What repos are supported?**
A: Any GitHub repo you have push access to.

**Q: Can I customize the commit message style?**
A: Yes — the skill analyzes your recent `git log` and follows your existing conventions.

## License

MIT
