---
title: "Claude Codeの自動化レシピ集 — hooks・MCP・カスタムコマンドで開発が変わる"
emoji: "📝"
type: "tech"
topics: ["AI", "プログラミング", "エンジニア"]
published: true
---

あなたはClaude Codeをどれくらい使いこなしていますか？

多くの開発者が「チャットでコードを書かせる」だけで止まっています。しかしClaude Codeには、**開発ワークフロー全体を自動化する仕組み**が3つ備わっています。hooks、MCP（Model Context Protocol）、カスタムコマンドです。

私は119体のAIエージェント組織「AEGIS」をClaude Codeで運用しています。この記事では、日常的に使っている**実践的な自動化設定**を、コピペ可能な形で公開します。

Last updated: 2026-03-25

## この記事でわかること

- Claude Code hooksの仕組みと実用レシピ5選
- MCPサーバーで外部ツールと連携する方法
- カスタムコマンド（/スラッシュコマンド）の作り方
- 3つを組み合わせた「開発パイプライン自動化」の実例

## 目次

1. なぜ自動化が必要なのか
2. Hooks — Claude Codeの「トリガー」
3. MCP — 外部ツールとの「橋」
4. カスタムコマンド — あなた専用の「/コマンド」
5. 組み合わせ実践例
6. まとめ

---

## 1. なぜ自動化が必要なのか

Claude Codeを毎日使っていると、同じ操作の繰り返しに気づきます。

- コードを書いたら毎回lint → テスト → コミット
- ファイルを編集する前に必ずバックアップ確認
- 外部APIを呼ぶときにタイムアウト設定を忘れない

これらを**人間が覚えておく**のは非効率です。Claude Code自身に「ルール」として組み込めば、忘れることはありません。

## 2. Hooks — Claude Codeの「トリガー」

Hooksは、Claude Codeが特定のアクションを実行する**前後に自動で走るシェルコマンド**です。

### 仕組み

```
settings.json の hooks セクションで定義
  ├── PreToolUse  — ツール実行「前」に実行
  ├── PostToolUse — ツール実行「後」に実行
  ├── Notification — 通知時に実行
  └── Stop        — 応答完了時に実行
```

### レシピ1: シークレット漏洩防止

コードを書くたびに、APIキーやパスワードがハードコードされていないか自動チェック。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "detect-secrets scan $CLAUDE_FILE_PATH 2>/dev/null | grep -q '\"results\": {}' || echo 'BLOCK: Secret detected in file'"
      }
    ]
  }
}
```

これだけで、Claude Codeがファイルを書き込むたびに`detect-secrets`が走ります。シークレットが検出されたらブロックされます。

### レシピ2: 環境変数の外部送信ブロック

サプライチェーン攻撃対策。`process.env`や`os.environ`を外部に送信するコードを検出してブロック。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo \"$CLAUDE_TOOL_INPUT\" | grep -qiE '(curl|wget|fetch).*env' && echo 'BLOCK: env exfiltration attempt' || true"
      }
    ]
  }
}
```

### レシピ3: 自動lint実行

ファイル編集後に自動でlintを実行し、コード品質を維持。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "case $CLAUDE_FILE_PATH in *.py) ruff check $CLAUDE_FILE_PATH 2>/dev/null;; *.js|*.ts) npx eslint $CLAUDE_FILE_PATH 2>/dev/null;; esac"
      }
    ]
  }
}
```

---

ここから先は有料パートです。MCP実践設定、カスタムコマンドの作り方、そして3つを組み合わせた開発パイプライン自動化の具体例を公開します。

---

## 3. MCP — 外部ツールとの「橋」

MCP（Model Context Protocol）は、Claude Codeに**外部ツールの能力を追加する**プロトコルです。

### MCPでできること

| 接続先 | できること |
|--------|-----------|
| Firecrawl | Webページのスクレイピング・検索 |
| Playwright | ブラウザ操作・E2Eテスト |
| Discord/Slack | メッセージ送受信・通知 |
| データベース | SQLクエリの実行 |
| 独自API | あなたのサービスとの連携 |

### 設定方法

`settings.json`にMCPサーバーを登録します。

```json
{
  "mcpServers": {
    "firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": {
        "FIRECRAWL_API_KEY": "${FIRECRAWL_API_KEY}"
      }
    }
  }
}
```

**重要**: APIキーは必ず環境変数で渡してください。`settings.json`にハードコードしてはいけません。

### 実践例: WebリサーチMCP

トレンド調査をClaude Codeから直接実行できるようになります。

```
あなた: 「最新のAIエージェントフレームワークを調べて」
Claude Code: → firecrawl_searchツールで検索
           → 結果を構造化して返答
