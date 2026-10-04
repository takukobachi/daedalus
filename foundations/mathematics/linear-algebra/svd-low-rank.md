# 特異値分解と低ランク近似 (Singular Value Decomposition and Low-Rank Approximation)

## 概要 (Overview)

線形代数シリーズの第3回。前回の対角化 `A = PDP⁻¹` は強力だが、正方行列で、かつ固有ベクトルが揃う場合にしか使えない。**特異値分解 (singular value decomposition, SVD)** はこの制約を完全に取り払う——**任意の**(長方形でもランク落ちでもよい)行列が

```text
A = U Σ Vᵀ    (直交 × 非負対角 × 直交)
```

に分解できる。幾何的に言えば、**どんな線形写像も「回転 → 軸ごとの伸縮 → 回転」の3段に分解できる**。この分解が与える特異値の減衰列は「行列の情報がどの方向にどれだけ詰まっているか」の定量化であり、途中で打ち切れば**低ランク近似 (low-rank approximation)**——最適な圧縮——になる。次元削減 (PCA)、レコメンデーション、機械学習の低ランク適応 (LoRA) は、すべてこの1つの定理の応用である。

## 前提知識 (Prerequisites)

- [`vectors-matrices-linear-maps.md`](vectors-matrices-linear-maps.md) — 正規直交基底、階数
- [`eigenvalues-diagonalization.md`](eigenvalues-diagonalization.md) — 対角化、スペクトル定理(対称行列の直交対角化)

## 本論 (Main Content)

### 1. 定義と幾何的意味 (definition and geometric meaning)

任意の `m×n` 実行列 A は次の形に分解できる (Golub & Van Loan 2013, §2.4)。

```text
A = U Σ Vᵀ
  U: m×m 直交行列 (列 = 左特異ベクトル, left singular vectors)
  Σ: m×n 対角行列, 対角成分 σ₁ ≥ σ₂ ≥ ⋯ ≥ 0 (特異値, singular values)
  V: n×n 直交行列 (列 = 右特異ベクトル, right singular vectors)
```

存在はすべての行列について保証され、例外はない。幾何的な読み方: A は単位球面を**楕円体 (ellipsoid)** に写す。右特異ベクトル vᵢ が「入力側の主軸」、σᵢ がその軸の伸縮率、左特異ベクトル uᵢ が「出力側で楕円体の軸が向く方向」である。つまり `Av_i = σ_i u_i` ——正規直交な入力基底が、正規直交な出力基底へ、それぞれ σᵢ 倍されて写る。ゼロでない特異値の個数が**階数 (rank)** に一致する。

### 2. スペクトル定理からの構成 (construction via the spectral theorem)

SVD は前回のスペクトル定理から直接作れる。`AᵀA` は対称かつ半正定値なので、直交行列で対角化でき、固有値は非負である。

```text
AᵀA = V (ΣᵀΣ) Vᵀ    → 固有値 = σᵢ², 固有ベクトル = 右特異ベクトル vᵢ
AAᵀ = U (ΣΣᵀ) Uᵀ    → 同じ非ゼロ固有値, 固有ベクトル = 左特異ベクトル uᵢ
```

「任意の行列 A 自体は対角化できなくても、`AᵀA` は必ず直交対角化できる」——非対称・長方形の世界の問題を、対称行列の良い世界に持ち込んで解く、という構図である。固有値分解との関係を整理すると: 対称行列では SVD と固有値分解は(符号を除き)一致し、一般の正方行列では固有値が「繰り返し適用の漸近挙動」(前回)を、特異値が「1回の適用での増幅率」を支配する、と役割が分かれる。

### 3. 低ランク近似とEckart–Young の定理 (low-rank approximation)

SVD は A を**ランク1行列の和**として書き直せる。

```text
A = σ₁u₁v₁ᵀ + σ₂u₂v₂ᵀ + ⋯ + σᵣuᵣvᵣᵀ    (r = rank A)
```

特異値は降順なので、これは「重要な成分から順に並べた展開」である。先頭 k 項で打ち切った `A_k` について、**Eckart–Young の定理** (Eckart & Young 1936) が最適性を保証する。

