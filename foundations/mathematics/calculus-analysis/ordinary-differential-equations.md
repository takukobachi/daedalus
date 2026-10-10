# 常微分方程式 (Ordinary Differential Equations)

## 概要 (Overview)

微分積分・解析シリーズの第2回。物理法則は「状態そのもの」ではなく「状態の**変化の規則**」として与えられる。ニュートンの運動方程式、回路の電圧電流関係、発電機の動揺方程式は、すべて**常微分方程式 (ordinary differential equation, ODE)**、すなわち未知関数とその導関数を結ぶ方程式である。前回の指数関数 `dy/dt = ky` はその最小例だった。

本回は工学で実際に出会う順に3段で進む: (1) **1階線形**(RC 回路と時定数の世界)、(2) **2階線形**(振動と減衰の世界。ばね・振り子・RLC 回路が**同じ方程式**を共有するという事実が、機械・電気を横断する「系のアナロジー」を成立させる)、(3) **連立線形系** `dx/dt = Ax`(安定性が行列 A の固有値で決まり、線形代数と解析がここで完全に合流する)。最後に、閉形式で解けない大多数の方程式のための数値解法(オイラー法・ルンゲ–クッタ法)と、その落とし穴(硬い方程式)に触れる。

## 前提知識 (Prerequisites)

- [`limits-derivatives-integrals.md`](limits-derivatives-integrals.md) — 微分、テイラー展開、指数関数と `dy/dt = ky`
- [`../linear-algebra/eigenvalues-diagonalization.md`](../linear-algebra/eigenvalues-diagonalization.md) — 固有値・対角化(§4 で使う)

## 本論 (Main Content)

### 1. 微分方程式という記述形式 (ODEs as a modeling language)

ODE は「各瞬間の変化率を、現在の状態の関数として指定する」記述形式である。

```text
dy/dt = f(y, t),  y(0) = y₀   (初期値問題)
階数 = 現れる最高階の導関数。線形 = y とその導関数について1次
```

f が滑らかなら解は存在して一意である(ピカール–リンデレーフの定理; Boyce & DiPrima 2017, §2.8)。これは**初期状態が未来を一意に決める**という古典物理の決定論の数学的表現であり、同時に「シミュレーションが原理的に可能」であることの保証でもある。高階方程式は変数を増やせば常に1階の連立系に直せる(`y'' = …` は `(y, y')` の2変数1階系)ため、理論も数値解法も1階連立系 `dx/dt = f(x)` を標準形として扱う。

### 2. 1階線形 — 時定数の世界 (first order: the world of time constants)

定数係数の1階線形方程式と、その代表例である RC 回路を示す。

```text
τ·dy/dt + y = u(t)        (τ: 時定数)
RC 回路: RC·dV/dt + V = V_in — 抵抗を通ってキャパシタを充電
解 (ステップ入力): V(t) = V_in(1 − e^{−t/τ}),  τ = RC
```

解は「初期状態の記憶が指数的に消え (過渡応答)、入力が決める平衡へ向かう (定常応答)」という普遍構造を持つ。線形方程式では両者の**重ね合わせ (superposition)** が成り立つ。つまり、複数入力の応答は個別応答の和になる。これが「線形システム論」という体系が成立する根拠であり、フーリエ解析(入力を正弦波の和に分解して個別に解く)への伏線でもある。1階系の応答は τ の1個の数に要約され、信号のエッジ遅延 (RC 遅延)、センサの応答速度、温度の緩和など、「遅れて追従する」系すべての第一近似になる。

### 3. 2階線形 — 振動と減衰の世界 (second order: oscillation and damping)

2階になると本質的に新しい現象である**振動**が現れる。標準形と、それを共有する物理系を示す。