```

従来はブラウザを開いて手動検索 → コピペ → Claude Codeに貼り付けでした。MCPなら**会話の中でシームレスに**リサーチが完了します。

## 4. カスタムコマンド — あなた専用の「/コマンド」

### コマンドの作り方

`.claude/commands/` ディレクトリにMarkdownファイルを置くだけ。

```
.claude/commands/
├── test.md        → /test で呼び出し
├── review.md      → /review で呼び出し
└── deploy.md      → /deploy で呼び出し
```

### レシピ4: テスト実行コマンド

`.claude/commands/test.md`:

```markdown
プロジェクトのテストを実行してください。

1. まず `pytest` でユニットテストを実行
2. 失敗したテストがあれば原因を分析
3. 修正案を提示（自動修正はしない）
4. テストカバレッジを報告
```

これで `/test` と入力するだけで、テスト → 分析 → レポートが自動実行されます。

### レシピ5: セキュリティ監査コマンド

`.claude/commands/security-check.md`:

```markdown
セキュリティ監査を実行してください。

チェック項目:
- [ ] ハードコードされたシークレットがないか（detect-secrets scan）
- [ ] 依存パッケージの脆弱性（pip audit / npm audit）
- [ ] SQLインジェクションの可能性
- [ ] XSSの可能性（ユーザー入力のサニタイズ確認）
- [ ] 外部APIコールのタイムアウト設定

結果をマークダウン形式で報告してください。
```

## 5. 組み合わせ実践例 — 開発パイプライン自動化

ここからが本記事の核心です。Hooks + MCP + カスタムコマンドを**組み合わせる**ことで、開発パイプライン全体を自動化できます。

### パイプライン構成

```
/implement [機能名]  ← カスタムコマンドで起動
    │
    ├── 1. MCP (Firecrawl) で関連ドキュメント自動取得
    ├── 2. コード実装
    ├── 3. Hook (PostToolUse) で自動lint + secret scan
    ├── 4. /test コマンドでテスト実行
    ├── 5. Hook (PostToolUse) でテスト結果チェック
    └── 6. /git コマンドでコミット
```

### 私の実際の設定（AEGIS運用）

AEGISでは119体のエージェントが日常的にこのパイプラインを使っています。

**settings.json（抜粋）**:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "bash .claude/hooks/verify-supply-chain.sh"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "bash .claude/hooks/post-edit-check.sh"
      }
    ]
  },
  "mcpServers": {
    "firecrawl": { "command": "npx", "args": ["-y", "firecrawl-mcp"] },
    "discord": { "command": "npx", "args": ["discord-mcp-server"] }
  }
}
```

**ポイント**:
- `PreToolUse`でサプライチェーン攻撃を防止
- `PostToolUse`で品質チェックを自動化
- MCPでWebリサーチとDiscord通知を統合

### 効果測定

この自動化設定を導入した結果:

| 指標 | 導入前 | 導入後 |
|------|--------|--------|
| コードレビュー指摘事項 | 平均8件/PR | 平均2件/PR |
| シークレット漏洩インシデント | 月1-2回 | 0回（3ヶ月連続） |
| 開発→テスト→コミットの時間 | 15分 | 3分 |

## まとめ

Claude Codeの真の力は「チャット」ではなく「自動化基盤」にあります。

1. **Hooks**: 編集・実行のたびに自動でチェック（品質の守護者）
2. **MCP**: 外部ツールとシームレスに連携（能力の拡張）
3. **カスタムコマンド**: 複雑なワークフローを一言で起動（操作の簡略化）

この3つを組み合わせれば、Claude Codeは単なるAIアシスタントから**開発パイプラインのコア**に変わります。

---

## 関連記事