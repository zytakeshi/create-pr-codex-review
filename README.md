# create-pr-codex-review

**[English](README.md)** | [日本語](README.ja.md) | [中文](README.zh.md)

---

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

```mermaid
sequenceDiagram
    actor You
    participant Claude as Claude Code
    participant GH as GitHub
    participant Codex as Codex

    You->>Claude: /create-pr
    Claude->>GH: git commit + push
    Claude->>GH: gh pr create
    GH->>Codex: webhook trigger
    Claude-->>You: "PR created, waiting review..."

    Note over You: You keep working

    loop Poll every 60s (up to 20 min)
        Claude->>GH: check comments & reviews
    end

    Codex->>GH: review comments
    GH->>Claude: comments found!
    Claude->>Claude: read code → fix issues
    Claude-->>You: "Codex found 2 issues, fixed"
    Claude->>GH: push fixes

    GH->>Codex: re-review
    Codex->>GH: all clear
    GH->>Claude: no new comments

    Claude-->>You: "All clear. Merge?"
    You->>Claude: y
    Claude->>GH: gh pr merge --squash
    Claude-->>You: "Merged!"
```

### Phase-by-Phase Flow

```mermaid
flowchart TD
    A["Phase 1: Prepare Commit
    git status / diff
    analyze changes
    generate commit message
    git add + commit"] --> B

    B["Phase 2: Push + Create PR
    git push -u origin
    gh pr create
    generate title + body"] --> C

    C["Phase 3: Wait for Review
    ⏳ background, non-blocking
    Poll every 60s:
    • inline comments
    • PR reviews
    • CI check runs
    Timeout: 20 min"] --> D

    D["Phase 4: Assess + Fix"] --> E{Agree with comment?}
    E -->|Yes| F[Fix the code]
    E -->|No| G[Skip — explain why]
    F --> H[lint + commit + push]
    G --> H

    H --> I{New comments?}
    I -->|Yes| C
    I -->|No| J["Phase 5: Merge
    confirm with user
    gh pr merge --squash
    verify merge status"]
```

## Enable Codex Review

1. Go to [chatgpt.com/codex](https://chatgpt.com/codex) and connect your GitHub account
2. Open [chatgpt.com/codex/settings/code-review](https://chatgpt.com/codex/settings/code-review)
3. Toggle on **Code review** for your repository
4. Toggle on **Automatic reviews** to review every new PR automatically

Once enabled, Codex posts a standard GitHub code review with inline comments whenever a PR is opened.

**Manual trigger:** Comment `@codex review` on any PR
**Focused review:** Comment `@codex review for security regressions`
**Let Codex fix:** Comment `@codex address that feedback`

**Tip:** Add review guidelines to `AGENTS.md` at your repo root to customize what Codex looks for.

> Requires ChatGPT Plus, Pro, Team, or Enterprise.

## FAQ

**Q: What repos are supported?**
A: Any GitHub repo you have push access to.

**Q: Can I customize the commit message style?**
A: Yes — the skill analyzes your recent `git log` and follows your existing conventions.

## License

MIT
