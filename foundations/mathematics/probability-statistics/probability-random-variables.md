# 確率空間と確率変数 (Probability Spaces and Random Variables)

## 概要 (Overview)

確率・統計シリーズの第1回。分岐予測の「ヒット率 95%」、歩留まりの「欠陥密度 0.1/cm²」、SLO の「p99 レイテンシ」——Daedalus の分析には確率の言葉が至る所に現れるが、それらが**正確には何を意味するのか**を定義できて初めて、計算し、合成し、反証できる。

本回は確率論の骨格を4段で積み上げる: (1) 確率とは何か——コルモゴロフの公理、(2) 情報が確率を変える仕組み——条件付き確率と独立、(3) ランダムな量の扱い方——確率変数と期待値・分散、(4) 現実に頻出する分布のカタログ。最後に、この理論全体を「測定」と接続する2大定理——**大数の法則**(平均は収束する)と**中心極限定理**(和は正規分布に近づく)——で締める。この2つが「有限回の測定から母集団を語る」統計学の存在根拠である。

## 前提知識 (Prerequisites)

- 集合の記法と Σ 記号(高校数学レベル)
- [`../linear-algebra/eigenvalues-diagonalization.md`](../linear-algebra/eigenvalues-diagonalization.md) —
  §「確率行列と定常分布」。本シリーズ第2回(マルコフ連鎖)で合流する

## 本論 (Main Content)

### 1. 確率空間 (probability space)

確率論の全体は3つ組 `(Ω, F, P)` の上に立つ (Kolmogorov 1933; Feller 1968, Ch. 1):

```text
Ω: 標本空間 (起こりうる結果すべての集合。例: サイコロなら {1,…,6})
F: 事象の族 (Ω の部分集合のうち「確率を問える」もの)
P: 確率測度 — 各事象 A に数 P(A) を割り当て、次の公理を満たす:
   (i)  P(A) ≥ 0
   (ii) P(Ω) = 1
   (iii) 互いに排反な事象について P(A₁ ∪ A₂ ∪ ⋯) = ΣP(Aᵢ) (可算加法性)
```

重要なのは、公理が「確率とは何か」の哲学(頻度か信念か)を**一切決めていない**ことである。頻度(分岐予測のヒット率)も信念(この仮説が正しい確度)も、同じ公理系・同じ計算規則に従う——だから1つの理論で両方を扱える。加法性から導かれる基本規則(補事象 `P(Aᶜ) = 1 − P(A)`、包除 `P(A∪B) = P(A) + P(B) − P(A∩B)`)だけで、かなりの実務計算が済む。

### 2. 条件付き確率と独立 (conditional probability and independence)

「B が起きたと分かった」という情報は確率を更新する:

```text
条件付き確率: P(A|B) = P(A∩B) / P(B)
ベイズの定理: P(A|B) = P(B|A)·P(A) / P(B)
独立:         P(A∩B) = P(A)·P(B)  (B を知っても A の確率が変わらない)
```

ベイズの定理は「原因→結果の確率」から「結果→原因の確率」への反転であり、診断・フィルタリング・推定のすべての基礎になる (Bertsekas & Tsitsiklis 2008, §1.4)。実務上の最重要注意は**独立性は仮定であって自明ではない**こと。「99.9% 信頼のコンポーネントを2重化すれば故障率は 10⁻⁶」という計算は独立性を仮定しており、共通原因故障(同一ロット、同一電源、同一バグ)があれば成り立たない——冗長設計の見積りが楽観に倒れる典型的な理由である。

### 3. 確率変数・期待値・分散 (random variables, expectation, variance)

**確率変数 (random variable)** X は結果に数を割り当てる関数 `X: Ω → ℝ` であり、分布はその確率の配り方である(離散なら確率質量関数 pmf、連続なら確率密度関数 pdf)。分布を1つの数に要約するのが:

```text
期待値: E[X] = Σ x·p(x)  または ∫ x·f(x)dx   (分布の重心)
分散:   Var(X) = E[(X − E[X])²]              (ばらつきの2乗平均)
標準偏差: σ = √Var(X)                        (X と同じ単位のばらつき)
```

計算を支える柱は**期待値の線形性** `E[aX + bY] = aE[X] + bE[Y]` ——これは X, Y が独立でなくても常に成り立つ(分散の加法 `Var(X+Y) = Var(X) + Var(Y)` は独立が要る、と対比して覚える)。複雑な量の期待値を、指示変数の和に分解して線形性で計算するのは確率計算の常套手段である。

実務では平均だけでは足りない——レイテンシ分析が平均でなく**分位点 (percentile)** p99 を見るのは、分布の裾 (tail) にこそユーザー影響が集中するからである。分布全体を語るには累積分布関数 `F(x) = P(X ≤ x)` を使い、p99 は `F(x) = 0.99` の解として定義される。

### 4. 主要分布のカタログ (catalog of distributions)

応用に現れる分布は少数の「生成メカニズム」から出てくる (Feller 1968; Ross 2019):

| 分布 | 生成メカニズム | E[X] | 典型的応用 |
|------|--------------|------|-----------|
| ベルヌーイ Bern(p) | 成功/失敗の1回試行 | p | 分岐の taken/not-taken、ビット誤り |
| 二項 B(n,p) | 独立な n 回試行の成功数 | np | n 個中の不良チップ数 |
| 幾何 Geom(p) | 初めての成功までの試行数 | 1/p | 再送回数、リトライ |
| ポアソン Pois(λ) | 稀な事象の一定時間内の生起数 | λ | 単位面積あたり欠陥数、到着リクエスト数 |
| 指数 Exp(λ) | ポアソン事象の間隔 | 1/λ | 故障までの時間、サービス時間モデル |
| 正規 N(μ,σ²) | 多数の独立要因の和 (§5) | μ | 測定誤差、プロセスばらつき |

