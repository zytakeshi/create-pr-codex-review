# create-pr-codex-review

[English](README.md) | [日本語](README.ja.md) | **[中文](README.zh.md)**

---

一个 Claude Code 技能，利用 **OpenAI Codex** 作为 AI 代码审查员，自动化完整的 PR 生命周期。

一条命令搞定：提交、推送、创建 PR、等待 Codex 审查、修复问题、合并。

## 安装

```bash
npx skills add zytakeshi/create-pr-codex-review
```

全局安装：

```bash
npx skills add zytakeshi/create-pr-codex-review -g
```

## 使用方法

在 Claude Code 中输入：

```
/create-pr                          # 提交、推送、创建 PR、等待审查
/create-pr and merge                # 同上 + 审查通过后自动合并
/create-pr fix: update auth service # 附带自定义提交信息
```

## 前提条件

- 已安装 [Claude Code](https://claude.ai/code)
- 已安装并认证 [GitHub CLI](https://cli.github.com/) (`gh`)
- 仓库已启用 [Codex 审查](#启用-codex-审查)

## 工作原理

```mermaid
sequenceDiagram
    actor You as 你
    participant Claude as Claude Code
    participant GH as GitHub
    participant Codex as Codex

    You->>Claude: /create-pr
    Claude->>GH: git commit + push
    Claude->>GH: gh pr create
    GH->>Codex: webhook 触发
    Claude-->>You: 「PR 已创建，等待审查中...」

    Note over You: 你可以继续工作

    loop 每 60 秒轮询（最多 20 分钟）
        Claude->>GH: 检查评论和审查
    end

    Codex->>GH: 审查评论
    GH->>Claude: 发现评论！
    Claude->>Claude: 阅读代码 → 修复问题
    Claude-->>You: 「Codex 提了 2 条建议，已修复」
    Claude->>GH: push 修复

    GH->>Codex: 再次审查
    Codex->>GH: 全部通过
    GH->>Claude: 无新评论

    Claude-->>You: 「全部通过，是否合并？」
    You->>Claude: y
    Claude->>GH: gh pr merge --squash
    Claude-->>You: 「已合并！」
```

### 分阶段流程

```mermaid
flowchart TD
    A["Phase 1: 准备提交
    git status / diff
    分析改动内容
    生成 commit message
    git add + commit"] --> B

    B["Phase 2: 推送 + 创建 PR
    git push -u origin
    gh pr create
    生成标题 + 描述"] --> C

    C["Phase 3: 等待审查
    ⏳ 后台运行，不阻塞你
    每 60 秒轮询:
    • 行内评论
    • PR 审查意见
    • CI 检查结果
    超时: 20 分钟"] --> D

    D["Phase 4: 评估 + 修复"] --> E{同意评论？}
    E -->|是| F[修复代码]
    E -->|否| G[跳过 — 告知原因]
    F --> H[lint + commit + push]
    G --> H

    H --> I{有新评论？}
    I -->|是| C
    I -->|没有| J["Phase 5: 合并
    向你确认
    gh pr merge --squash
    验证合并状态"]
```

## 启用 Codex 审查

1. 打开 [chatgpt.com/codex](https://chatgpt.com/codex)，连接你的 GitHub 账号
2. 打开 [chatgpt.com/codex/settings/code-review](https://chatgpt.com/codex/settings/code-review)
3. 为目标仓库开启 **Code review**
4. 开启 **Automatic reviews**，新 PR 创建时自动触发审查

启用后，每次创建 PR，Codex 会自动以行内评论的形式进行代码审查。

**手动触发：** 在 PR 评论中输入 `@codex review`
**重点审查：** 在 PR 评论中输入 `@codex review for security regressions`
**让 Codex 修复：** 在 PR 评论中输入 `@codex address that feedback`

**提示：** 在仓库根目录添加 `AGENTS.md` 文件可以自定义审查标准。

> 需要 ChatGPT Plus、Pro、Team 或 Enterprise 订阅。

## 常见问题

**Q: 支持哪些仓库？**
A: 任何你有 push 权限的 GitHub 仓库。

**Q: 可以自定义 commit message 风格吗？**
A: 可以。技能会分析 `git log`，自动遵循你现有的提交规范。

## 许可证

MIT
