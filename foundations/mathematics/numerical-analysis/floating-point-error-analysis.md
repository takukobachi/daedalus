# 浮動小数点演算と誤差解析 (Floating-Point Arithmetic and Error Analysis)

## 概要 (Overview)

数値解析シリーズの第1回。計算機は実数を扱えない。扱えるのは有限個の**浮動小数点数 (floating-point number)** だけであり、四則演算のたびに結果は最も近い表現可能な数へ**丸め (rounding)** られる。1回の丸め誤差は倍精度で 10⁻¹⁶ 程度にすぎないが、問題によってはこの誤差が何桁にも増幅され、また総和の順序を変えただけで結果のビット列が変わる。本回の目標は、この「小さな誤差がいつ、なぜ大きくなるのか」を定量的に語れるようにすることである。

構成は次の3段である。(1) **表現**: IEEE 754-2019 の形式、丸めモード、機械イプシロンと単位丸め、非正規化数・NaN・Inf、そして機械学習が持ち込んだ bfloat16 と FP8。(2) **解析**: 桁落ち、前進誤差と後退誤差、条件数、そして Higham 流の誤差解析を総和に適用し、再帰総和・ペアワイズ総和・Kahan の補償総和の誤差限界を導く。(3) **工学**: FMA、非結合性と並列縮約の再現性、混合精度学習と損失スケーリング。

線形代数シリーズの [`matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) §6 は、後退誤差解析を連立一次方程式に限って紹介した。本回はその土台である浮動小数点そのものへ降りていき、枠組みを一般の計算へ広げる。

## 前提知識 (Prerequisites)

- [`../calculus-analysis/limits-derivatives-integrals.md`](../calculus-analysis/limits-derivatives-integrals.md) — 微分と線形近似(条件数の定義に使う)
- [`../linear-algebra/matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) — 条件数 κ(A)、後退安定性の初出
- 2進数と整数のビット表現(固定小数点との違いを理解する程度)

## 本論 (Main Content)

### 1. 浮動小数点数の構造 (anatomy of a floating-point number)

正規化された2進浮動小数点数は、符号 s、指数 e、精度 p ビットの仮数 (significand) を使って次のように表す。

```text
x = (−1)^s × (1.f₁f₂…f_{p−1})₂ × 2^e,    e_min ≤ e ≤ e_max
格納されるのは s・バイアス付き指数・小数部 f (先頭の 1 は暗黙: hidden bit)
```

表現可能な数は等間隔ではない。区間 [2^e, 2^{e+1}) の中には 2^{p−1} 個の数が等間隔に並び、隣の区間へ移るたびに間隔が2倍になる。このため**相対的な**間隔はどの桁でもほぼ一定であり、これが浮動小数点の本質である。絶対誤差ではなく相対誤差を一定に保つ表現だと言える (Goldberg 1991)[^goldberg]。

IEEE 754-2019 は2進交換形式として binary16/32/64/128 を定める[^ieee754]。機械学習が持ち込んだ非 IEEE の形式も並べると、次の表になる。

| 形式 | 符号/指数/小数部 (bit) | 精度 p | 最大有限値 | 最小正規化数 | 単位丸め u | 出典 |
|------|----------------------|--------|-----------|-------------|-----------|------|
| binary64 (倍精度) | 1 / 11 / 52 | 53 | ≈ 1.80×10³⁰⁸ | 2⁻¹⁰²² ≈ 2.2×10⁻³⁰⁸ | 2⁻⁵³ ≈ 1.1×10⁻¹⁶ | IEEE 754 |
| binary32 (単精度) | 1 / 8 / 23 | 24 | ≈ 3.40×10³⁸ | 2⁻¹²⁶ ≈ 1.2×10⁻³⁸ | 2⁻²⁴ ≈ 6.0×10⁻⁸ | IEEE 754 |
| binary16 (半精度, FP16) | 1 / 5 / 10 | 11 | 65504 | 2⁻¹⁴ ≈ 6.1×10⁻⁵ | 2⁻¹¹ ≈ 4.9×10⁻⁴ | IEEE 754 |
| bfloat16 (BF16) | 1 / 8 / 7 | 8 | ≈ 3.39×10³⁸ | 2⁻¹²⁶ | 2⁻⁸ ≈ 3.9×10⁻³ | 非 IEEE[^kalamkar] |
| FP8 E5M2 | 1 / 5 / 2 | 3 | 57344 | 2⁻¹⁴ | 2⁻³ = 0.125 | OCP OFP8[^ocp-fp8][^fp8] |
| FP8 E4M3 | 1 / 4 / 3 | 4 | 448 | 2⁻⁶ | 2⁻⁴ = 0.0625 | OCP OFP8[^ocp-fp8][^fp8] |

