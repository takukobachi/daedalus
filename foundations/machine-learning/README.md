# Machine Learning (機械学習)

深層学習 (deep learning)、とくに現代の LLM を支える理論の系列。数学・物理・情報理論に
続く 4 つ目の基礎理論系列であり、線形代数（行列積、低ランク）、確率・統計（最尤推定、
交差エントロピー）、最適化（勾配法）、情報理論（符号長＝圧縮）の**合流点**にあたる。

応用側との分担は次のとおりである。

- **`foundations/machine-learning/`（本ディレクトリ）** — 数理: なぜこの式か、
  どこから導出されるか
- **`../../software/ai-llm/`** — モデルの上のシステム: エージェント、プロンプティング、
  ループの工学。とくに `llm-internals-prompting.md` は本系列の「実務向けの出口」
- **`../../software/ml-systems/`** — モデルの下のシステム: 推論・学習の実行基盤。
  本系列が導出する量（パラメータ数、FLOPs、KV cache サイズ）の消費者

## 構成

- `transformer-architecture.md` — Transformer の数理: 残差流 (residual stream) という
  見取り図、注意機構の導出（QKV、スケーリング因子 1/√d_k、因果マスク）、
  マルチヘッド、RoPE、MLP ブロック＝連想記憶、学習目的（交差エントロピー＝符号長）、
  パラメータ・FLOPs・KV cache の会計
- `scaling-laws.md` — スケーリング則: 損失のべき則、計算最適学習 (Chinchilla)、
  過学習ならぬ「過訓練」の経済学（推論コストを含む最適化）、創発をめぐる論争、
  テスト時計算という第二のスケーリング軸

## 今後のテーマ候補

- 逆伝播と最適化の実務 (backpropagation & optimizers) — 自動微分、Adam 系列、
  学習率スケジュール、損失地形。`../mathematics/optimization/` の非凸側への拡張
- 強化学習の基礎 (reinforcement learning) — MDP、方策勾配、PPO。
  RLHF (`../../software/ai-llm/llm-internals-prompting.md` §4) の理論的土台
- 汎化の理論 (generalization) — 二重降下、宝くじ仮説、暗黙の正則化
