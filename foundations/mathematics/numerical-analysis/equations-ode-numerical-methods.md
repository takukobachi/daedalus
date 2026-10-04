# 方程式と常微分方程式の数値解法 (Numerical Methods for Equations and ODEs)

## 概要 (Overview)

数値解析シリーズの第2回。前回 ([`floating-point-error-analysis.md`](floating-point-error-analysis.md)) は1回の演算の誤差を扱った。今回は、**有限回の演算では厳密解に届かない問題**を、近似を反復・前進させて解く方法を扱う。対象は工学の計算で最も頻繁に現れる3種類の問題である。

```text
非線形方程式      f(x) = 0                → 求根法 (二分法・ニュートン法)
大規模線形方程式  Ax = b (A は疎で巨大)   → 反復法 (Jacobi・Gauss–Seidel・共役勾配法・Krylov 部分空間法)
初期値問題        dy/dt = f(t, y)         → 時間積分法 (オイラー法・ルンゲ–クッタ法・陰解法・シンプレクティック法)
```

3つは独立した話題に見えるが、実際には入れ子になっている。硬い ODE を陰解法で解くと各ステップで非線形方程式が現れ、それをニュートン法で解くと各反復で線形方程式が現れる。その線形方程式が大規模なら反復法で解く。本回の主題はこの階層を通して2つの概念を区別することである。1つは**収束の速さ (精度・次数)**、もう1つは**誤差が増幅されないこと (安定性)** である。前者はステップを細かくすれば改善するが、後者はステップを細かくしても解決しないことがある。

[`../calculus-analysis/ordinary-differential-equations.md`](../calculus-analysis/ordinary-differential-equations.md) §5 は、オイラー法・ルンゲ–クッタ法と硬い方程式を概観した。本回はそれを、安定領域・A 安定性・ステップ幅制御・幾何学的積分の言葉で掘り下げる。

## 前提知識 (Prerequisites)

