# 多変数の微積分 (Multivariable Calculus)

## 概要 (Overview)

微分積分・解析シリーズの第3回(最終回)。変数を多次元に広げると、微積分は2つの方向に花開く。1つは**最適化の言語**——勾配 (gradient)・ヤコビアン・ヘッセ行列という「多変数の導関数」の階層は、機械学習の学習アルゴリズム(勾配降下と誤差逆伝播)の骨格そのものである。もう1つは**場の言語**——空間に分布する量(電場、流れ)の局所的な振る舞いを測る発散 (divergence)・回転 (curl) と、局所と大域を結ぶ積分定理(ガウス・ストークス)は、電磁気学シリーズで積分形のまま留保してきたマクスウェル方程式を微分形で書き直すことを可能にする。

シリーズを貫いてきた主題は「微分=最良の線形近似」だった(第1回)。多変数でもそれは変わらない——勾配・ヤコビアンは線形代数の行列にほかならず、2次の情報(ヘッセ行列)の正定値性は Cholesky 分解(線形代数第4回)で判定される。解析と線形代数という2本のシリーズが、この最終回で完全に噛み合う。

## 前提知識 (Prerequisites)

- [`limits-derivatives-integrals.md`](limits-derivatives-integrals.md) — 微分・テイラー展開・積分
- [`ordinary-differential-equations.md`](ordinary-differential-equations.md) — 安定性と固有値(§3 で対比)
- [`../linear-algebra/vectors-matrices-linear-maps.md`](../linear-algebra/vectors-matrices-linear-maps.md) — 行列 = 線形写像
- [`../linear-algebra/matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) — 正定値性と Cholesky(§3 で使う)

## 本論 (Main Content)

### 1. 偏微分・勾配・ヤコビアン (partial derivatives, gradient, Jacobian)

多変数関数 `f(x₁,…,xₙ)` の**偏微分** ∂f/∂xᵢ は「他の変数を止めて xᵢ だけ動かした変化率」。これを束ねたものが1次の導関数の全体である。

```text
スカラー値 f: ℝⁿ → ℝ   → 勾配 ∇f = (∂f/∂x₁, …, ∂f/∂xₙ)
ベクトル値 F: ℝⁿ → ℝᵐ  → ヤコビアン J (m×n 行列, Jᵢⱼ = ∂Fᵢ/∂xⱼ)

線形近似: f(x+h) ≈ f(x) + ∇f·h,   F(x+h) ≈ F(x) + Jh
```

「微分=最良の線形近似」の多変数版であり、**導関数の実体は行列**になる (Marsden & Tromba 2012, Ch. 2)。勾配の幾何的意味は2つ: (i) ∇f は f が最も急に増える方向を向き、その大きさが増加率である(内積 `∇f·h` を `|h|` 固定で最大化すると h ∥ ∇f)。(ii) ∇f は等高線(等値面)に直交する。物理では「電場は電位の最急降下方向 `E = −∇V`」(電磁気学第1回の微分版)、最適化では「損失を減らしたければ −∇f へ進め」——**勾配降下法 (gradient descent)** の1行の根拠である。

### 2. 連鎖律と誤差逆伝播 (chain rule and backpropagation)

多変数の連鎖律は「線形近似の合成=行列の積」として書ける。

```text
h = g∘f  ⟹  J_h = J_g · J_f   (ヤコビアンの積)
```

深い合成 `f_L∘⋯∘f₂∘f₁`(=ニューラルネットワーク)の勾配は、この積を**どちらの端から掛けるか**で計算量が桁で変わる。出力がスカラー(損失関数)なら、出力側から掛け戻す**リバースモード自動微分**——すなわち**誤差逆伝播 (backpropagation)**——が圧倒的に有利で、全パラメータの勾配が順伝播とほぼ同じコストで得られる (Griewank & Walther 2008)。行列積の結合順序で計算量が変わるという線形代数第5回 §3(縮約順序)と同じ構造が、ここでは「forward vs reverse mode」という形で現れる——深層学習が実用になった計算量的理由はこの1点に集約される。

### 3. 2次近似とヘッセ行列 (second-order approximation and the Hessian)

テイラー展開(第1回 §3)の多変数版は、2次の係数として対称行列を持つ。

```text
f(x+h) ≈ f(x) + ∇f·h + ½ hᵀHh,   H: ヘッセ行列 (Hᵢⱼ = ∂²f/∂xᵢ∂xⱼ)

∇f = 0 の点(臨界点)の分類は H の固有値で決まる:
  全固有値 > 0 (正定値)  → 極小    全固有値 < 0 → 極大
  正負が混在            → 鞍点 (saddle point)
```

1変数の「f'' > 0 なら極小」の一般化だが、多変数では**鞍点**——ある方向には谷、別の方向には尾根——という新しい可能性が生まれ、高次元ではむしろ臨界点の大多数を占める。正定値性の判定は固有値計算のほか **Cholesky 分解の成否**(線形代数第4回 §3)でも行え、Newton 法(`H·Δx = −∇f` を解いて跳ぶ)の内側では毎反復この分解が走る。ヘッセ行列の条件数(線形代数第3回)は「谷の細長さ」であり、悪条件な谷では勾配降下がジグザグして遅くなる——前処理・モーメンタム・適応的学習率という最適化の工夫群は、すべてこの幾何への対処である (Nocedal & Wright 2006, Ch. 3)。

### 4. ベクトル場の局所解析 — 発散と回転 (divergence and curl)

空間の各点にベクトルが付いた**ベクトル場 (vector field)**(電場、磁場、流速)の局所的な振る舞いは、2つの微分演算子で測る (Schey 2005)。

```text
発散 div F = ∇·F = Σ ∂Fᵢ/∂xᵢ   (スカラー) — その点は湧き出し口か吸い込み口か
回転 curl F = ∇×F              (ベクトル) — その点の周りに渦があるか
```

直観は流体で作るのが良い: 発散は「微小体積から正味に流れ出す量」、回転は「その点に置いた微小な風車が回るか」。`curl(∇f) = 0`(勾配場は渦なし)と `div(curl F) = 0`(渦場は湧き出しなし)という2つの恒等式が、場の分類——ポテンシャルを持つ場(静電場)と持たない場(誘導電場)——の代数的な根拠を与える。電磁気学第3回で「誘導起電力は電位では書けない」と述べた事実は、`curl E ≠ 0` の場にはポテンシャルが存在しない、というこの恒等式の帰結である。

### 5. 積分定理 — 局所と大域を結ぶ (the integral theorems)

微積分学の基本定理(第1回 §4)「内部の微分の総和=境界での値」は、次元を上げて2つの定理に一般化される。

```text
ガウスの発散定理:   ∫∫∫_V (∇·F) dV = ∯_∂V F·dA   (体積内の湧き出し合計 = 表面を出る流束)
ストークスの定理:   ∫∫_S (∇×F)·dA = ∮_∂S F·dl    (面上の渦の合計 = 縁に沿う循環)
```

「内部の局所量を足し合わせると、境界の量だけが残る」——この共通構造により、電磁気学の積分形と微分形が**同じ法則の2つの見え方**であることが示せる。マクスウェル方程式(電磁気学第4回)に適用すると次のようになる。

```text
∮E·dA = Q/ε₀      ⟺  ∇·E = ρ/ε₀           (発散定理)
∮B·dA = 0         ⟺  ∇·B = 0
∮E·dl = −dΦ_B/dt  ⟺  ∇×E = −∂B/∂t         (ストークスの定理)
∮B·dl = μ₀I + …   ⟺  ∇×B = μ₀J + μ₀ε₀·∂E/∂t
```

積分形は「任意の面・閉路」についての大域的な言明、微分形は「各点」についての局所的な言明であり、微分形にして初めて偏微分方程式として解ける(波動方程式の導出、数値電磁界解析)。電磁気学シリーズが積分形で留保してきた式は、これで完全な形になった。

## 工学への接続 (Engineering Connections)

- **深層学習の計算構造**: 学習ループの実体は「順伝播(合成関数の評価)→ リバースモード自動微分 (§2) → 勾配降下 (§1) の反復」であり、GPU が加速しているのはこのヤコビアン積の連鎖である。フレームワークの計算グラフは連鎖律のデータ構造化にほかならない。
- **最適化ソルバの内部**: 電力潮流、回路の動作点解析、モデル予測制御、構造最適化——「非線形方程式/最適化を Newton 系で解く」あらゆるソルバは、§3 のヘッセ(またはヤコビアン)+ 線形代数第4回の分解、という同じ2段構造を持つ。
- **数値電磁界・流体解析**: FDTD・FEM は微分形のマクスウェル方程式 (§5) や流体方程式を離散化して解く。アンテナ設計・シグナルインテグリティ解析・EMC シミュレーションの土台であり、「積分形しか知らないと市販ソルバの中身が読めない」の境界線が §5 である。
- **勾配の物理的読み**: 熱流は温度勾配に逆らって流れ (`q = −k∇T`)、拡散流は濃度勾配に従う(半導体第2回の拡散電流)——「勾配に比例した流れ」は輸送現象の普遍文法であり、熱設計・デバイス物理の式が同じ形をしている理由である。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`software/ai/ml-inference-systems.md`](../../../software/ai/ml-inference-systems.md) | 誤差逆伝播 = リバースモード自動微分 (§2)、学習の幾何(条件数とモーメンタム, §3) |
| [`hardware/accelerators/gpu-simt-architecture.md`](../../../hardware/accelerators/gpu-simt-architecture.md) | 学習ワークロードの実体はヤコビアン積の連鎖 (§2) —— GEMM 列に還元される理由 |
| [`../../physics/electromagnetism/maxwell-equations-em-waves.md`](../../physics/electromagnetism/maxwell-equations-em-waves.md) | 積分形 ⟺ 微分形の翻訳 (§5)。第4回で留保した微分形がここで完成 |
| [`../../physics/electromagnetism/charge-field-potential.md`](../../physics/electromagnetism/charge-field-potential.md) | `E = −∇V` (§1)、ガウスの法則の局所化 (§5) |
| [`../../physics/quantum-solid-state/pn-junction-diode.md`](../../physics/quantum-solid-state/pn-junction-diode.md) | 拡散電流 ∝ −∇n ——「勾配に比例した流れ」の輸送文法 (工学への接続 §4) |
| [`engineering/energy/generation-grid-fundamentals.md`](../../../engineering/energy/generation-grid-fundamentals.md) | 潮流計算の Newton 法 = ヤコビアン + 疎 LU (§3, 工学への接続 §2) |

## 参考文献 (References)

- Marsden, J. E., & Tromba, A. (2012). *Vector Calculus* (6th ed.). W. H. Freeman. — 偏微分・勾配・積分定理の主参照
- Schey, H. M. (2005). *Div, Grad, Curl, and All That* (4th ed.). W. W. Norton. — 発散・回転の物理的直観の主参照
- Nocedal, J., & Wright, S. J. (2006). *Numerical Optimization* (2nd ed.). Springer. DOI: 10.1007/978-0-387-40065-5 — ヘッセ行列・Newton 法・条件数と収束の主参照
- Griewank, A., & Walther, A. (2008). *Evaluating Derivatives: Principles and Techniques of Algorithmic Differentiation* (2nd ed.). SIAM. DOI: 10.1137/1.9780898717761 — 自動微分の forward/reverse mode の主参照
- Apostol, T. M. (1969). *Calculus, Vol. 2* (2nd ed.). Wiley. — 多変数の理論的裏付けの副読本
