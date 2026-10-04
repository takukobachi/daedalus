# 最適化 (Optimization)

制約の下で最良を選ぶ数学。ロケットの着陸誘導は燃料最小化を凸問題に変換して
機上で解き、電力市場は経済負荷配分の双対変数を電気の値段として公表し、
機械学習の訓練は確率的勾配降下の一大応用である。既存の分析ドキュメントの
あちこちに現れるこうした「min f(x) subject to …」を、凸性・勾配法・KKT 条件・双対性
という一つの体系として基礎から積み上げる。シリーズ全体を通じた主題は2つある。
1つは**凸性が「解ける/解けない」の分水嶺であること**で、凸なら局所最適＝大域最適となり、
実務は「どう凸に落とすか」の工学になる。もう1つは**双対変数が「制約の値段」という
経済的実体を持つこと**で、電力市場の価格はこの数学的対象がそのまま商品になった例である。

## 文書一覧 (Documents)

1. [`convex-optimization-duality.md`](convex-optimization-duality.md) —
   最適化問題の標準形と凸性（局所＝大域の定理）、LP/QP/SOCP の階層、
   勾配降下と条件数、ニュートン法、確率的勾配降下 (SGD)、ラグランジュ乗数と
   KKT 条件、双対性と影の価格（電力市場の限界価格・TCP 公平性の効用最大化）、
   凸化の工学（着陸誘導の lossless convexification）と MPC

## 今後のテーマ候補

- 数値最適化の実装各論 (numerical optimization) — 直線探索と信頼領域、
  L-BFGS、内点法の実装、疎性の搾取。`../linear-algebra/matrix-factorizations-numerical.md`
  の解法側の続き（`../numerical-analysis/` のロードマップと合流する）
- 整数計画と組合せ最適化 (integer & combinatorial optimization) — 分枝限定法、
  緩和と近似、スケジューリング・配置問題。`../discrete-logic/graphs-trees-algorithms.md`
  との合流点
- オンライン最適化と後悔最小化 (online optimization & regret) — バンディット、
  オンライン凸最適化。広告配信・キャッシュ置換・適応制御の理論
