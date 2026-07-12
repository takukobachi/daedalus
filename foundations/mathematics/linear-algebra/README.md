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
4. [`matrix-factorizations-numerical.md`](matrix-factorizations-numerical.md) —
   LU / Cholesky / QR 分解、最小二乗法、浮動小数点 (IEEE 754) と後退誤差解析。
   「計算できた」と「信用できる」を分ける数値線形代数の入口
5. [`tensors-multilinear.md`](tensors-multilinear.md) —
   多線形写像としてのテンソル、縮約と einsum 記法、メモリレイアウトとビュー、
   CP/Tucker 分解。ML フレームワークと Tensor Core の演算単位。シリーズ最終回

当初計画した5本のシリーズはこれで完結。以降のテーマ(反復法・疎行列、
ランダム化数値線形代数など)は必要が生じた時点でロードマップを再開する。
