---
title: "RotorQuant vs TurboQuant — KVキャッシュ量子化の最前線"
emoji: "📝"
type: "tech"
topics: ["AI", "プログラミング", "エンジニア"]
published: true
---

## はじめに：なぜKVキャッシュが問題なのか

ローカルLLM推論で最大のボトルネックの一つが **KVキャッシュのメモリ消費**です。例えばQwen2.5-14Bで128Kコンテキストを処理すると、KVキャッシュだけで約18GBのVRAMを消費します。24GB GPUではモデル本体を載せた時点でほぼ余裕がなく、長文コンテキストの処理が実質不可能になります。

この問題を解決するのが **KVキャッシュ量子化** — 推論時にKey-Valueキャッシュを低ビットに圧縮し、メモリ使用量を劇的に削減する技術です。2026年3月、この分野で2つの注目すべき手法が登場しました。

## KVキャッシュ量子化とは

通常のモデル量子化（GGUF、AWQなど）がモデルの**重み**を圧縮するのに対し、KVキャッシュ量子化は**推論時に動的に生成されるAttentionのKey/Valueテンソル**を圧縮します。

- 重み量子化：モデルサイズ削減（ロード時に確定）
- KVキャッシュ量子化：コンテキスト長に比例するメモリ削減（推論中にリアルタイム処理）

つまり、両者は**併用可能**であり、組み合わせることで大幅なメモリ削減が実現します。

## TurboQuant — Googleが提案する最適圧縮

**TurboQuant**はGoogle Researchが開発し、ICLR 2026で発表された手法です。

### アルゴリズム概要

TurboQuantは2段階の圧縮を行います：

1. **PolarQuant**: ベクトルをデカルト座標から極座標に変換し、半径（大きさ）と角度（方向）に分離。角度分布が予測可能なため、従来必要だったブロック単位の正規化をスキップ可能
2. **QJL補正**: Johnson-Lindenstrauss変換で高次元データを縮小しつつ距離関係を保持。各値を1ビットの符号（+1/-1）に削減

### 性能

| 指標 | 値 |
|------|-----|
| 圧縮率 | 3ビットで**6倍**メモリ削減 |
| 速度向上 | Attention計算が最大**8倍**高速（H100） |
| 精度劣化 | **ゼロ**（LongBench, RULER等で検証済み） |
| 検証モデル | Gemma, Mistral |

### 使い方（Python / HuggingFace）

```python
from turboquant import TurboQuantCache

# 4ビットKVキャッシュで生成
cache = TurboQuantCache(bits=4)
outputs = model.generate(
    input_ids,
    past_key_values=cache,
    max_new_tokens=512
)
```

### llama.cppでの利用（コミュニティ実装）

```bash
# turboquant_plusフォーク（Apple Silicon Metal対応）
git clone https://github.com/TheTom/turboquant_plus
cmake -B build -DGGML_METAL=ON && cmake --build build -j8

# turbo3: 3.25bit, 4.9x圧縮 / turbo4: 4.25bit, 3.8x圧縮
./llama-cli -m model.gguf \
  --cache-type-k turbo3 --cache-type-v turbo3 \
  -ngl 99 -c 32768
```

> **注意**: 2026年3月時点で、llama.cpp本家にはまだマージされていません。フォーク版での利用となります。

## RotorQuant — Clifford代数による高速化

**RotorQuant**はScrya社が開発した手法で、TurboQuantのコアアルゴリズムを**Clifford代数（幾何代数）のロータ**で再設計したものです。

### 核心的アイデア

TurboQuantでは`d x d`のランダム直交回転行列（d=128で16,384回の乗算加算）を使いますが、RotorQuantはこれをClifford代数 Cl(3,0) のロータ `R = exp(B/2)` に置き換えます。

ベクトルを3次元グループに分割し、各グループに4パラメータのロータでサンドイッチ積 `RvR~` を適用。結果として：

- **乗算加算**: 16,384 → 約2,064回（**7.9倍削減**）
- **パラメータ数**: 16,399 → 372（**44倍削減**）

### 性能

| 指標 | 値 |
|------|-----|
| vs TurboQuant速度 | CUDA: **10-19倍**、Metal: **9-31倍**高速 |
| パラメータ数 | **44倍少ない**（372 vs 16,399、d=128） |
| Attention忠実度 | コサイン類似度 0.990（TurboQuant: 0.991） |
| Tritonカーネル | PyTorchの**100-650倍**高速（量子化/逆量子化） |
| 検証モデル | Qwen2.5-3B-Instruct |

### 使い方

```python
# GitHubからインストール
# pip install rotorquant  (PyPI) or clone from GitHub
from rotorquant import RotorQuantKVCache

# RotorQuantMSE: MSE最適化
# RotorQuantProd: 内積保存最適化
# RotorQuantKVCache: KVキャッシュ専用
quantizer = RotorQuantKVCache(bits=3, dim=128)
compressed_keys = quantizer.quantize(key_states)
restored_keys = quantizer.dequantize(compressed_keys)
```

