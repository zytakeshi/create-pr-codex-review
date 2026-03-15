# create-pr-codex-review

[English](README.md) | **[日本語](README.ja.md)** | [中文](README.zh.md)

---

**OpenAI Codex** をAIコードレビュアーとして活用し、PRライフサイクルを完全自動化する Claude Code スキルです。

1コマンドで: コミット、プッシュ、PR作成、Codexレビュー待機、修正、マージ。

## インストール

```bash
npx skills add zytakeshi/create-pr-codex-review
```

グローバルインストール:

```bash
npx skills add zytakeshi/create-pr-codex-review -g
```

## 使い方

Claude Code で入力:

```
/create-pr                          # コミット、プッシュ、PR作成、レビュー待機
/create-pr and merge                # 同上 + レビュー通過後に自動マージ
/create-pr fix: update auth service # カスタムコミットメッセージ付き
```

## 前提条件

- [Claude Code](https://claude.ai/code) インストール済み
- [GitHub CLI](https://cli.github.com/) (`gh`) インストール・認証済み
- リポジトリで [Codex レビュー](#codex-レビューの有効化) を有効化

## 仕組み

```mermaid
sequenceDiagram
    actor You as あなた
    participant Claude as Claude Code
    participant GH as GitHub
    participant Codex as Codex

    You->>Claude: /create-pr
    Claude->>GH: git commit + push
    Claude->>GH: gh pr create
    GH->>Codex: webhook トリガー
    Claude-->>You: 「PR作成完了、レビュー待機中...」

    Note over You: 作業を続行可能

    loop 60秒ごとにポーリング（最大20分）
        Claude->>GH: コメント・レビューを確認
    end

    Codex->>GH: レビューコメント
    GH->>Claude: コメント発見！
    Claude->>Claude: コード読む → 修正
    Claude-->>You: 「Codexが2件の指摘、修正済み」
    Claude->>GH: 修正をpush

    GH->>Codex: 再レビュー
    Codex->>GH: 問題なし
    GH->>Claude: 新しいコメントなし

    Claude-->>You: 「全て通過。マージしますか？」
    You->>Claude: y
    Claude->>GH: gh pr merge --squash
    Claude-->>You: 「マージ完了！」
```

### フェーズ別フロー

```mermaid
flowchart TD
    A["Phase 1: コミット準備
    git status / diff
    変更内容を分析
    コミットメッセージ生成
    git add + commit"] --> B

    B["Phase 2: プッシュ + PR作成
    git push -u origin
    gh pr create
    タイトル + 本文を生成"] --> C

    C["Phase 3: レビュー待機
    ⏳ バックグラウンド・非ブロック
    60秒ごとにポーリング:
    • インラインコメント
    • PR レビュー
    • CI チェック
    タイムアウト: 20分"] --> D

    D["Phase 4: 評価 + 修正"] --> E{コメントに同意？}
    E -->|はい| F[コードを修正]
    E -->|いいえ| G[スキップ — 理由を説明]
    F --> H[lint + commit + push]
    G --> H

    H --> I{新コメント？}
    I -->|はい| C
    I -->|なし| J["Phase 5: マージ
    ユーザーに確認
    gh pr merge --squash
    マージ状態を検証"]
```

## Codex レビューの有効化

1. [chatgpt.com/codex](https://chatgpt.com/codex) を開き、GitHub アカウントを接続
2. [chatgpt.com/codex/settings/code-review](https://chatgpt.com/codex/settings/code-review) を開く
3. 対象リポジトリの **Code review** をオンにする
4. **Automatic reviews** をオンにすると、新しい PR ごとに自動レビューされる

有効化後、PR が作成されるたびに Codex がインラインコメントで自動レビューします。

**手動トリガー:** PR コメントに `@codex review` と入力
**重点レビュー:** PR コメントに `@codex review for security regressions` と入力
**Codex に修正させる:** PR コメントに `@codex address that feedback` と入力

**ヒント:** リポジトリのルートに `AGENTS.md` を追加して、レビュー基準をカスタマイズできます。

> ChatGPT Plus、Pro、Team、Enterprise のいずれかが必要です。

## FAQ

**Q: どのリポジトリに対応していますか？**
A: push 権限のある任意の GitHub リポジトリで使えます。

**Q: コミットメッセージのスタイルをカスタマイズできますか？**
A: はい。スキルは `git log` を分析して既存の規約に従います。

## ライセンス

MIT
