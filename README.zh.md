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

```
  你                   Claude Code              GitHub               Codex
  │                        │                      │                    │
  │  /create-pr            │                      │                    │
  │───────────────────────>│                      │                    │
  │                        │  git commit + push   │                    │
  │                        │─────────────────────>│                    │
  │                        │  gh pr create        │                    │
  │                        │─────────────────────>│                    │
  │                        │                      │  webhook 触发      │
  │  「PR 已创建，         │                      │───────────────────>│
  │   等待审查中...」      │                      │                    │
  │<───────────────────────│                      │                    │
  │                        │                      │   Codex 审查       │
  │  (你可以继续工作)      │  轮询评论中...       │   你的代码         │
  │                        │─────────────────────>│                    │
  │                        │  (暂无评论)          │                    │
  │                        │─────────────────────>│                    │
  │                        │                      │  审查评论           │
  │                        │  发现评论！          │<───────────────────│
  │                        │<─────────────────────│                    │
  │                        │                      │                    │
  │  「Codex 提了 2 条     │  阅读→修复→push     │                    │
  │   建议，已修复」       │─────────────────────>│                    │
  │<───────────────────────│                      │   再次审查          │
  │                        │  继续轮询...         │───────────────────>│
  │                        │─────────────────────>│                    │
  │                        │  无新评论            │  全部通过           │
  │                        │<─────────────────────│<───────────────────│
  │                        │                      │                    │
  │  「全部通过，          │                      │                    │
  │   是否合并？」         │                      │                    │
  │<───────────────────────│                      │                    │
  │  "y"                   │                      │                    │
  │───────────────────────>│  gh pr merge         │                    │
  │                        │─────────────────────>│                    │
  │  「已合并！」          │                      │                    │
  │<───────────────────────│                      │                    │
```

### 分阶段流程

```
  ┌──────────────────────────────┐
  │  Phase 1: 准备提交           │
  │  - git status / diff         │
  │  - 分析改动内容              │
  │  - 生成 commit message       │
  │  - git add + commit          │
  └──────────────┬───────────────┘
                 │
                 v
  ┌──────────────────────────────┐
  │  Phase 2: 推送 + 创建 PR     │
  │  - git push -u origin        │
  │  - gh pr create              │
  │  - 生成标题 + 描述           │
  └──────────────┬───────────────┘
                 │
                 v
  ┌──────────────────────────────┐
  │  Phase 3: 等待审查           │
  │  (后台运行，不阻塞你)        │
  │                              │
  │  每 60 秒轮询：              │
  │  - PR 行内评论               │
  │  - PR 审查意见               │
  │  - CI 检查结果               │
  │                              │
  │  超时：20 分钟               │
  └──────────────┬───────────────┘
                 │
                 v
  ┌──────────────────────────────┐
  │  Phase 4: 评估 + 修复        │
  │                              │
  │  对每条评论：                │
  │  ┌──────────┐               │
  │  │ 同意？   │               │
  │  └────┬─────┘               │
  │   是  │  否                  │
  │   v   v                     │
  │  修复  跳过（告知原因）      │
  │                              │
  │  完成后：lint + commit + push│
  └──────────────┬───────────────┘
                 │
                 v
          ┌───────────────┐
          │ 有新评论？     │──是──> 回到 Phase 3
          └──────┬────────┘
                 │ 没有
                 v
  ┌──────────────────────────────┐
  │  Phase 5: 合并               │
  │  - 向你确认                  │
  │  - gh pr merge --squash      │
  │  - 验证合并状态              │
  └──────────────────────────────┘
```

## 启用 Codex 审查

1. 打开 [chatgpt.com/codex](https://chatgpt.com/codex)
2. 点击左下角 **Settings**（设置）
3. 在 **GitHub** 部分连接你的 GitHub 账号
4. 点击 **Configure repositories**，选择要启用的仓库
5. 在 **Settings → General** 中开启：
   - **Auto-review pull requests**（自动审查 PR）
   - 触发条件：_Open a pull request for review_（PR 创建时）
6. 保存

启用后，每次创建 PR，Codex 会自动以行内评论的形式进行代码审查。

**手动触发：** 在 PR 评论中输入 `@codex review`
**让 Codex 修复：** 在 PR 评论中输入 `@codex address that feedback`

> 需要 ChatGPT Plus、Pro、Team 或 Enterprise 订阅。

## 常见问题

**Q: 没有 Codex 也能用吗？**
A: 可以。没有 Codex 评论的话，20 分钟后会询问你是否直接合并。

**Q: 支持哪些仓库？**
A: 任何你有 push 权限的 GitHub 仓库。

**Q: 可以自定义 commit message 风格吗？**
A: 可以。技能会分析 `git log`，自动遵循你现有的提交规范。

## 许可证

MIT