この表から形式設計の基本的なトレードオフが読める。**指数ビットはダイナミックレンジを、小数部ビットは精度を買う**。bfloat16 は binary32 の上位16ビットそのもので、指数を binary32 と同じ 8 ビットに保ったまま精度を 8 ビットまで削った。そのため binary32 と同じ値域を持ち、変換は切り詰めと丸めだけで済む[^kalamkar]。FP16 は逆に精度を残して値域を削ったため、§7 で扱う勾配のアンダーフローが起きる。

FP8 の2形式は役割で使い分ける。Micikevicius et al. (2022) は重みと活性化に E4M3、勾配に E5M2 を推奨した[^fp8]。E5M2 は IEEE 754 の特殊値の規約(Inf と NaN)にそのまま従い、「仮数を削った binary16」とみなせる。E4M3 は無限大を持たず、NaN のビットパターンを1つ(S.1111.111)に絞った。空いたパターンを正規化数に回すことで、最大値を 240 から 448 へ広げている[^fp8]。この2形式は OCP (Open Compute Project) の 8-bit Floating Point Specification (OFP8) として業界仕様にまとめられた[^ocp-fp8]。

### 2. 丸めと標準モデル (rounding and the standard model)

IEEE 754 は、四則演算・平方根・FMA (§6) について**正しく丸められた (correctly rounded)** 結果、すなわち無限精度で計算した値を指定の丸めモードで丸めた結果を返すことを要求する[^ieee754]。丸め方向は5種類ある。

| 丸め方向属性 | 意味 | 主な用途 |
|-------------|------|---------|
| roundTiesToEven | 最近接。中間値なら偶数側へ | 2進形式の既定。統計的な偏りがない |
| roundTiesToAway | 最近接。中間値なら絶対値の大きい側へ | 10進形式での利用を想定 |
| roundTowardPositive | +∞ 方向 | 区間演算の上界 |
| roundTowardNegative | −∞ 方向 | 区間演算の下界 |
| roundTowardZero | 0 方向(切り捨て) | 整数変換、一部の低精度ハードウェア |

最近接丸めの下で、精度 p の形式に関する2つの量を区別する。

```text
機械イプシロン (machine epsilon)  ε = 2^{1−p}   … 1 と、1 の次に大きい浮動小数点数との差
単位丸め (unit roundoff)          u = 2^{−p} = ε/2 … 丸めの相対誤差の上限
```

文献によって「machine epsilon」を u の意味で使うものがあり(C の `DBL_EPSILON` は ε = 2⁻⁵²)、係数2の食い違いがよく起きる。本シリーズは Higham に従って u を誤差限界に使う[^higham]。

正しい丸めから、誤差解析のすべての出発点となる**標準モデル (standard model)** が得られる (Higham 2002, §2.2)[^higham]。

```text
fl(x op y) = (x op y)(1 + δ),   |δ| ≤ u,    op ∈ {+, −, ×, ÷}
fl(√x)     = √x (1 + δ),        |δ| ≤ u
(オーバーフロー・アンダーフローが起きない範囲で成立)
```

「計算結果は真値の相対 u 以内にある」というこの一文が、ハードウェアの詳細を捨象して算法の誤差を論じるための公理になる。

### 3. 非正規化数・無限大・NaN (subnormals, infinities, NaN)

指数が最小値に達すると、暗黙の先頭ビットを 0 にした**非正規化数 (subnormal number)** が 0 と最小正規化数の間を等間隔で埋める。binary64 では最小の非正規化数が 2⁻¹⁰⁷⁴ になる。これにより**段階的アンダーフロー (gradual underflow)** が成立し、`x ≠ y ⇒ x − y ≠ 0` が保証される (Goldberg 1991)[^goldberg]。非正規化数がなく 0 へ突然落ちる「flush-to-zero」では、この性質が崩れて `if (x != y) z = 1/(x−y)` のような安全確認が無意味になる。ただし多くの実装では非正規化数の演算がマイクロコードや例外処理に回って遅くなるため、性能重視の環境では FTZ/DAZ モードで意図的に切り捨てることがある。

特殊値は例外を「止めずに伝播させる」ための装置である。