```text
A_k = Σᵢ₌₁ᵏ σᵢuᵢvᵢᵀ は、ランク k 以下の全行列の中で ‖A − B‖ を最小化する
      (スペクトルノルム・フロベニウスノルムの双方で)
```

つまり **SVD の打ち切りは、ランク k という制約下での最良の近似**であり、しかも誤差は捨てた特異値で明示的に分かる(スペクトルノルムで σ_{k+1})。ストレージは `mn` 要素から `k(m+n+1)` 要素へ落ちる。特異値が速く減衰する行列——自然画像、レコメンデーションの評価行列、言語モデルの重み更新——ほど、少ない k で良く近似できる。「データに低ランク構造があるか」は「特異値スペクトルが速く落ちるか」と同義である。

### 4. 条件数 (condition number)

正方可逆行列に対し、最大・最小特異値の比

```text
κ(A) = σ_max / σ_min
```

を**条件数 (condition number)** と呼ぶ。`Ax = b` を解くとき、入力 b の相対誤差は解 x では最大 κ 倍に増幅される (Trefethen & Bau 1997, Lecture 12)。κ が大きい(**悪条件, ill-conditioned**)系では、アルゴリズムがいくら正確でも、丸め誤差・測定誤差が解を支配する。「この計算は数値的に信用できるか」という問いの一次判定量であり、次回の行列分解・浮動小数点誤差の回で主役になる。

### 5. 主成分分析 (principal component analysis, PCA)

各行が1サンプルのデータ行列 X(列平均を引いて中心化済み)に対し、共分散行列 `XᵀX/(n−1)` の固有ベクトル=X の右特異ベクトルが**主成分 (principal components)**、すなわち「データの分散が最大の直交方向」である (Jolliffe 2002, Ch. 1, 3)。上位 k 主成分への射影が、分散を最も保つ k 次元圧縮を与える——§3 の低ランク近似の統計版である。実装上も、共分散行列を作らず X を直接 SVD する方が数値的に安定(条件数が κ² にならない)という、§4 がそのまま効く実例になっている。

## 工学への接続 (Engineering Connections)

- **圧縮の理論限界**: Eckart–Young は「ランク制約下でこれ以上は縮まない」という下限を与える。埋め込みベクトルの次元削減、重み行列の圧縮・蒸留の理論的裏付け。
- **低ランク適応 (LoRA)**: 大規模モデルのファインチューニングで、重み更新 ΔW を低ランク積 BA に制約する手法は「更新は低ランクで十分」という仮説の直接の工学化であり、学習・保存するパラメータを k(m+n)/mn に削減する。
- **計算量の削減**: ランク k 近似を使えば行列ベクトル積は O(mn) → O(k(m+n))。GEMM の計算量そのものを問題側から削る手段である。
- **数値的信頼性の物差し**: 条件数は、回路シミュレーション・制御設計・最適化ソルバなど「連立方程式を解く」あらゆる場面で、結果の有効桁を見積もる共通言語になる。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`software/ai/`](../../../software/ai/) | 埋め込みの次元削減、モデル圧縮、LoRA。学習パイプラインに現れる低ランク構造の数学的根拠 |
| [`hardware/accelerators/gpu-simt-architecture.md`](../../../hardware/accelerators/gpu-simt-architecture.md) | 低ランク化は GEMM のサイズ自体を削減する——演算器を速くする(第1回 §4)のと相補的な、問題側からの高速化 |

## 参考文献 (References)

- Golub, G. H., & Van Loan, C. F. (2013). *Matrix Computations* (4th ed.). Johns Hopkins University Press. — SVD の存在・構成・数値計算の主参照
- Trefethen, L. N., & Bau, D. (1997). *Numerical Linear Algebra*. SIAM. — 特異値・条件数の幾何的説明の主参照
- Strang, G. (2016). *Introduction to Linear Algebra* (5th ed.). Wellesley-Cambridge Press. — SVD と応用の副読本
- Eckart, C., & Young, G. (1936). The approximation of one matrix by another of lower rank. *Psychometrika*, 1(3), 211–218. DOI: 10.1007/BF02288367
- Jolliffe, I. T. (2002). *Principal Component Analysis* (2nd ed.). Springer. DOI: 10.1007/b98835