```text
d²y/dt² + 2ζω₀·dy/dt + ω₀²y = 入力
ω₀: 固有角周波数, ζ: 減衰比 (damping ratio)

ばね-質量-ダンパ: m·y'' + c·y' + k·y = F     ω₀ = √(k/m)
直列 RLC:        L·q'' + R·q' + q/C = V      ω₀ = 1/√(LC)
対応: 質量 m ↔ インダクタンス L, 摩擦 c ↔ 抵抗 R, ばね k ↔ 1/C, 変位 ↔ 電荷
```

**同じ方程式は同じ振る舞いをする**。この対応表により、機械系の直観(重いほどゆっくり揺れる)が電気系(L が大きいほど共振周波数が低い)へそのまま移送でき、逆に回路シミュレータで機械系を解ける。解の性格は ζ だけで分類される (Boyce & DiPrima 2017, Ch. 3)。

```text
ζ > 1: 過減衰 — 振動せずゆっくり収束
ζ = 1: 臨界減衰 — 振動せず最速で収束 (メーターの針、ドアクローザーの設計点)
ζ < 1: 不足減衰 — 減衰振動。オーバーシュートとリンギング
ζ = 0: 無減衰 — 共振周波数 ω₀ で振動し続ける
```

周期入力の周波数が ω₀ に近づくと応答が増幅される**共振 (resonance)** が起き、ζ が小さいほど鋭く増幅する(増幅率 ≈ 1/2ζ)。共振は設計の敵にも味方にもなる。電源分配網の反共振ピークやリンギングは抑えるべき敵、水晶振動子・RLC 同調回路・原子時計は共振を精度の源として使う味方である。

### 4. 連立線形系と安定性 — 固有値との合流 (linear systems and stability)

1階連立線形系の一般解は行列指数関数で書ける。

```text
dx/dt = Ax   ⟹   x(t) = e^{At}x(0),   e^{At} = Σ (At)ⁿ/n!
A = PDP⁻¹ と対角化できれば e^{At} = P·diag(e^{λᵢt})·P⁻¹
```

解は固有値 λᵢ ごとのモード `e^{λᵢt}` の重ね合わせに分解され、**安定性が固有値の実部だけで決まる** (Strogatz 2015, Ch. 5)。

```text
すべての Re(λᵢ) < 0 ⟹ 漸近安定 (どんな初期値からも 0 へ)
ある Re(λᵢ) > 0   ⟹ 不安定 (そのモードが指数的に成長)
虚部 Im(λᵢ)       ⟹ 振動周波数
```

線形代数第2回の「固有値は繰り返し適用の漸近挙動を支配する」の連続時間版であり、制御工学の「極 (pole)」の正体である。非線形系 `dx/dt = f(x)` も、平衡点まわりでテイラー展開(前回 §3)して1次で打ち切れば `dx/dt ≈ J·x`(J はヤコビアン)となり、この判定がそのまま使える。**線形化 → 固有値 → 安定判別**という3段は、制御設計・電力系統の安定度解析・回路の発振条件判定に共通する標準手順である。

### 5. 数値解法 (numerical methods)

閉形式で解ける ODE は例外であり、実務の解は数値積分で得る。1ステップ h だけ進める基本形を示す。

```text
オイラー法:    xₙ₊₁ = xₙ + h·f(xₙ)          誤差 O(h) — 概念実証用
古典 RK4:     勾配を4点で評価して加重平均      誤差 O(h⁴) — 汎用の標準
```

精度はテイラー展開との一致次数で決まり、誤差評価は前回 §3 の剰余項の直接の応用である。実用上の最大の罠は**硬い方程式 (stiff equations)** である。時定数が桁違いに混在する系(高速な寄生成分と低速な主応答を併せ持つ回路が典型)では、陽解法は「最速の時定数」に h を縛られ、興味のある遅い応答を解くのに天文学的なステップ数を要する。解決は**陰解法 (implicit methods)** である。各ステップで連立方程式(非線形なら Newton 法 → 内側は線形代数第4回の LU 分解)を解く代わりに、大きな h でも数値的に安定になる。SPICE 系回路シミュレータが陰解法(後退オイラー・台形法)を標準とするのはこのためである (Hairer & Wanner 1996)。安定性解析や硬い方程式への対処の詳細は [`../numerical-analysis/equations-ode-numerical-methods.md`](../numerical-analysis/equations-ode-numerical-methods.md) で扱う。