- **±∞**: オーバーフローと `1/0` の結果。`1/∞ = 0` のように算術が自然に続く
- **NaN (Not a Number)**: `0/0`、`∞ − ∞`、`√−1` の結果。NaN を含む演算はすべて NaN になり、比較 `NaN == NaN` は偽になる。学習中に損失が NaN になった場合、どこかで ∞ − ∞ か 0/0 が起きたという信号として読める
- **符号付きゼロ ±0**: `1/(−0) = −∞`。分岐切断を持つ複素関数の正しい扱いに使う

IEEE 754 は5種類の例外(invalid operation、division by zero、overflow、underflow、inexact)をフラグとして記録することを定め、既定では処理を止めずに上記の特殊値を返す[^ieee754]。

### 4. 桁落ちと条件数 (cancellation and conditioning)

浮動小数点の誤差が最も劇的に現れるのは、ほぼ等しい2数の減算である。減算そのものは正確でも(Sterbenz の補題により、y/2 ≤ x ≤ 2y なら x − y は丸めなしで表現できる)、**入力にすでに含まれていた誤差**が相対的に拡大される。これを**桁落ち (catastrophic cancellation)** と呼ぶ (Higham 2002, §1.7)[^higham]。

```text
x̂ = x(1+δ₁), ŷ = y(1+δ₂) とすると
|(x̂−ŷ) − (x−y)| / |x−y|  ≤  u · (|x| + |y|) / |x − y|
                                  └──── x ≈ y で巨大化 ────┘
```

典型例は2次方程式の解の公式である。`b² ≫ 4ac` のとき `−b + √(b²−4ac)` は打ち消し合う。分子を有理化した `2c / (−b − √(b²−4ac))` で小さい方の解を計算すれば、桁落ちは起きない。同じ事情は `1 − cos x`(→ `2 sin²(x/2)`)、分散の1パス公式 `E[x²] − E[x]²`、ソフトマックスの `exp` のオーバーフロー(→ 最大値を引いてから指数を取る)にも現れる。

ここで、誤差の責任を**問題**と**算法**に分ける語彙を導入する。関数 f の(相対)**条件数 (condition number)** は、入力の相対摂動が出力の相対変化に何倍で効くかを表す。

```text
κ_f(x) = | x f′(x) / f(x) |       (1変数・微分可能な場合)
減算 f(x, y) = x − y の条件数:  (|x| + |y|) / |x − y|
```

桁落ちは「減算という算法が悪い」のではない。**x ≈ y での減算は問題そのものが悪条件**であり、上の式はちょうどその条件数になっている。対処は算法の工夫で悪条件の減算を避けることであって、精度を上げることではない。

### 5. 前進誤差・後退誤差と総和の誤差解析 (forward/backward error; analysis of summation)

計算値 ŷ = f̂(x) の誤差は2通りに測れる (Higham 2002, §1.5)[^higham]。

```text
前進誤差 (forward error):   |ŷ − f(x)| / |f(x)|            … 答はどれだけずれたか
後退誤差 (backward error):  min{ |Δx|/|x| : ŷ = f(x + Δx) }  … どれだけ近い問題を厳密に解いたか

前進誤差  ≲  κ_f(x) × 後退誤差        (経験則の基本不等式)
```