- [`floating-point-error-analysis.md`](floating-point-error-analysis.md) — 単位丸め u、条件数、前進/後退誤差
- [`../calculus-analysis/ordinary-differential-equations.md`](../calculus-analysis/ordinary-differential-equations.md) — 初期値問題、線形系 `dx/dt = Ax` の安定性と固有値
- [`../calculus-analysis/limits-derivatives-integrals.md`](../calculus-analysis/limits-derivatives-integrals.md) — テイラー展開(収束次数と局所誤差の導出に使う)
- [`../linear-algebra/matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) — LU/Cholesky 分解、条件数 κ(A)

## 本論 (Main Content)

### 1. 求根法と収束次数 (root finding and order of convergence)

反復列 xₖ → x* の誤差 eₖ = xₖ − x* が

```text
|e_{k+1}| ≈ C |eₖ|^q       (C > 0)
```

を満たすとき、収束次数 (order of convergence) を q と呼ぶ。q = 1(かつ C < 1)を1次収束、q = 2 を2次収束という。1次収束では1反復ごとに一定の桁数が増え、2次収束では**正しい桁数が1反復ごとにほぼ倍になる** (Süli & Mayers 2003, Ch. 1)[^suli]。

**二分法 (bisection)**。f(a) と f(b) の符号が異なる区間を半分にし、符号変化を含む側を残す。中間値の定理により、連続関数なら必ず根に収束する。誤差は1反復で半分になる(1次収束、C = 1/2)ので、許容誤差 tol に達する反復回数は `⌈log₂((b−a)/tol)⌉` と事前に確定する。遅いが、**決して失敗せず、反復回数が予測できる**。この性質はリアルタイム処理で重要になる。

**ニュートン法 (Newton's method)**。f を xₖ で1次近似し、その零点を次の近似とする。

```text
x_{k+1} = xₖ − f(xₖ) / f′(xₖ)
単純根 (f′(x*) ≠ 0) の近くで:  e_{k+1} ≈ [f″(x*) / (2f′(x*))] eₖ²     … 2次収束
```

2次収束は強力で、良い初期値からなら数回の反復で倍精度の限界に達する。ただし代償が3つある。(i) 収束は**局所的**であり、初期値が遠いと発散・振動する。(ii) 導関数が必要になる。(iii) 重根 (f′(x*) = 0) では1次収束に落ちる。導関数を差分商で置き換えた**割線法 (secant method)** の収束次数は黄金比 (1+√5)/2 ≈ 1.618 になる[^suli]。実用のソルバは、二分法の確実さと高次法の速さを組み合わせ、高次のステップが区間を外れたら二分法に戻る、という安全策を取る。

**多変数ニュートン法**。F: ℝⁿ → ℝⁿ に対しては、ヤコビアン J(x) を使って

```text
J(xₖ) Δx = −F(xₖ),    x_{k+1} = xₖ + Δx
```

を反復する。**1反復ごとに n×n の線形方程式を1本解く**ことが、§2 の線形ソルバとの接点になる。ヤコビアンの評価と分解が高価な場合は、J を数反復にわたって使い回す簡易ニュートン法 (simplified Newton) を使う。この場合、収束は1次に落ちるが1反復は安くなる。電力潮流計算、逆運動学、内点法、陰的 ODE 積分のいずれでも、計算の中核はこの反復である。

停止判定にも浮動小数点の事情が絡む。前回の議論から、|f(xₖ)| は丸め誤差のために u·(f の評価に含まれる項の大きさ) 程度より小さくはならない。そのため、許容誤差を u 以下に設定すると反復は終わらない。

### 2. 線形方程式の反復解法 (iterative methods for linear systems)

LU 分解や Cholesky 分解などの直接法は O(n³) の演算と、フィルイン (fill-in) による密化したメモリを要する。偏微分方程式の離散化や回路・系統の方程式では n が 10⁶ を超え、各行の非零要素は数個しかない。このような**疎行列 (sparse matrix)** に対しては、行列ベクトル積 (1回 O(nnz)) だけで解を改善していく反復法が有利になる[^saad]。

**定常反復法 (stationary iterations)**。A = D + L + U(対角・狭義下三角・狭義上三角)と分解する。

```text
Jacobi:        x^{(k+1)} = D⁻¹ (b − (L + U) x^{(k)})
Gauss–Seidel:  x^{(k+1)} = (D + L)⁻¹ (b − U x^{(k)})       … 更新済みの成分をすぐ使う
一般形:        x^{(k+1)} = G x^{(k)} + c,   収束 ⟺ ρ(G) < 1  (ρ: スペクトル半径)
```

誤差は1反復ごとに G 倍される。そのため収束の速さはスペクトル半径 ρ(G) で決まり、1次収束になる。狭義対角優位な行列なら両者とも収束し、対称正定値 (SPD) 行列なら Gauss–Seidel は収束する[^saad]。Jacobi は各成分を独立に更新できるので並列化が容易であり、Gauss–Seidel は逐次依存があるかわりに一般に速い。問題が大きくなるほど ρ(G) は 1 に近づき(2次元ポアソン方程式の格子では 1 − O(h²))、定常反復法単独では実用的な速さが出ない。そのため現在は、後述の前処理やマルチグリッドの平滑化器 (smoother) として使われることが多い。

**Krylov 部分空間法 (Krylov subspace methods)**。初期残差 r₀ = b − Ax₀ から作る部分空間

```text
𝒦ₖ(A, r₀) = span{ r₀, A r₀, A² r₀, …, A^{k−1} r₀ }
```

の中から、各反復で「最良の」近似解を選ぶ方法の総称である。必要な操作は行列ベクトル積だけなので、A を陽に格納しない(作用だけ定義する)行列にも使える (Trefethen & Bau 1997, Lectures 32–40)[^trefethen-bau]。

**共役勾配法 (conjugate gradient, CG)**。A が SPD のとき、CG (Hestenes & Stiefel 1952) は x₀ + 𝒦ₖ の中で A ノルム誤差 ‖e‖_A = √(eᵀAe) を最小化する近似を、短い漸化式(1反復で行列ベクトル積1回、内積2回)で生成する[^hestenes]。これは2次関数 `½xᵀAx − bᵀx` の最小化と等価であり、最適化の勾配法と同じ問題を、探索方向を互いに A 共役に取ることで解いている。

```text
厳密演算なら高々 n 反復で厳密解に到達
誤差限界:  ‖eₖ‖_A ≤ 2 ( (√κ − 1)/(√κ + 1) )ᵏ ‖e₀‖_A,   κ = κ₂(A)
⇒ 誤差を ε 倍にする反復回数 ≈ ½ √κ · ln(2/ε)
```

反復回数は κ ではなく √κ に比例する (Trefethen & Bau 1997, Lecture 38)[^trefethen-bau]。なお、有限精度では探索方向の共役性が丸め誤差で崩れるため、「n 反復で終わる」直接法としては働かない。CG は反復法として、残差が許容値を下回った時点で打ち切って使う。非対称行列には、残差ノルムを 𝒦ₖ 上で最小化する GMRES や BiCGSTAB などを使う[^saad]。

**前処理 (preconditioning)**。収束が κ で決まるなら、κ を小さくすればよい。A に近く、かつ逆を作用させやすい行列 M を用意して、

```text
M⁻¹ A x = M⁻¹ b      (κ(M⁻¹A) ≪ κ(A) を狙う)
```

を解く。M = D (Jacobi 前処理)、不完全 Cholesky/不完全 LU 分解 (IC/ILU: フィルインを捨てた近似分解)、マルチグリッドなどがある。M = A なら1反復で終わるが M⁻¹ の作用が元の問題と同じだけ高価になり、M = I なら安いが効果がない。**前処理の設計はこの両極の間の工学**であり、実務では反復法の選択よりも前処理の選択が性能を決めることが多い (Saad 2003, Ch. 9–10)[^saad]。

### 3. ODE 積分法の基本 — 次数と安定性 (ODE integrators: order and stability)

初期値問題 `y′ = f(t, y), y(t₀) = y₀` を刻み幅 h で前進させる。精度は**局所打ち切り誤差 (local truncation error)**、すなわち1ステップで生じる誤差の大きさで測る。局所誤差が O(h^{p+1}) なら、1/h ステップ分が蓄積して大域誤差 (global error) は O(hᵖ) になる。この p を方法の次数 (order) と呼ぶ (Hairer, Nørsett & Wanner 1993, §II.1–II.3)[^hnw1]。

| 方法 | 更新式 | 次数 | 1ステップの f 評価 |
|------|--------|------|------------------|
| 陽的オイラー (explicit Euler) | `y_{n+1} = yₙ + h f(tₙ, yₙ)` | 1 | 1 |
| 陰的オイラー (implicit Euler) | `y_{n+1} = yₙ + h f(t_{n+1}, y_{n+1})` | 1 | 非線形方程式を解く |
| 台形則 (trapezoidal / Crank–Nicolson) | `y_{n+1} = yₙ + (h/2)[f(tₙ, yₙ) + f(t_{n+1}, y_{n+1})]` | 2 | 非線形方程式を解く |
| 古典的ルンゲ–クッタ (RK4) | 4段の重み付き平均 `yₙ + (h/6)(k₁ + 2k₂ + 2k₃ + k₄)` | 4 | 4 |

収束は精度だけでは保証されない。線形多段法については **Dahlquist の同値定理**が「収束 ⟺ 整合性 (consistency) + ゼロ安定性 (zero-stability)」を示している[^suli]。整合性は「h → 0 で方程式を近似していること」、ゼロ安定性は「誤差が増幅されないこと」を意味する。

安定性を具体的に調べるには、**Dahlquist のテスト方程式** `y′ = λy`(Re λ < 0 で厳密解は減衰)に方法を適用し、z = hλ として `y_{n+1} = R(z) yₙ` の形にする。R(z) を**安定性関数 (stability function)**、|R(z)| ≤ 1 となる z の集合を**安定領域 (stability region)** と呼ぶ[^hw2]。

```text
陽的オイラー:  R(z) = 1 + z           安定領域: |1 + z| ≤ 1  (中心 −1、半径 1 の円板)
陰的オイラー:  R(z) = 1 / (1 − z)     安定領域: |1 − z| ≥ 1  (左半平面全体を含む)
台形則:        R(z) = (1 + z/2)/(1 − z/2)   安定領域: ちょうど左半平面
RK4:           R(z) = 1 + z + z²/2 + z³/6 + z⁴/24   (実軸上は約 −2.79 まで)
```

陽的オイラーで減衰する解 (λ < 0 実数) を解くには、`h < 2/|λ|` が必要になる。これを破ると、厳密解が減衰しているのに数値解は振動しながら発散する。**精度が十分でも、安定性が刻み幅を縛る**のである。

### 4. 硬い方程式と A 安定性 (stiffness and A-stability)

**硬い (stiff)** 方程式とは、ヤコビアンの固有値の実部が大きく散らばった系、つまり非常に速く減衰する成分と、追跡したいゆっくりした成分が共存する系である[^hw2]。

```text
例:  y′ = −1000 (y − cos t)
     解は cos t に沿う滑らかな曲線 (+ 一瞬で消える過渡項 e^{−1000t})
陽的オイラー:  精度上は h ≈ 0.1 で十分なのに、安定性のため h < 2/1000 を強制される
```

過渡項はとっくに消えているのに、その固有値 λ = −1000 が全区間で刻み幅を縛り続ける。化学反応(反応速度の桁違い)、電子回路(寄生容量の小さな時定数)、電力系統(電気的過渡と機械的動揺の共存)、剛性の高い接触を含むロボット・多体系は典型的に硬い。

対策は安定領域の広い方法を使うことである。安定領域が左半平面全体を含む方法を **A 安定 (A-stable)** と呼ぶ。A 安定な方法なら、減衰する成分に対して刻み幅の安定性制約がなくなり、h を精度だけで選べる。陰的オイラーと台形則は A 安定である。さらに `R(z) → 0 (z → −∞)` を満たす方法を **L 安定**と呼び、陰的オイラーは L 安定だが台形則はそうではない(R(−∞) = −1 なので、硬い成分が減衰せずに符号反転しながら残る)[^hw2]。

A 安定性には理論的な限界がある。

- **陽的ルンゲ–クッタ法は A 安定になりえない**: R(z) が多項式なので、|z| → ∞ で必ず |R| → ∞ となる[^hw2]
- **Dahlquist の第2障壁 (second barrier)**: A 安定な線形多段法の次数は高々 2 であり、その中で誤差定数が最小なのは台形則である[^dahlquist]

このため硬い問題では、陰的ルンゲ–クッタ法 (Radau IIA など) や、A 安定性を少し緩めた後退微分公式 (BDF) の可変次数法を使うことになる[^hw2]。陰解法の代償は、各ステップで非線形方程式 `y_{n+1} − h f(y_{n+1}) = …` を解くことである。これを §1 のニュートン法で解くと、各反復で行列 `I − hJ` を係数とする線形方程式が現れ、大規模なら §2 の反復法と前処理で解く。冒頭で述べた入れ子構造がここで完成する。

### 5. 刻み幅制御 (adaptive step-size control)

解の変化の速さは区間ごとに異なるので、刻み幅を固定すると、急な区間に合わせて全体が過剰に細かくなるか、緩やかな区間に合わせて急な区間で精度を失う。実用の ODE ソルバは、**局所誤差を推定し、許容誤差に合わせて h を自動調整する** (Hairer, Nørsett & Wanner 1993, §II.4)[^hnw1]。

標準的な手法は**埋め込み型ルンゲ–クッタ法 (embedded Runge–Kutta pair)** である。同じ段の評価値 kᵢ を共有して、次数 p と p−1 の2つの解を計算し、その差を局所誤差の推定値とする。Dormand & Prince (1980) の 5(4) 次の組は7段 (FSAL: 最終段を次ステップの初段に再利用するため実質6評価) で、多くの汎用ソルバの既定になっている[^dopri]。

```text
err = ‖ y^{[p]} − y^{[p−1]} ‖ / (atol + rtol · |y|)     (成分ごとに尺度化したノルム)
err ≤ 1 なら受理、> 1 なら棄却してやり直し
h_new = h · min( fac_max, max( fac_min, fac · err^{−1/p} ) )     (fac ≈ 0.9 は安全係数)
```

atol(絶対許容誤差)と rtol(相対許容誤差)の組合せは、0 付近の成分と大きな成分を同じ尺度で扱うためにある。注意点が2つある。(i) 制御しているのは**局所誤差**であり、大域誤差は系の安定性(誤差の伝播)次第で許容誤差より大きくなりうる。(ii) h を小さくしすぎると、打ち切り誤差 O(hᵖ) は減るが丸め誤差は 1/h ステップぶん蓄積する。そのため全誤差には最適な h が存在し、要求精度を u 近くまで下げても答は良くならない。

### 6. 幾何学的積分とシンプレクティック法 (geometric integration and symplectic methods)

摩擦のない力学系(惑星の運動、保存力のもとの剛体)は**ハミルトン系 (Hamiltonian system)** であり、エネルギー H(q, p) を保存する。また流れは相空間の体積(より強く、シンプレクティック構造)を保存する。汎用の ODE 解法はこの構造を知らないので、長時間積分では誤差が**系統的に**蓄積する。

```text
調和振動子 q′ = p, p′ = −q  (H = (q² + p²)/2) に陽的オイラーを適用すると:
  H_{n+1} = (1 + h²) Hₙ    → エネルギーが指数的に増大し、軌道は外向きの渦巻きになる
```

**シンプレクティック積分法 (symplectic integrator)** は、離散写像そのものがシンプレクティック性を保つように作られた方法である。代表例の **Störmer–Verlet 法(リープフロッグ法)** は次の形をとる。

```text
p_{n+1/2} = pₙ − (h/2) ∇V(qₙ)
q_{n+1}   = qₙ + h p_{n+1/2}            (H = |p|²/2 + V(q) の場合)
p_{n+1}   = p_{n+1/2} − (h/2) ∇V(q_{n+1})
```

2次の陽解法で、1ステップの力の評価は(前ステップの値を再利用すれば)1回で済み、時間反転対称である。シンプレクティック法が長時間積分で優れる理由は**後退誤差解析**で説明できる。数値解は、元の H に O(hᵖ) の補正を加えた**修正ハミルトニアン (modified Hamiltonian)** H̃ の厳密な流れに、指数的に長い時間にわたって極めて近い。したがって真のエネルギー H の誤差は増大せず、O(hᵖ) の幅で有界に振動する (Hairer, Lubich & Wanner 2006, Ch. IX)[^hlw]。これは前回 §5 の「近い問題を厳密に解く」という後退誤差の考え方の、ODE 版での最も美しい実例である。

注意点として、シンプレクティック性は**固定刻み幅**を前提とする。素朴に刻み幅を適応制御すると性質が失われるため、可変刻み幅には時間変換などの特別な工夫が要る[^hlw]。また、シンプレクティック法は「位相誤差がない」わけではない。軌道の形は保たれるが、軌道上の位置(位相)の誤差は時間に比例して増える。

天体力学では、この考え方を発展させた Wisdom & Holman (1991) の写像法が使われている。ケプラー運動(解析的に解ける部分)と惑星間摂動(小さい部分)にハミルトニアンを分割して交互に厳密に解くことで、太陽系の長期積分を可能にした[^wisdom-holman]。ロボットや多体系のシミュレーションでも同じ問題意識が現れる。回転行列の直交性や単位四元数のノルムといった**拘束 (constraint)** は、汎用解法では少しずつ破れていく。そのため、射影による再正規化や、リー群上で直接積分する方法が使われる[^hlw]。

### 7. 安定性と精度の関係のまとめ (stability versus accuracy)

本回の話題を、精度と安定性の2軸で整理する。

| 文脈 | 精度(速さ)を決めるもの | 安定性(破綻しないこと)を決めるもの |
|------|----------------------|-------------------------------|
| 求根 | 収束次数 (二分法 1、割線法 1.618、ニュートン法 2) | 大域的収束性 (二分法は常に収束、ニュートン法は局所的) |
| 定常反復 | ρ(G) が小さいほど速い | ρ(G) < 1 なら収束 |
| Krylov 法 | √κ に比例する反復回数、前処理 | 有限精度での直交性の喪失、ブレークダウン |
| ODE 積分 | 次数 p、刻み幅 h | 安定領域と hλ の位置関係 (硬さ)、A/L 安定性 |
| 長時間積分 | 局所誤差 | 構造の保存 (シンプレクティック性・拘束) |

すべての行に共通する教訓は、**高次・高速な方法ほど、安定性の前提条件を満たす範囲でしか力を発揮しない**ということである。ニュートン法は初期値が良ければ速く、陽的 RK4 は硬くなければ効率的で、CG は SPD で前処理が効いていれば速い。数値解法の選択とは、問題がその前提を満たすかを見極める作業である。

## 工学への接続 (Engineering Connections)

- **実時間制御は「固定コスト」を優先する**: 1 kHz のサーボループでは、平均的に速いことより**最悪実行時間が周期に収まる**ことが要求される。そのため、組込み制御では固定刻み幅の陽解法や、反復回数に上限を設けたソルバ(MPC の QP を固定反復数で打ち切るなど)が選ばれる。二分法の「反復回数が事前に決まる」性質 (§1) が評価されるのも同じ理由である。
- **制御器の離散化は ODE 積分法の選択そのもの**: 連続時間で設計した制御器を実装する際の前進差分・後退差分・Tustin 変換は、それぞれ陽的オイラー・陰的オイラー・台形則に対応する。台形則が A 安定である (§4) ことは、Tustin 変換が安定な連続系を安定な離散系へ写すことに対応する。
- **最適化ソルバの内側はニュートン法と線形ソルバ**: 内点法は KKT 条件にニュートン法を適用し、各反復で対称な線形方程式を解く。MPC の実時間性は、この線形方程式の構造(帯行列・疎性)をどれだけ活かせるかで決まる (§1–2)。
- **長時間シミュレーションの信頼性は構造保存で決まる**: 軌道伝播、分子動力学、物理エンジンでは、局所誤差の小ささより、エネルギーや拘束が系統的にずれないこと (§6) が長期的な結果の妥当性を支配する。
- **系統解析は硬い ODE と大規模非線形方程式の両方を含む**: 電力潮流は多変数ニュートン法と疎 LU 分解の反復で解かれ、過渡安定度解析は電気的な速い時定数と機械的な遅い動揺が共存する硬い系の時間積分になる (§1, §4)。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`../calculus-analysis/ordinary-differential-equations.md`](../calculus-analysis/ordinary-differential-equations.md) | 数値解法と硬い方程式の概観 (§5) を、安定領域・A 安定性・刻み幅制御 (§3–5) で掘り下げる |
| [`engineering/aerospace/orbital-mechanics.md`](../../../engineering/aerospace/orbital-mechanics.md) | J2 などの摂動を含む運動方程式の数値積分としての軌道伝播 (§5)、長時間積分での構造保存 (§6) |
| [`engineering/aerospace/avionics-gnc.md`](../../../engineering/aerospace/avionics-gnc.md) | 着陸誘導の機上内点法ソルバ(内側のニュートン反復と線形ソルバ, §1–2) |
| [`engineering/robotics/kinematics-dynamics.md`](../../../engineering/robotics/kinematics-dynamics.md) | 数値逆運動学のニュートン–ラフソン反復 (§1)、順動力学の時間積分と回転の再直交化 (§6) |
| [`engineering/robotics/control-systems.md`](../../../engineering/robotics/control-systems.md) | 制御周期内で解く MPC の数値ソルバ、制御器の離散化と固定コスト要求(工学への接続) |
| [`../optimization/convex-optimization-duality.md`](../optimization/convex-optimization-duality.md) | ニュートン法・勾配法と条件数 (§1–2)。CG は2次関数の最小化と等価 |
| [`engineering/energy/generation-grid-fundamentals.md`](../../../engineering/energy/generation-grid-fundamentals.md) | 動揺方程式の時間積分と過渡安定度 (§3–4)、潮流計算のニュートン法 (§1) |
| [`engineering/energy/electricity-markets-dispatch.md`](../../../engineering/energy/electricity-markets-dispatch.md) | 経済負荷配分 (ED) の凸最適化を解くソルバの内側(ニュートン反復と疎線形ソルバ, §1–2) |

## 参考文献 (References)

[^suli]: Süli, E., & Mayers, D. F. (2003). *An Introduction to Numerical Analysis*. Cambridge University Press. DOI: 10.1017/CBO9780511801181 — 求根法と収束次数 (Ch. 1)、ODE の整合性・ゼロ安定性と Dahlquist の同値定理 (Ch. 12)。
[^saad]: Saad, Y. (2003). *Iterative Methods for Sparse Linear Systems* (2nd ed.). SIAM. DOI: 10.1137/1.9780898718003 — 定常反復法 (Ch. 4)、Krylov 部分空間法・CG・GMRES (Ch. 6–7)、前処理 (Ch. 9–10) の主参照。
[^trefethen-bau]: Trefethen, L. N., & Bau, D. (1997). *Numerical Linear Algebra*. SIAM. DOI: 10.1137/1.9780898719574 — Krylov 部分空間法 (Lectures 32–40)、CG の収束限界 (Lecture 38)。
[^hestenes]: Hestenes, M. R., & Stiefel, E. (1952). Methods of Conjugate Gradients for Solving Linear Systems. *Journal of Research of the National Bureau of Standards*, 49(6), 409–436. DOI: 10.6028/jres.049.044 — 共役勾配法の原典。
[^hnw1]: Hairer, E., Nørsett, S. P., & Wanner, G. (1993). *Solving Ordinary Differential Equations I: Nonstiff Problems* (2nd ed.). Springer. DOI: 10.1007/978-3-540-78862-1 — ルンゲ–クッタ法の次数と誤差 (§II.1–II.3)、埋め込み公式と刻み幅制御 (§II.4)。
[^hw2]: Hairer, E., & Wanner, G. (1996). *Solving Ordinary Differential Equations II: Stiff and Differential-Algebraic Problems* (2nd ed.). Springer. DOI: 10.1007/978-3-642-05221-7 — 硬い方程式、安定性関数、A/L 安定性、陰的 RK・BDF の主参照。
[^dahlquist]: Dahlquist, G. G. (1963). A Special Stability Problem for Linear Multistep Methods. *BIT*, 3(1), 27–43. DOI: 10.1007/BF01963532 — A 安定性の定義と第2障壁。
[^dopri]: Dormand, J. R., & Prince, P. J. (1980). A Family of Embedded Runge–Kutta Formulae. *Journal of Computational and Applied Mathematics*, 6(1), 19–26. DOI: 10.1016/0771-050X(80)90013-3 — 5(4) 次の埋め込み公式。
[^hlw]: Hairer, E., Lubich, C., & Wanner, G. (2006). *Geometric Numerical Integration: Structure-Preserving Algorithms for Ordinary Differential Equations* (2nd ed.). Springer. DOI: 10.1007/3-540-30666-8 — シンプレクティック法、Störmer–Verlet 法、修正ハミルトニアンによる後退誤差解析 (Ch. IX)、可変刻み幅と拘束の扱い。
[^wisdom-holman]: Wisdom, J., & Holman, M. (1991). Symplectic Maps for the n-Body Problem. *The Astronomical Journal*, 102, 1528. DOI: 10.1086/115978 — ケプラー運動と摂動の分割による n 体問題のシンプレクティック写像。
