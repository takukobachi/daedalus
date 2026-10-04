# 行列分解と数値線形代数 (Matrix Factorizations and Numerical Linear Algebra)

## 概要 (Overview)

線形代数シリーズの第4回。ここまでの3回は「行列とは何か」を理論として積み上げてきたが、今回は**計算機で実際に解く**話に移る。中心となる問いは2つある。`Ax = b` をどう解くか、そして**その答は何桁信用できるか**である。

鍵となる発想は、逆行列を作らないことである。実用の数値線形代数は `A⁻¹` を決して陽に計算せず、A を構造の良い行列の積に**分解 (factorization)** してから、各因子について簡単な問題を順に解く。

```text
LU 分解       A = LU      (下三角 × 上三角)     — 一般の正方行列
Cholesky 分解 A = LLᵀ     (下三角 × その転置)    — 対称正定値行列
QR 分解       A = QR      (直交 × 上三角)       — 長方形も可、最小二乗法
```

後半では、この計算を支える基盤である**浮動小数点 (floating-point)** 演算の誤差を扱う。「アルゴリズムがどれだけ誤差を増やすか(後退安定性)」と「問題自体がどれだけ誤差に敏感か(条件数、前回 §4)」を分離する誤差解析の枠組みは、数値計算全体を貫く考え方である。

## 前提知識 (Prerequisites)

- [`vectors-matrices-linear-maps.md`](vectors-matrices-linear-maps.md) — 行列積の計算構造、直交行列
- [`eigenvalues-diagonalization.md`](eigenvalues-diagonalization.md) — 対称行列と正定値性
- [`svd-low-rank.md`](svd-low-rank.md) — 条件数 κ(A) = σ_max/σ_min

## 本論 (Main Content)

### 1. なぜ逆行列を作らないのか (why not compute the inverse)

`Ax = b` の解は形式上 `x = A⁻¹b` だが、`A⁻¹` を陽に計算するのは (i) 演算量が約3倍、(ii) 丸め誤差が大きく、(iii) A の疎性 (sparsity) を破壊する、と三重に損である (Higham 2002, §14.1)。分解によるアプローチは同じ解をより安く・より正確に与え、さらに**分解の再利用**という決定的な利点を持つ。A を1度分解すれば(O(n³))、異なる右辺 b に対する求解は三角行列の代入計算(O(n²))だけで済む。「係数行列は固定で入力だけ変わる」という状況（回路シミュレーションの時間ステップ、制御の逐次更新）はこの形をしている。

### 2. ガウス消去 = LU 分解 (Gaussian elimination as LU)

中学以来の消去法を行列の言葉で書き直すと、消去の各ステップは下三角行列の乗算であり、全体として

```text
A = LU    L: 単位下三角 (消去で使った係数の記録)
          U: 上三角 (消去後に残る行列)
```

という**分解を計算していたこと**になる (Golub & Van Loan 2013, §3.2)。求解は前進代入 `Ly = b` → 後退代入 `Ux = y` の2段。演算量は分解が約 `2n³/3` flops、代入が `2n²` flops。

素朴な消去はピボット(対角要素)が小さいと係数が爆発して破綻するため、実用では各列で絶対値最大の行を選んで交換する**部分ピボット選択 (partial pivoting)** を行い、`PA = LU`(P は置換行列)を計算する。これが LAPACK の `getrf`、すなわち世界中の「連立一次方程式を解く」ライブラリ呼び出しの実体である。

### 3. Cholesky 分解 (Cholesky factorization)

A が**対称正定値 (symmetric positive definite, SPD)**（`Aᵀ = A` かつすべての固有値が正）のとき、より速く安定な分解が存在する。

```text
A = LLᵀ    L: 対角成分が正の下三角行列 (一意に定まる)
```

演算量は LU の半分(約 `n³/3` flops)で、しかも**ピボット選択が不要**である。SPD 性が係数の爆発を構造的に防ぐためである (Trefethen & Bau 1997, Lecture 23)。共分散行列(統計)、剛性行列(構造解析)、正規方程式の係数 `AᵀA`(§5)など、応用に現れる行列は SPD であることが多く、「SPD だと気づけば計算が半分になる」は実務で最も頻繁に役立つ定理の1つである。逆に、Cholesky 分解の成否は SPD 性の実用的な判定法にもなる。

### 4. QR 分解 (QR factorization)

任意の `m×n` 行列(m ≥ n)は

```text
A = QR    Q: 直交行列 (列が正規直交)
          R: 上三角行列
```

に分解できる。第1回の Gram–Schmidt 直交化は QR 分解の一構成法だが、数値的には直交性が崩れやすい。実用は **Householder 変換**（鏡映 `H = I − 2vvᵀ/(vᵀv)` を繰り返し当てて列を消去する）で、直交性が丸め誤差の範囲で厳密に保たれる (Trefethen & Bau 1997, Lectures 7–10)。演算量は約 `2mn² − 2n³/3` flops。

直交行列は長さを変えない(`‖Qx‖ = ‖x‖`)ため誤差を増幅しない。「問題を直交変換で三角形に持ち込む」のは、悪条件化を避ける数値線形代数の定石であり、固有値計算(QR 法)や SVD の実装もこの上に立つ。

### 5. 最小二乗法 (least squares)

方程式より条件が多い(m > n の)系 `Ax ≈ b` は厳密解を持たず、残差を最小化する

```text
min_x ‖Ax − b‖₂
```