後退誤差が u の小さな定数倍で抑えられる算法を**後退安定 (backward stable)** と呼ぶ。後退安定な算法は、入力をデータの丸めと同程度に揺らした問題を厳密に解いていることになる。そのため、それ以上の精度を算法に求めることには意味がない。残る誤差は κ が決める問題自体の性質である。この分離は [`matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) §6 の `Ax = b` の解析と同じ構造をしている。

**再帰総和 (recursive summation)**。n 個の数を左から順に足す `s ← s + xᵢ` を標準モデルで展開すると、x₁ は n−1 回の丸めを受ける。Higham は `γ_n = nu/(1 − nu)`(nu < 1)という記法で積 `∏(1+δᵢ)` を束ね、次の限界を導く (Higham 2002, §3.1, §4.2)[^higham][^higham-sum]。

```text
ŝ = Σ xᵢ (1 + θᵢ),   |θᵢ| ≤ γ_{n−1}
|ŝ − s| ≤ γ_{n−1} Σ|xᵢ|  ≈ (n−1) u Σ|xᵢ|        … 後退誤差は各 xᵢ の相対 γ_{n−1}
相対前進誤差 ≤ γ_{n−1} · κ_sum,   κ_sum = Σ|xᵢ| / |Σxᵢ|
```

限界が n に比例するのは最悪の場合であり、丸め誤差が独立なランダム変数のように振る舞うなら √n u 程度に収まる、という経験則もある (Higham 2002, §2.8)[^higham]。それでも、FP16 (u ≈ 4.9×10⁻⁴) で 10⁴ 項を素朴に足せば、最悪の場合の限界は 1 を超える。

**ペアワイズ総和 (pairwise summation)**。隣同士を足して半分の長さにする操作を繰り返す(木の形の縮約)と、各項が受ける丸めは木の深さ ⌈log₂ n⌉ 回になる。

```text
|ŝ − s| ≤ γ_{⌈log₂ n⌉} Σ|xᵢ|
```

演算回数は再帰総和と同じ n−1 回で、限界は n から log₂ n へ劇的に改善する[^higham-sum]。これは並列縮約(GPU のリダクション、all-reduce の木)が自然に取る形でもあり、並列化はたまたま精度も改善する。

**Kahan の補償総和 (compensated summation)**。Kahan (1965) は、各加算で失われた下位ビットを補正項 c に回収して次の加算へ戻す方法を示した[^kahan]。

```text
s = 0; c = 0
for each x:
    y = x − c          # 前回失った分を補正
    t = s + y          # 大きな s に小さな y を足す(y の下位ビットが落ちる)
    c = (t − s) − y    # 落ちた分を厳密に回収 (代数的には 0)
    s = t
限界: |ŝ − s| ≤ (2u + O(nu²)) Σ|xᵢ|      (Higham 2002, §4.3)
```

誤差限界は n にほぼ依存しなくなり、1項あたり加算4回の負担で「1つ上の精度で足した」のに近い結果が得られる。ただし `c = (t − s) − y` は代数的には 0 なので、コンパイラが結合則による並べ替え(`-ffast-math` 等)を許すと補正が消えてしまう。補償総和は「浮動小数点演算は実数演算ではない」ことを前提にした算法であり、最適化オプションの選択まで含めて設計する必要がある。

### 6. 融合積和演算 (fused multiply-add, FMA)

FMA は `a×b + c` を**1回の丸め**で計算する演算で、IEEE 754-2008 で標準化された[^ieee754]。

```text
fma(a, b, c) = fl(a×b + c)          … 丸め1回
fl(fl(a×b) + c)                      … 丸め2回 (分離した乗算と加算)
```

内積や行列積の内側ループは積和の連鎖なので、FMA は1命令で2演算を行い(スループットが倍になる)、同時に丸め回数を半分にする。SIMD (x86 の FMA3/AVX-512 の `VFMADD` 系命令) や GPU の演算器の基本単位が FMA であるのはこのためである。また `fma(a, b, −fl(a×b))` は積の丸め誤差を厳密に取り出せる。これは倍々精度演算や誤差のない変換 (error-free transformation) の基本部品になる。

一方で、FMA は**同じ式でも結果が変わる**原因にもなる。コンパイラが `a×b + c` を FMA に縮約するかどうか(GCC/Clang の `-ffp-contract`)は、プラットフォームや最適化レベルで異なる。また `x×x − y×y` を FMA で評価すると、x = y でも厳密に 0 にならないことがある(片方の積だけが丸められるため)。対称性を前提にしたコードは、FMA の有無を前提条件として明示する必要がある。

### 7. 非結合性と再現性 (non-associativity and reproducibility)

浮動小数点の加算は可換だが**結合的でない**。

```text
binary64:  (2⁵³ + 1) − 2⁵³ = 0      ← 2⁵³ + 1 は 2⁵³ に丸められる
           (2⁵³ − 2⁵³) + 1 = 1
```

したがって総和の結果は**足す順序の関数**になる。逐次プログラムでは順序が固定されているが、並列計算では順序がスケジューリングに依存する。

- **GPU のアトミック加算**: 複数スレッドが `atomicAdd` で同じアドレスへ足し込む順序は実行ごとに変わる
- **GEMM の split-K と自動チューニング**: 内積の分割数やカーネル選択がハードウェア・問題サイズ・ライブラリ版で変わり、縮約木の形が変わる
- **集団通信**: all-reduce のアルゴリズム(リング/木)やスイッチ内縮約の選択で、加算順序が変わる

PyTorch は「リリース、個々のコミット、プラットフォームをまたいだ完全な再現性は保証されない」と明記している。そのうえで `torch.use_deterministic_algorithms()` で決定的なアルゴリズムを強制し、非決定的な演算はエラーにできる手段を用意している[^pytorch-repro]。決定性の代償は性能である。

決定性(同じ環境で同じ結果)と再現性(異なる並列度・環境でも同じ結果)は別物である。後者を達成するには、総和の結果を順序に依存させない算法が要る。Ahrens, Demmel & Nguyen (2020) は、各項を事前に決めた境界で分割して誤差なく集計することで、**任意の順序・並列度でビット一致する総和**を、通常の総和に定数倍のコストを上乗せするだけで実現した(ReproBLAS)[^reprosum]。誤差解析の観点からは、ペアワイズ総和も Kahan 総和も誤差を小さくするが、ビット一致は保証しない点に注意する。

### 8. 混合精度と損失スケーリング (mixed precision and loss scaling)

混合精度 (mixed precision) は、演算ごとに必要な精度を見積もり、許される箇所だけ低精度にする設計である。§5 の誤差解析がその根拠になる。深層学習の標準構成は Micikevicius et al. (2018) が示した3要素からなる[^amp]。

1. **FP32 マスター重み**: 重み更新 `w ← w − η·g` では |η·g| ≪ |w| になりやすい。FP16 で足すと、更新量が w の u 倍未満のとき丸めで消える(§5 の「大きな s に小さな y」の問題)。そこで FP32 の重みのコピーに累積する
2. **FP32 累積**: 行列積の入力は FP16 でも、内積の累積は FP32 で行う(Tensor Core の基本形)。§5 の総和誤差の限界は累積精度の u で決まるからである
3. **損失スケーリング (loss scaling)**: FP16 の最小非正規化数は 2⁻²⁴ ≈ 6.0×10⁻⁸ であり、小さな勾配はこれを下回って 0 に消える(アンダーフロー)。そこで損失に係数 S を掛けて逆伝播し、勾配全体を FP16 の表現範囲へ持ち上げる。重み更新の直前に 1/S で戻す。動的損失スケーリングでは、オーバーフロー(Inf/NaN)を検出したらそのステップを捨てて S を下げ、一定ステップ問題がなければ S を上げる

損失スケーリングは、FP16 が値域を削った形式であることへの対症療法である。指数が binary32 と同じ BF16 では原理的に不要になり、大規模学習の既定が BF16 へ移った理由の1つになった[^kalamkar]。FP8 では値域がさらに狭く(E4M3 で 2⁻⁹ 〜 448)、1つの損失スケールでは全テンソルを覆えない。そのためテンソルごと(さらにブロックごと)のスケーリング係数を持ち、値域外の値は最大値へ飽和させる設計が取られる[^fp8]。数値形式を削るほど、誤差解析の仕事は「ハードウェアの外」のスケール管理へ移っていく。

## 工学への接続 (Engineering Connections)

- **ハードウェアは標準モデルの実装である**: SIMD レーンや Tensor Core の仕様書が「FMA で丸め1回」「累積は FP32」と書くとき、それは §2 の標準モデルと §5 の総和の誤差限界の、どの u が効くかを指定している。低精度演算器の性能値は、精度が半分になればスループットが倍になる関係で語られることが多い。その性能値を読む際には「累積精度はどれか」を必ず確認する。
- **再現性はコストを払って買う性質である**: 金融・規制対応・デバッグでビット一致が必要な場合、決定的カーネル、縮約順序の固定、再現可能総和 (§7) のいずれかを選び、性能低下を受け入れる。「同じ入力なら同じ出力」は浮動小数点の並列計算では既定の性質ではない。
- **式の選択は数値設計である**: 2次方程式の解の公式、分散、ソフトマックス、log-sum-exp のように、代数的に等価な式のどれを使うかで結果の有効桁数が決まる (§4)。ライブラリが「安全な」実装を提供するのは、条件数の悪い部分式を避けるためである。
- **低精度化は値域と精度の再配分である**: FP16 → BF16 → FP8 → MX/FP4 の流れは、「指数と仮数のビットをどう配るか」と「足りない値域をスケール係数でどう補うか」という §1 と §8 の設計問題の繰り返しである。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/cpu/simd-vector.md`](../../../hardware/cpu/simd-vector.md) | SIMD レーンのデータ型としての binary32/64、FMA 命令 (§6)、ベクトル化による縮約順序の変化 (§7) |
| [`hardware/accelerators/gpu-simt-architecture.md`](../../../hardware/accelerators/gpu-simt-architecture.md) | Tensor Core の混合精度(FP16/BF16/FP8 入力・FP32 累積, §1, §8)、アトミック加算と非決定性 (§7) |
| [`hardware/accelerators/tpu-systolic-arrays.md`](../../../hardware/accelerators/tpu-systolic-arrays.md) | bfloat16 を前提とした行列演算器 (§1) |
| [`software/ai/ml-training-systems.md`](../../../software/ai/ml-training-systems.md) | 混合精度・FP32 マスター・損失スケーリング (§8)、並列縮約と再現性の緊張 (§7)、FP8 のスケーリング管理 |
| [`software/ai/ml-inference-systems.md`](../../../software/ai/ml-inference-systems.md) | 低精度推論・量子化の誤差を見積もる枠組み(単位丸め・条件数, §2, §4) |
| [`../linear-algebra/matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) | 後退誤差解析の連立一次方程式版、混合精度反復改良 (§5 の一般論の特殊化) |

## 参考文献 (References)

[^higham]: Higham, N. J. (2002). *Accuracy and Stability of Numerical Algorithms* (2nd ed.). SIAM. DOI: 10.1137/1.9780898718027 — 標準モデル (§2.2)、前進/後退誤差 (§1.5)、桁落ち (§1.7)、γ_n 記法 (§3.1)、総和 (Ch. 4) の主参照。
[^higham-sum]: Higham, N. J. (1993). The Accuracy of Floating Point Summation. *SIAM Journal on Scientific Computing*, 14(4), 783–799. DOI: 10.1137/0914050 — 再帰・ペアワイズ・補償総和の誤差限界の比較。
[^goldberg]: Goldberg, D. (1991). What Every Computer Scientist Should Know About Floating-Point Arithmetic. *ACM Computing Surveys*, 23(1), 5–48. DOI: 10.1145/103162.103163 — 浮動小数点の構造、段階的アンダーフロー、特殊値の設計意図。
[^ieee754]: IEEE. (2019). *IEEE Standard for Floating-Point Arithmetic* (IEEE Std 754-2019). DOI: 10.1109/IEEESTD.2019.8766229 — 交換形式、丸め方向属性、正しい丸め、FMA、例外。
[^kahan]: Kahan, W. (1965). Pracniques: Further Remarks on Reducing Truncation Errors. *Communications of the ACM*, 8(1), 40. DOI: 10.1145/363707.363723 — 補償総和の原典。
[^reprosum]: Ahrens, W., Demmel, J., & Nguyen, H. D. (2020). Algorithms for Efficient Reproducible Floating Point Summation. *ACM Transactions on Mathematical Software*, 46(3), 1–49. DOI: 10.1145/3389360 — 順序に依存しない再現可能総和 (ReproBLAS)。
[^amp]: Micikevicius, P., Narang, S., Alben, J., et al. (2018). Mixed Precision Training. *International Conference on Learning Representations (ICLR 2018)*. arXiv:1710.03740 — FP32 マスター重み・損失スケーリング・FP32 累積の原典。
[^fp8]: Micikevicius, P., Stosic, D., Burgess, N., et al. (2022). FP8 Formats for Deep Learning. arXiv:2209.05433 — E4M3/E5M2 の定義 (Table 1: バイアス 7/15、最大値 448/57344、最小非正規化数 2⁻⁹/2⁻¹⁶)、特殊値の設計、テンソル単位スケーリング。
[^ocp-fp8]: [OCP 8-bit Floating Point Specification (OFP8), Revision 1.0](https://www.opencompute.org/documents/ocp-8-bit-floating-point-specification-ofp8-revision-1-0-2023-12-01-pdf-1) — Open Compute Project. Accessed: 2026-10-04 — E4M3/E5M2 の業界仕様。
[^kalamkar]: Kalamkar, D., Mudigere, D., Mellempudi, N., Das, D., Banerjee, K., et al. (2019). A Study of BFLOAT16 for Deep Learning Training. arXiv:1905.12322 — bfloat16 の形式と、FP32 と同じ値域によりハイパーパラメータ調整なしで学習できることの実証。
[^pytorch-repro]: [PyTorch: Reproducibility](https://docs.pytorch.org/docs/2.14/notes/randomness.html) — Accessed: 2026-10-04 — 完全な再現性は保証しない旨と `torch.use_deterministic_algorithms()`。
