# 線形代数 (Linear Algebra)

ベクトル空間と線形写像の理論。SIMD・GPU・機械学習システムの計算の実体である
行列演算を、公理から計算構造まで一貫した視点で文書化する。

## 文書一覧 (Documents)

1. [`vectors-matrices-linear-maps.md`](vectors-matrices-linear-maps.md) —
   ベクトル空間の公理、内積と直交、「行列=線形写像の表現」「行列積=写像の合成」、
   行列積の計算構造(演算強度・ブロック化・Strassen)。シリーズの起点
2. [`eigenvalues-diagonalization.md`](eigenvalues-diagonalization.md) —
   固有値・固有ベクトル、対角化と行列の冪、スペクトル定理、確率行列と定常分布。
   反復・安定性・マルコフ連鎖の言語
3. [`svd-low-rank.md`](svd-low-rank.md) —
   特異値分解(任意の行列の「回転・伸縮・回転」分解)、Eckart–Young の定理と
   低ランク近似、条件数、PCA。圧縮・次元削減・LoRA の土台

## ロードマップ (Roadmap)

番号は推奨する読み書きの順序(依存順)。

4. 行列分解と数値線形代数 (LU / QR / Cholesky) — 連立一次方程式・最小二乗法の
   実用解法。浮動小数点誤差と条件数 (condition number)
5. テンソルと多線形代数 (tensors & multilinear algebra) — 行列の多次元一般化。
   機械学習フレームワークのデータモデルと Tensor Core の演算単位
