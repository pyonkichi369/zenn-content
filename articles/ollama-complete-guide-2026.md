---
title: "Ollama完全解説2026 — ローカルLLMをゼロコストで動かす方法"
emoji: "🦙"
type: "tech"
topics: ["ollama", "llm", "ai", "python", "rag"]
published: true
---

**Ollama**は、LlamaやMistral、Qwenなどの大規模言語モデルをローカルで動かすオープンソースツールです。APIコストゼロ、インターネット不要、OpenAI互換のREST APIを提供します。

この記事では、インストールから本番RAGパイプラインまで実装コード付きで解説します。

---

## 対応モデル一覧（2026年）

| モデル | パラメータ | 必要VRAM | 用途 |
|-------|-----------|---------|------|
| Llama 3.3 | 70B | 48GB | 汎用、GPT-4相当の品質 |
| Llama 3.2 | 3B | 4GB | 低スペック端末、高速応答 |
| Qwen2.5 | 14B | 10GB | 多言語、コーディング |
| Qwen2.5-Coder | 32B | 24GB | コード生成特化 |
| DeepSeek-R1 | 7B〜671B | 8GB〜 | 推論・思考チェーン |
| Gemma 3 | 9B | 8GB | Googleのオープンモデル |
| Mistral | 7B | 8GB | 高速、命令追従 |
| nomic-embed-text | 137M | 1GB | テキスト埋め込み（RAG用） |

---

## クイックスタート

```bash
# インストール（macOS）
brew install ollama

# インストール（Linux）
curl -fsSL https://ollama.com/install.sh | sh

# モデルを引っ張って実行
ollama run llama3.3

# バックグラウンドでAPIサーバー起動
ollama serve
```

起動後、`http://localhost:11434` でAPIが使えます。

---

## REST API

```python
import requests

# チャット
response = requests.post("http://localhost:11434/api/chat", json={
    "model": "llama3.3",
    "messages": [{"role": "user", "content": "RAGを3行で説明して"}],
    "stream": False
})
print(response.json()["message"]["content"])
```

### OpenAI互換エンドポイント

既存のOpenAI SDKコードをそのまま使い回せます：

```python
from openai import OpenAI

# base_urlを変えるだけ
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

response = client.chat.completions.create(
    model="llama3.3",
    messages=[{"role": "user", "content": "こんにちは"}]
)
print(response.choices[0].message.content)
```

LangChain、LlamaIndex、CrewAIもこれで動きます。

---

## Ollamaを使ったRAGパイプライン

```python
import ollama
import chromadb

# ChromaDB初期化
client = chromadb.Client()
collection = client.create_collection("docs")

# ドキュメントをローカルで埋め込み
documents = [
    "RAGは関連ドキュメントを検索してLLMに渡す技術",
    "SupabaseはPostgreSQLベースのBaaS",
    "Claude APIはAnthropicが提供するLLM API"
]

embeddings = [
    ollama.embeddings(model="nomic-embed-text", prompt=doc)["embedding"]
    for doc in documents
]

collection.add(
    documents=documents,
    embeddings=embeddings,
    ids=[str(i) for i in range(len(documents))]
)

# クエリ
query = "検索技術とは？"
query_embedding = ollama.embeddings(model="nomic-embed-text", prompt=query)["embedding"]
results = collection.query(query_embeddings=[query_embedding], n_results=2)

# ローカルLLMで回答生成
context = "\n".join(results["documents"][0])
response = ollama.chat(model="llama3.3", messages=[{
    "role": "user",
    "content": f"以下の情報を元に答えてください。\n\n{context}\n\n質問: {query}"
}])
print(response["message"]["content"])
```

**全処理がローカルで完結。APIコストゼロ。**

---

## Modelfile — モデルのカスタマイズ

```dockerfile
# Modelfile
FROM llama3.3

SYSTEM """
あなたはシニアソフトウェアエンジニアです。
技術的に正確で簡潔な回答をしてください。
必ずコード例を含めてください。
"""

PARAMETER temperature 0.2
PARAMETER num_ctx 8192
```

```bash
ollama create my-engineer -f Modelfile
ollama run my-engineer
```

---

## Ollama vs Claude API — 使い分け

| 観点 | Ollama（ローカル） | Claude API |
|------|-----------------|-----------|
| コスト | **¥0**（電気代のみ） | $3/1Mトークン（Sonnet） |
| プライバシー | **完全ローカル** | Anthropicにデータ送信 |
| インターネット | 不要 | 必要 |
| セットアップ | 5分 | APIキーのみ |
| 品質（コーディング） | 良い | **最高** |
| コンテキスト長 | モデル依存（4K〜128K） | 200K |
| 速度 | ハードウェア依存 | 高速 |

**使い分けの原則:**
- プロトタイプ・コスト削減・プライバシー重視 → **Ollama**
- 本番品質・複雑な推論・エージェント → **Claude API**

---

## ハードウェア要件

| 用途 | 最低スペック | 推奨 |
|------|------------|------|
| 軽量チャット（7B） | 8GB RAM、CPU | Apple M1、8GB VRAM |
| 開発用（14B） | 16GB RAM | Apple M2/M3、16GB |
| 本番品質（70B） | 48GB VRAM | RTX 4090 ×2、M2 Ultra |
| 埋め込みのみ | 4GB RAM | 現代的なCPUなら何でも |

Apple Silicon（M1/M2/M3/M4）はCPUとGPUがメモリを共有するため、16〜32GBのMacが高コスパで動きます。

---

## まとめ

- OllamaはLLMをローカル実行するオープンソースツール（完全無料）
- OpenAI互換APIで既存コードをそのまま流用可能
- nomic-embed-textで埋め込みもローカル処理できRAGが全コストゼロに
- Modelfileでモデルの挙動をカスタマイズ可能

より高品質な本番アプリには Claude API が最適です: [claude.ai/referral/gvWKlhQXPg](https://claude.ai/referral/gvWKlhQXPg)

詳細な比較: [ai-wikis — Ollama完全ガイド](https://github.com/pyonkichi369/ai-wikis/blob/main/tools/ollama.md)