見どころは分布間の関係である。二項分布は `n→∞, np=λ 一定` の極限でポアソン分布になる(稀な事象の法則)——ウェハ上の欠陥数モデルが二項でなくポアソンで書かれる理由。指数分布は**無記憶性 (memorylessness)** `P(X > s+t | X > s) = P(X > t)` を持つ唯一の連続分布であり、「使い込んでも故障しやすくならない」偶発故障期のモデルとして機能する一方、摩耗故障には原理的に合わない——モデル選択の判断がここに宿る。

### 5. 大数の法則と中心極限定理 (LLN and CLT)

独立同分布 (i.i.d.) の確率変数列 X₁, X₂, … (平均 μ、分散 σ²)の標本平均 `X̄ₙ = (X₁+⋯+Xₙ)/n` について:

```text
大数の法則 (LLN):     X̄ₙ → μ  (n→∞ で確率的に収束)
中心極限定理 (CLT):   (X̄ₙ − μ)/(σ/√n) → N(0,1)  (分布が標準正規に収束)
```

LLN は「測定を繰り返せば真値に近づく」ことの保証——頻度と確率を結ぶ蝶番であり、モンテカルロ法の正当化でもある。CLT はさらに強く、**収束の速さと誤差の形**まで与える: 標本平均の誤差は `σ/√n` のスケールで、形は正規分布。ここから実務の基本律「精度を1桁上げるには測定回数が100倍要る」(√n 則)が出る。また CLT は「多数の独立微小要因の和は正規分布になる」ことを言うので、測定ノイズや製造ばらつきが正規分布でモデルされることの理論的根拠にもなる——ただし**裾の重い分布 (heavy-tailed)** には収束が遅い・分散が無限で適用不能などの限界があり、レイテンシのような裾の重い量に正規近似を使うのは典型的な誤用である (Ross 2019, Ch. 8)。

## 工学への接続 (Engineering Connections)

- **ヒット率と期待コストの計算**: 「ヒット率 h、ヒット時コスト c₁、ミス時 c₂」の期待コスト `hc₁ + (1−h)c₂` は期待値の線形性の直接の応用であり、キャッシュ・分岐予測・メモリ階層の平均性能式(AMAT など)はすべてこの形をしている。
- **歩留まりモデル**: ダイ面積 A、欠陥密度 D のとき、ポアソンモデルは歩留まり `Y = e^{−AD}` を与える。「ダイを2倍にすると歩留まりが指数関数的に落ちる」——チップレット化の経済的根拠は §4 の分布論そのものである。
- **テイルレイテンシとファンアウト**: 1台の p99 が良くても、100 台に並列問い合わせて全応答を待つと約 63%(= 1 − 0.99¹⁰⁰)の確率でどれかが p99 より遅い——独立性の計算が「テイルの増幅」を定量化し、ヘッジリクエスト等の対策設計を導く。
- **測定の設計**: ベンチマークの反復回数、信頼区間、外れ値の扱いは LLN/CLT の直接の応用。「何回測れば有意か」に答えられることが、性能主張の再現性を支える。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/cpu-architecture/branch-prediction-speculation.md`](../../../hardware/cpu-architecture/branch-prediction-speculation.md) | 予測ヒット率と誤予測ペナルティの期待値計算。予測器の学習は条件付き確率の推定 |
| [`hardware/cpu-architecture/cache-memory-hierarchy.md`](../../../hardware/cpu-architecture/cache-memory-hierarchy.md) | AMAT = ヒット率で重み付けた期待アクセス時間——期待値の線形性の実例 |
| [`hardware/semiconductor-manufacturing/semiconductor-fabrication.md`](../../../hardware/semiconductor-manufacturing/semiconductor-fabrication.md) | 欠陥密度のポアソンモデルと歩留まり式、プロセスばらつきの正規モデル |
| [`software/observability/observability-performance-engineering.md`](../../../software/observability/observability-performance-engineering.md) | 分位点 (p50/p99) の定義、テイルレイテンシの合成、ベンチマークの統計的設計 |
| [`hardware/automotive/functional-safety-iso26262.md`](../../../hardware/automotive/functional-safety-iso26262.md) | 故障率 (FIT)・冗長系の信頼度計算と、その独立性仮定の限界(共通原因故障) |

## 参考文献 (References)

- Feller, W. (1968). *An Introduction to Probability Theory and Its Applications, Vol. 1* (3rd ed.). Wiley. — 公理・主要分布・極限定理の主参照
- Bertsekas, D. P., & Tsitsiklis, J. N. (2008). *Introduction to Probability* (2nd ed.). Athena Scientific. — 条件付き確率・ベイズ・期待値の教育的主参照
- Ross, S. M. (2019). *A First Course in Probability* (10th ed.). Pearson. — 分布カタログ・CLT とその限界の副読本
- Kolmogorov, A. N. (1933). *Grundbegriffe der Wahrscheinlichkeitsrechnung*. Springer. — 公理系の原典(英訳: *Foundations of the Theory of Probability*, Chelsea, 1956)