## 比較表

| 項目 | TurboQuant | RotorQuant |
|------|------------|------------|
| 開発元 | Google Research | Scrya |
| 発表 | ICLR 2026 | 2026年3月（GitHub） |
| 理論基盤 | PolarQuant + QJL | Clifford代数ロータ |
| 圧縮率（3bit） | 6倍 | 5倍（同等） |
| 速度（vs FP16） | 最大8倍 | TurboQuantの10-19倍 |
| パラメータ数（d=128） | 16,399 | 372 |
| Attention忠実度 | cos sim 0.991 | cos sim 0.990 |
| GPU対応 | CUDA | CUDA + Metal |
| llama.cpp統合 | フォーク版あり | 未対応 |
| HuggingFace統合 | `turboquant` PyPI | GitHub |
| 成熟度 | Google論文 + コミュニティ実装 | 新規・実験的 |

## どちらを選ぶべきか

**TurboQuantが適しているケース：**
- llama.cppやOllamaで**今すぐ**使いたい（フォーク版経由）
- HuggingFace Transformersとの統合が必要
- Google論文に裏付けられた安定性を重視

**RotorQuantが適しているケース：**
- **Apple Silicon**（Metal）で最大性能を引き出したい
- PyTorch/Tritonベースのカスタム推論パイプラインを構築中
- パラメータ効率が重要（エッジデバイス等）
- 最先端の研究成果を試したい

**現実的な推奨：**
2026年3月時点では、llama.cppエコシステムを使うならTurboQuantのフォーク版が最も実用的です。RotorQuantは理論的に優れた点が多いですが、推論フレームワークへの統合はこれからです。両者の技術は相互排他ではなく、RotorQuantの回転手法がTurboQuantのパイプラインに統合される可能性も十分にあります。

## 速報：Intel Arc Pro B70 — 32GB VRAM が $949 で登場

KVキャッシュ量子化の話をしているまさにこのタイミングで、ハードウェア側にも大きな動きがありました。

**Intel Arc Pro B70**が2026年3月25日に発売。スペック：

| 項目 | 値 |
|------|-----|
| VRAM | **32GB GDDR6** |
| 帯域幅 | 608 GB/s |
| AI性能 | 367 TOPS |
| 価格 | **$949** |
| TDP | 290W |

比較：RTX 5090（32GB）= $1,999、RTX 5080（16GB）= $999。**半額で同等のVRAM**です。

### KVキャッシュ量子化との組み合わせ

32GB VRAMがあれば、KVキャッシュ量子化なしでも14B-27Bモデルは動きます。しかし**量子化と組み合わせることで次元が変わります**：

| シナリオ | KVキャッシュなし | TurboQuant 3bit |
|---------|----------------|-----------------|
| Qwen 3.5 27B (Q4) + 32Kコンテキスト | KV: ~4.5GB → 残り余裕 | KV: ~0.75GB → 70Bモデルの余地 |
| Qwen 3.5 27B (Q4) + 128Kコンテキスト | KV: ~18GB → ギリギリ | KV: ~3GB → 余裕 |
| 70Bモデル (Q4) + 32Kコンテキスト | VRAM不足 | KV圧縮で**動作可能に** |

つまり、**$949のGPU + KVキャッシュ量子化 = $2,000クラスの推論能力**になる可能性があります。

ただし注意点として、Intel GPUのllama.cppサポート（SYCL/oneAPI経由）はNVIDIA CUDAほど成熟していません。TurboQuant/RotorQuantのIntel GPU対応も現時点では未検証です。ソフトウェアエコシステムの成熟が鍵になります。

## まとめ

KVキャッシュ量子化は、ローカルLLMの実用性を劇的に変える技術です。24GB GPUで128Kコンテキストが現実的になり、M1/M2 Macでもより大きなモデルが動作可能になります。

Intel Arc Pro B70のような大容量VRAM GPUの低価格化と、TurboQuant/RotorQuantのようなKVキャッシュ量子化技術の進化が同時に起きています。この2つの波が合流すれば、ローカルLLM推論のコスパは2026年後半に劇的に改善するでしょう。

## リンク

- [TurboQuant - Google Research Blog](https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/)
- [TurboQuant PyTorch実装 (tonbistudio)](https://github.com/tonbistudio/turboquant-pytorch)
- [turboquant PyPI](https://pypi.org/project/turboquant/)
- [RotorQuant GitHub (scrya-com)](https://github.com/scrya-com/rotorquant)
- [RotorQuant公式サイト](https://www.scrya.com/rotorquant/)
- [llama.cpp TurboQuantディスカッション](https://github.com/ggml-org/llama.cpp/discussions/20969)
- [llama.cpp TurboQuant Feature Request](https://github.com/ggml-org/llama.cpp/issues/20977)
- [turboquant_plus (Metal対応フォーク)](https://github.com/TheTom/turboquant_plus)