## 工学への接続 (Engineering Connections)

- **回路シミュレーション**: SPICE の過渡解析は「回路 → 連立 ODE → 陰解法で時間積分 → 各ステップの Newton 法 → 疎 LU」というパイプラインであり、本回 §5 と線形代数第4回がその全部品を与える。
- **制御系設計の語彙**: 極 = 固有値 (§4)、減衰比・オーバーシュート・整定時間 (§3) は、PID 調整からフライトコントロールまで、フィードバック設計の共通言語。制御理論シリーズ(foundations 将来候補)はここから伸びる。
- **電力系統の動揺方程式**: 同期発電機の回転子は `M·δ'' + D·δ' = P_m − P_e` で記述される2階系 (§3) そのものであり、系統擾乱後の同期維持(過渡安定度)は §4 の安定判別の産業規模の応用である。
- **軌道伝播と誤差の管理**: 軌道力学の数値積分では、積分法の次数・ステップ制御 (§5) が予測精度と計算量を直接決める。長時間積分ではエネルギー保存を尊重する解法(シンプレクティック法)が選ばれる、という発展もこの延長にある。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/hw-sw-interface/clock-power-reset.md`](../../../hardware/hw-sw-interface/clock-power-reset.md) | RC 遅延と時定数 (§2)、電源シーケンスの過渡応答、PLL のループ動特性 (§3–4) |
| [`hardware/hw-sw-interface/bus-interconnect.md`](../../../hardware/hw-sw-interface/bus-interconnect.md) | リンギング=不足減衰の2階応答 (§3)。終端設計は ζ の設計 |
| [`engineering/energy/generation-grid-fundamentals.md`](../../../engineering/energy/generation-grid-fundamentals.md) | 動揺方程式と過渡安定度 (§3–4)、周波数応答・慣性の意味 |
| [`engineering/aerospace/orbital-mechanics.md`](../../../engineering/aerospace/orbital-mechanics.md) | 運動方程式の数値積分としての軌道伝播 (§5) |
| [`engineering/aerospace/avionics-gnc.md`](../../../engineering/aerospace/avionics-gnc.md) | 姿勢・軌道制御 = 線形化 → 固有値配置 (§4) の応用領域 |
| [`engineering/automotive/ev/tesla-battery-powertrain.md`](../../../engineering/automotive/ev/tesla-battery-powertrain.md) | モータ制御・電流ループの動特性、バッテリの等価回路モデル (RC 網, §2) |
| [`../../physics/electromagnetism/induction-inductance.md`](../../physics/electromagnetism/induction-inductance.md) | L·di/dt がもたらす1階・2階の過渡現象の数学的処理 |

## 参考文献 (References)

- Boyce, W. E., & DiPrima, R. C. (2017). *Elementary Differential Equations and Boundary Value Problems* (11th ed.). Wiley. — 1階・2階線形の解法と分類の主参照
- Strogatz, S. H. (2015). *Nonlinear Dynamics and Chaos* (2nd ed.). Westview Press. — 連立系・安定性・線形化の直観の主参照
- Hairer, E., & Wanner, G. (1996). *Solving Ordinary Differential Equations II: Stiff and Differential-Algebraic Problems* (2nd ed.). Springer. DOI: 10.1007/978-3-642-05221-7 — 硬い方程式と陰解法の主参照
- Hairer, E., Nørsett, S. P., & Wanner, G. (1993). *Solving Ordinary Differential Equations I: Nonstiff Problems* (2nd ed.). Springer. DOI: 10.1007/978-3-540-78862-1 — ルンゲ–クッタ法と誤差解析
