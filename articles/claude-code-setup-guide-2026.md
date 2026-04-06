---
title: "Claude Code完全セットアップガイド2026 — インストールから実践まで"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "claude", "ai", "mcp", "開発効率化"]
published: true
---

**Claude Code**はAnthropicが提供するターミナルベースのAIコーディングアシスタントです。コードベース全体を自律的に読み書き・実行でき、ファイルパスを指定しなくてもプロジェクト全体を理解して作業します。

この記事では、インストールから生産性を最大化する設定まで完全解説します。

---

## 必要なもの

- Node.js 18以上
- Anthropicアカウント（Claude ProまたはMax）
- macOS / Linux / Windows（WSL）

---

## インストール

```bash
# グローバルインストール
npm install -g @anthropic-ai/claude-code

# バージョン確認
claude --version

# 起動（初回はブラウザで認証）
claude
```

---

## サブスクリプション

| プラン | Claude Code | 料金 |
|-------|------------|------|
| Claude Pro | ✅ | $20/月 |
| Claude Max（5x） | ✅ 上限高め | $100/月 |
| Claude Max（20x） | ✅ 上限最大 | $200/月 |
| API従量課金 | ✅ | トークン単価 |

[Claude Proを始める →](https://claude.ai/referral/gvWKlhQXPg)

---

## 主要コマンド

### スラッシュコマンド

| コマンド | 動作 |
|---------|------|
| `/help` | 使えるコマンド一覧 |
| `/clear` | 会話コンテキストをリセット |
| `/compact` | コンテキストを圧縮してトークン節約 |
| `/model` | モデル切り替え（Haiku/Sonnet/Opus） |
| `/cost` | トークン使用量とコスト確認 |
| `/init` | CLAUDE.mdを自動生成 |
| `/permissions` | ツール権限の確認・変更 |

### キーボードショートカット

| ショートカット | 動作 |
|-------------|------|
| `Ctrl+C` | 実行中の作業をキャンセル |
| `↑` / `↓` | コマンド履歴 |
| `Esc` | 入力キャンセル |

---

## CLAUDE.md — プロジェクト永続コンテキスト

プロジェクトルートに `CLAUDE.md` を作ると、毎セッション自動で読み込まれます：

```markdown
# プロジェクトコンテキスト

## スタック
- Next.js 15、TypeScript、Tailwind CSS
- Supabase（PostgreSQL + pgvector）
- Vercelでデプロイ

## コーディング規約
- 関数コンポーネントのみ
- TypeScript strict mode
- `any`型禁止 — 適切なinterfaceを使う
- テストは `__tests__/` にVitest

## 重要ファイル
- `lib/supabase.ts` — Supabaseクライアント
- `components/` — 再利用可能UIコンポーネント
- `app/api/` — Next.js APIルート

## 注意事項
- `.env.local` は絶対にコミットしない
- 新テーブルには必ずRLSを有効化
- コミット前に `npm run typecheck` を実行
```

この1ファイルで「毎回同じ説明をする手間」がゼロになります。

---

## MCP — 外部ツールとの接続

MCPサーバーを設定するとClaude CodeがGitHub・DB・ファイルを直接操作できます：

```json
// ~/.claude/mcp_settings.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..." }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

設定後に再起動すると「このIssueを見て実装して」「このテーブルのスキーマ確認して」が使えます。

---

## 効果的なプロンプト例

### バグ修正

```
src/api/auth.ts:47のTypeError（OAuthコールバック時にuser.idがundefined）を修正して
```

### 機能追加

```
/api/generateにRedisを使ったレート制限（IP単位で毎分10リクエスト）を追加して
```

### コードレビュー

```
最後のコミットの変更をレビューして。セキュリティ問題とパフォーマンス問題を優先して指摘して
```

### リファクタリング

```
components/UserTable.tsxをuseEffect+fetchからReact Queryに移行して
```

### プロジェクト全体の分析

```
全APIエンドポイントを洗い出して、認証ミドルウェアがないものを特定して
```

---

## Claude Code vs Cursor vs Windsurf

| 観点 | Claude Code | Cursor | Windsurf |
|-----|------------|--------|---------|
| インターフェース | ターミナルCLI | VS Code IDE | VS Codeフォーク + 40 IDE |
| 自律性 | **高** | 中 | 高 |
| コンテキスト | **コードベース全体** | 200Kトークン | 100Kトークン |
| マルチファイル編集 | **無制限** | 5〜8ファイル | 3〜4ファイル |
| ターミナル実行 | **ネイティブ** | なし | Cascade経由 |
| 料金 | $20/月 | $20/月 | $15/月 |
| 最適な用途 | DevOps・大規模コードベース・CLI | フロントエンド・React | JetBrains・コスト重視 |

---

## ヘッドレスモード（CI/CD活用）

```bash
# インタラクティブなし、出力のみ
claude --print "このPRの変更を要約して" --no-interactive

# ファイルに出力
claude --print "APIの変更点を列挙して" > pr-description.md

# git hookで活用
# .git/hooks/pre-commit
claude --print "このdiffにセキュリティ問題があれば指摘して" < <(git diff --cached)
```

---

## コスト管理のコツ

| 方法 | 効果 |
|-----|------|
| `/compact` をタスク切替時に使う | コンテキスト圧縮でトークン節約 |
| `/cost` で使用量を定期確認 | 予算管理 |
| 単純作業は `/model claude-haiku` | 最安モデルに切替 |
| CLAUDE.mdで冗長な説明を省く | 毎回のトークン削減 |

---

## まとめ

- `npm install -g @anthropic-ai/claude-code` でインストール完了
- CLAUDE.mdでプロジェクトコンテキストを永続化
- MCPで外部ツール（GitHub・DB・ファイル）と接続
- ヘッドレスモードでCI/CDに組み込み可能

今すぐ始める: [claude.ai/code](https://claude.ai/referral/gvWKlhQXPg)

詳細なセットアップガイド: [ai-wikis — Claude Code Setup](https://github.com/pyonkichi369/ai-wikis/blob/main/guides/claude-code-setup.md)
