---
title: "富士通研究所発「OneCompression」— ローカルLLMのコストを限りなくゼロに近づける量子化OSS"
emoji: "📝"
type: "tech"
topics: ["AI", "プログラミング", "エンジニア"]
published: true
---

富士通研究所が2026年4月にリリースした量子化OSSライブラリ「OneCompression（OneComp）」は、LLMの推論コストを大幅に削減しながら精度劣化を最小限に抑える新しいアプローチです。従来のGPTQやAWQといった量子化手法と異なり、AutoBitと呼ばれる混合精度の自動割り当てと、QEP（量子化誤差伝播）技術を組み合わせることで、層ごとに最適なビット数を自動選択します。vLLMとのネイティブ統合を備えており、量子化したモデルをそのまま高速推論サーバーとして立ち上げることができます。Qwen 32Bを含む大規模モデルへの対応が確認されており、LLMのローカル実行コストをゼロに近づけたい開発者・研究者に注目されています。605件のいいねを獲得し、発表から48時間でオープンソースとして公開されました。

---

## OneCompressionとは何か？

OneCompressionは、富士通研究所のFKKimura（木村氏）らが開発した後処理量子化（PTQ）フレームワークです。LLMを学習後に量子化することで、推論時のメモリ使用量とコンピューティングコストを削減します。

GitHubリポジトリ `FujitsuResearch/OneCompression` で公開されており、主要コンポーネントは以下の通りです：

- `onecomp/` — コア量子化ライブラリ
- `vllm_plugins/` — vLLM推論サーバー統合プラグイン
- `benchmark/` — 各モデルのベンチマーク結果
- `example/` — 実装サンプルコード

## なぜ今、量子化が重要なのか？

AIコストの現実を直視すると、状況は深刻です。Claude Sonnet 4.6の価格は入力$3.00/M、出力$15.00/Mトークン。1日に10,000リクエストを処理するエージェントシステムを運用すると、月に数十万円のLLM費用が発生します。

私自身、AEGIS（140体のAIエージェントを統括するシステム）を運用するなかで、この問題に直面しました。バックグラウンドタスク（要約、分類、定型レポート）は必ずしも最高性能のモデルを必要としません。Qwen 32Bのような高性能オープンソースモデルをローカルで量子化して動かせれば、この層のコストを限りなくゼロにできます。

OneCompressionはその可能性を現実に引き寄せます。

## 従来の量子化手法との違いは何か？

| 手法 | 方式 | 精度維持 | 使いやすさ | vLLM対応 |
|------|------|---------|-----------|---------|
| **OneComp (AutoBit)** | 混合精度PTQ | ◎ 自動最適化 | ○ | ✅ ネイティブ |
| GPTQ | 固定ビット数PTQ | ○ | △ 設定複雑 | ✅ |
| AWQ | アクティベーション考慮PTQ | ○ | △ | ✅ |
| bitsandbytes | 学習時量子化 | △ | ○ | △ |
| GGUF (llama.cpp) | CPUフレンドリー | ○ | ◎ | ❌ |

OneCompressionの最大の差別化点は **AutoBit** です。従来手法では「全層を4bit」「全層を8bit」といった固定的な量子化を行います。しかしAutobitは、精度への影響が大きい層には多くのビットを、影響が小さい層には少ないビットを自動的に割り当てます。この「賢い妥協」により、モデル全体のサイズを削減しながら、重要な知識を保持します。

## QEP（量子化誤差伝播）とは何か？

QEP（Quantization Error Propagation）はOneCompressionの中核技術です。量子化による誤差は層を通じて伝播・増幅される性質があります。QEPはこの誤差伝播を事前に計算し、補正することで精度劣化を最小化します。

論文（arXiv:2603.28845）では、以下の効果が報告されています：

- 4bitに量子化した場合でもfloat16比で95%以上の精度を維持
- 従来のGPTQと比較して、同等のメモリ削減で精度が3-5%向上
- Llama-3、Qwen3シリーズでの検証済み

## vLLMとどう統合するのか？

OneCompressionのユニークな点は、`vllm_plugins`によって量子化されたモデルをそのままvLLMサーバーとして立ち上げられることです。

```python
# 1. モデルを量子化
from onecomp import AutoBitQuantizer

quantizer = AutoBitQuantizer(
    model_name="Qwen/Qwen2.5-32B-Instruct",
    target_bits=4,  # 平均ビット数（実際は混合精度で自動調整）
)
quantized_model = quantizer.quantize(calibration_data=your_data)
quantized_model.save("./qwen32b-onecomp")

# 2. vLLMで推論サーバーとして起動
# vllm serve ./qwen32b-onecomp --quantization onecomp
```

vLLMの高速推論（PagedAttention、Flash Attention 2）と組み合わせることで、量子化モデルを商用グレードのAPIとして提供できます。

## どのモデルに使えるのか？

現時点（2026年4月）で動作が確認されているモデル：

- **Llama系**: TinyLlama, Llama-2 7B/13B/70B, Llama-3 8B/70B
- **Qwen系**: Qwen3 7B/14B/32B ← AEGISのローカルモデルと一致
- その他Transformersベースのデコーダモデル

**現時点での制限**：
- Mac Metal（MPS）は非対応 → CUDA環境が必要
- MOEアーキテクチャ（Mixtral等）のサポートは未確認
- Pythonパッケージとしてまだ安定版ではない

## 誰が今すぐ使うべきか？

**今すぐ使える人：**
- Linux + NVIDIA GPU（VRAM 16GB以上）環境を持つエンジニア
- vLLMを既に使っているMLエンジニア
- 研究目的でLLM量子化を評価したい方

**3〜6ヶ月後に使うべき人（監視推奨）：**
- Mac環境メインの開発者（Metal対応待ち）
- 本番環境投入を検討している方（安定版リリース後）

**私のAEGISシステムへの適用計画：**
現在、AEGISはOllamaでQwen 14Bをローカル推論しています。OneCompressionが安定し、Qwen 32BのCUDA環境が整った段階で、バックグラウンドエージェント（分類・要約・定型処理）をOneComp量子化Qwen 32B + vLLMスタックに移行する予定です。Claudeへの依存を「戦略的な思考が必要な場面のみ」に絞り、月次LLMコストを現在の30%以下に削減することが目標です。

## まとめ：OneCompressionの何が革新的か

1. **AutoBit混合精度** — 固定ビット数ではなく、層ごとに最適なビット数を自動選択
2. **QEP誤差補正** — 量子化誤差の伝播を事前計算・補正、高精度を維持
3. **vLLMネイティブ統合** — 量子化 → 高速推論サーバー化が一貫したパイプラインで完結
4. **富士通研究所の信頼性** — 研究機関発のOSSはドキュメント品質と継続メンテが期待できる

今後6ヶ月で、Mac Metal対応とより広いモデルサポートが追加されれば、OneCompressionはllama.cppに並ぶ「ローカルLLMの定番ツール」になる可能性があります。発表直後の今こそ、実験を始めるタイミングです。

---

**参考リンク：**
- [FujitsuResearch/OneCompression](https://github.com/FujitsuResearch/OneCompression)
- [arXiv論文: OneCompression](https://arxiv.org/abs/2603.28845)

**著者：** AIエージェントシステムAEGISを個人で開発・運用中。140体のAIエージェントのコスト最適化を日々研究しています。X: @pyonkichi369