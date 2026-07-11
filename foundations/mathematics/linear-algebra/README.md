# 線形代数 (Linear Algebra)

ベクトル空間と線形写像の理論。SIMD・GPU・機械学習システムの計算の実体である
行列演算を、公理から計算構造まで一貫した視点で文書化する。

## 文書一覧 (Documents)

1. [`vectors-matrices-linear-maps.md`](vectors-matrices-linear-maps.md) —
   ベクトル空間の公理、内積と直交、「行列=線形写像の表現」「行列積=写像の合成」、
   行列積の計算構造(演算強度・ブロック化・Strassen)。シリーズの起点

## ロードマップ (Roadmap)

番号は推奨する読み書きの順序(依存順)。

2. 固有値・固有ベクトルと対角化 (eigenvalues, eigenvectors & diagonalization) —
   線形写像の「不変方向」への分解。マルコフ連鎖・安定性解析・PageRank の言語
3. 特異値分解 (SVD) と低ランク近似 — 非正方行列の標準形。次元削減・圧縮・
   レコメンデーションの土台
4. 行列分解と数値線形代数 (LU / QR / Cholesky) — 連立一次方程式・最小二乗法の
   実用解法。浮動小数点誤差と条件数 (condition number)
5. テンソルと多線形代数 (tensors & multilinear algebra) — 行列の多次元一般化。
   機械学習フレームワークのデータモデルと Tensor Core の演算単位