の解を求める。解法は3つあり、速度と安定性のトレードオフをなす (Golub & Van Loan 2013, Ch. 5)。

| 解法 | 演算量 | 数値的性質 |
|------|--------|-----------|
| 正規方程式 `AᵀAx = Aᵀb` を Cholesky で解く | 最速 (`mn² + n³/3`) | 条件数が **κ(A)² に悪化** |
| QR 分解 (`Rx = Qᵀb`) | 中間 (`2mn²`) | κ(A) のまま。標準の選択 |
| SVD (前回) | 最も高価 | ランク落ち・悪条件でも動く最終手段 |

正規方程式の κ² 問題は前回 §5 (PCA) で予告した通りである。`AᵀA` を作った時点で条件数が2乗され、有効桁が半分になる。「数学的に等価な式が数値的には等価でない」ことの代表例である。

### 6. 浮動小数点と誤差解析 (floating-point and error analysis)

IEEE 754 倍精度 (binary64) は実数を `±(1.f) × 2^e`(仮数 52 bit、指数 11 bit)で近似し、1回の四則演算の相対誤差は**単位丸め (unit roundoff)** `u = 2⁻⁵³ ≈ 1.1×10⁻¹⁶` で抑えられる (IEEE 754-2019; Higham 2002, §2.3)。個々の誤差は微小だが、O(n³) 回の演算で蓄積・増幅されうる。その解析の枠組みが**後退誤差解析 (backward error analysis)** である。

```text
計算結果 x̂ は、わずかに摂動した問題の厳密解である:  (A + ΔA)x̂ = b
アルゴリズムが後退安定 ⟺ ‖ΔA‖/‖A‖ が u の小さい定数倍

前進誤差(解の誤差) ≲ κ(A) × 後退誤差(問題の摂動)
```

この不等式が誤差解析の中心定理であり、責任を明確に分離する: **後退誤差はアルゴリズムの品質**(Householder QR や Cholesky は後退安定、部分ピボット LU も実用上安定)、**κ(A) は問題自体の敏感さ**であり、どんな良いアルゴリズムでも救えない。`κ(A) ≈ 10ᵏ` なら約 k 桁失う、が実務の目安になる。倍精度 (u ≈ 10⁻¹⁶) で κ = 10¹⁰ の系を解けば残りは約6桁である。「計算できた」と「信用できる」の距離を定量化するのが条件数である。

## 工学への接続 (Engineering Connections)

- **ライブラリの階層構造**: LAPACK(LU/Cholesky/QR/固有値)が BLAS(行列積 GEMM ほか)の上に組まれ、分解の演算量の大半をキャッシュ効率の良い GEMM に押し込む**ブロック化アルゴリズム**として実装される。第1回 §4 の「行列積の計算構造」が、なぜ GEMM だけ最適化すればライブラリ全体が速くなるのかの答である。
- **疎行列と回路・系統シミュレーション**: SPICE 系回路シミュレータや電力潮流計算は、修正節点方程式/ヤコビアンの**疎 LU 分解**を時間ステップ・反復ごとに再利用する。§1 の「分解の再利用」の産業規模の実例である。
- **混合精度と反復改良 (iterative refinement)**: FP16/BF16 で高速に LU 分解し、倍精度の残差で解を補正して精度を回復する手法は、Tensor Core 等の低精度演算器で倍精度品質を得る HPC の標準技法になっている。§6 の誤差解析がその成立条件(κ が低精度の逆数を超えないこと)を与える。
- **「等価な式」の選択**: 正規方程式 vs QR (§5) と同型の判断（分散の2パス計算 vs 1パス計算、ソフトマックスの安全な実装など）は数値ソフトウェア全域に現れる。式変形の自由度は数学の性質、式の選択は数値解析の責任である。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/cpu/simd-vector.md`](../../../hardware/cpu/simd-vector.md) | 分解のブロック化は SIMD/キャッシュに合わせた行列積への還元。浮動小数点 (IEEE 754) は SIMD レーンのデータ型そのもの |
| [`hardware/accelerators/gpu-simt-architecture.md`](../../../hardware/accelerators/gpu-simt-architecture.md) | Tensor Core の低精度演算を混合精度反復改良で倍精度品質に引き上げる——誤差解析が可能にするハードウェア活用 |
| [`software/ai/ml-inference-systems.md`](../../../software/ai/ml-inference-systems.md) | FP16/BF16/INT8 量子化の精度影響を見積もる枠組み(単位丸め・誤差伝播)の出所 |
| [`engineering/energy/generation-grid-fundamentals.md`](../../../engineering/energy/generation-grid-fundamentals.md) | 電力潮流計算 (Newton–Raphson) の内側は疎 LU 分解の反復再利用 |

## 参考文献 (References)

- Golub, G. H., & Van Loan, C. F. (2013). *Matrix Computations* (4th ed.). Johns Hopkins University Press. — LU/Cholesky/QR・最小二乗法の主参照
- Trefethen, L. N., & Bau, D. (1997). *Numerical Linear Algebra*. SIAM. — Householder QR・安定性の概念の主参照
- Higham, N. J. (2002). *Accuracy and Stability of Numerical Algorithms* (2nd ed.). SIAM. DOI: 10.1137/1.9780898718027 — 後退誤差解析・浮動小数点の主参照
- IEEE. (2019). *IEEE Standard for Floating-Point Arithmetic* (IEEE 754-2019). DOI: 10.1109/IEEESTD.2019.8766229
