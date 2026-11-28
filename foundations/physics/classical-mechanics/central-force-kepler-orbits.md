# 中心力とケプラー軌道 (Central Forces and Kepler Orbits)

## 概要 (Overview)

古典力学シリーズの第3回。軌道力学の分析 ([`engineering/aerospace/orbital-mechanics.md`](../../../engineering/aerospace/orbital-mechanics.md)) は、ヴィス・ヴィヴァ式 v² = μ(2/r − 1/a)、ケプラーの3法則、6つの軌道要素を道具として使い、Δv の収支を組み立てた。本回はそれらの式がどこから来るかを、運動方程式 r̈ = −μr/r³ から導く。

道筋は次のとおりである。(1) 二体問題を、換算質量を使って1体の中心力問題に帰着させる。(2) 中心力は回転対称なので角運動量が保存し、運動は平面内に限られ、面積速度は一定になる(ケプラーの第2法則)。(3) エネルギー保存と**有効ポテンシャル (effective potential)** を使うと、半径方向の運動を1次元問題として読める。束縛軌道と脱出の境界もここで見える。(4) 軌道方程式を解くと解は円錐曲線になり(第1法則)、周期は長半径だけで決まる(第3法則)。ヴィス・ヴィヴァ式と脱出速度もこの過程で出てくる。(5) 3次元の向きを表す6つの軌道要素と、時刻から位置を求めるケプラー方程式を導入する。(6) 現実の軌道を理想の楕円からずらす摂動のうち、地球の扁平 (J2) による軌道面の歳差を扱い、太陽同期軌道がなぜ成立するかを示す。

## 前提知識 (Prerequisites)

- [`newton-laws-oscillations.md`](newton-laws-oscillations.md) — 運動方程式、運動量・角運動量・エネルギーの保存則
- [`lagrangian-multibody-dynamics.md`](lagrangian-multibody-dynamics.md) — 循環座標と保存則 (§5)。本回の角運動量保存はその代表例
- [`../../mathematics/calculus-analysis/ordinary-differential-equations.md`](../../mathematics/calculus-analysis/ordinary-differential-equations.md) — 2階非線形 ODE と保存量による次元削減
- [`../../mathematics/linear-algebra/vectors-matrices-linear-maps.md`](../../mathematics/linear-algebra/vectors-matrices-linear-maps.md) — 外積、座標回転(軌道要素の座標変換で使う)

## 本論 (Main Content)

### 1. 二体問題の1体問題への帰着 (reduction of the two-body problem)

質量 m₁, m₂ の2物体が万有引力 F = G m₁ m₂ / r² で引き合う系では、外力がないので重心は等速直線運動をする。相対位置 r = r₂ − r₁ の運動方程式を書くと、次のように1体問題になる (Goldstein et al. 2002, Ch. 3)。

```text
μ_red r̈ = −G m₁ m₂ r / r³,   μ_red = m₁ m₂ / (m₁ + m₂)  (換算質量)
⟺ r̈ = −μ r / r³,   μ = G (m₁ + m₂)
人工衛星では m₂ ≪ m₁ なので μ ≈ G M_⊕
  地球: μ = 398 600.4418 km³/s² (IERS Conventions の数値標準)
```

G と M_⊕ を別々に測るより、積 μ = GM を衛星の軌道から直接決めるほうがはるかに精度が高い。このため軌道計算は G ではなく μ で書く。月と地球のように質量比が大きくない場合でも、換算質量を使えば同じ式がそのまま成り立ち、地球自身も共通重心のまわりを回る。

### 2. 角運動量保存と面積速度 (angular momentum and Kepler's second law)

力が常に中心を向く(**中心力 (central force)**)とき、トルク r × F はゼロなので、単位質量あたりの角運動量 h = r × v は一定である。h が一定のベクトルなので、r と v は h に垂直な一つの平面内に留まる。軌道が平面になるのはこのためで、軌道面の向きを変えるには推進剤が必要になる(面変更が高価な理由)。

極座標 (r, θ) では h = r² θ̇ と書ける。動径が微小時間 dt に掃く面積は dA = ½ r² dθ なので dA/dt = h/2 で一定になる。これがケプラーの第2法則で、近地点で速く遠地点で遅く動くことを表す。第2回の言葉で言えば、ラグランジアンに θ が現れない(循環座標)ことから p_θ = r² θ̇ が保存する。回転対称性が角運動量保存を生む、ネーターの定理の最も古い例である (Goldstein et al. 2002, Ch. 3)。

### 3. エネルギーと有効ポテンシャル (energy and the effective potential)

単位質量あたりの力学的エネルギーは保存する。θ̇ = h/r² を使って角度方向の運動エネルギーを r の関数に書き換えると、半径方向の運動は1次元のポテンシャル問題になる (Curtis 2020, Ch. 2)。

```text
ε = ½ ṙ² + ½ r² θ̇² − μ/r = ½ ṙ² + U_eff(r)
U_eff(r) = h²/(2r²) − μ/r      (遠心力の障壁 + 重力の井戸)
  ε < 0:  r は r_p ≤ r ≤ r_a の間を往復 — 束縛軌道 (楕円)
  ε = 0:  無限遠でちょうど速度ゼロ — 放物線 (脱出の境界)
  ε > 0:  無限遠でも速度が残る — 双曲線 (フライバイ・惑星間)
  U_eff の最小点 r = h²/μ では ṙ = 0 が続く — 円軌道
```

h²/(2r²) の項は角運動量をもつ物体が中心に近づけないことを表す「遠心力の障壁」である。角運動量をもつ衛星が地球に落ちない理由は、この障壁が重力の井戸の底を持ち上げていることにある。第1回 §5 の言葉では、円軌道は U_eff の谷底にある安定平衡点であり、少しずらすと半径方向に振動する。この振動がわずかに離心率をもつ楕円軌道である。

ε = 0 と置くと**脱出速度 (escape velocity)** v_esc = √(2μ/r) が出る。地球の赤道半径 6378.137 km で計算すると 11.18 km/s(本稿の試算)になり、同じ高度の円軌道速度 √(μ/r) のちょうど √2 倍である。高度 400 km では円軌道速度 7.67 km/s、脱出速度 10.84 km/s になる(本稿の試算)。

### 4. 軌道方程式と円錐曲線 (the orbit equation and conic sections)

半径方向の運動方程式で時間を消去し、u = 1/r と置くと、θ についての線形 ODE(**ビネの式 (Binet equation)**)が得られる。

```text
d²u/dθ² + u = μ/h²     ⟹   r(θ) = p / (1 + e cos ν)
  p = h²/μ: 半通径 (semi-latus rectum),  e: 離心率,  ν: 真近点角 (近点から測った角)
  楕円 (0 ≤ e < 1): 長半径 a = p/(1 − e²),  r_p = a(1 − e),  r_a = a(1 + e)
  エネルギーと長半径:  ε = −μ/(2a)
```

非線形な運動方程式が、u に置き換えると単振動と同じ形の線形方程式になる。解は焦点を中心天体に置く円錐曲線で、これがケプラーの第1法則である (Bate et al. 1971, Ch. 1)。逆2乗則の場合に限り、近点の向きが動かない閉じた軌道になる(ベルトランの定理)。したがって §6 の摂動で力が逆2乗則から少しずれると、近点の向きはゆっくり回り始める。

ε = −μ/(2a) を ε = v²/2 − μ/r に代入すれば、ヴィス・ヴィヴァ式 v² = μ(2/r − 1/a) が出る。エネルギーが長半径だけで決まり、離心率によらないことがこの式の要点である。ホーマン遷移の Δv 計算は出発軌道・遷移楕円・到着軌道の3つの a をこの式に入れるだけで済む。

第3法則は面積速度から導かれる。楕円の面積 πab を面積速度 h/2 で割ると周期になり、b = a√(1 − e²) と h = √(μp) を使うと次式になる。

```text
T = 2π √(a³/μ)          (ケプラーの第3法則: 周期は長半径だけで決まる)
  例 (本稿の試算): 高度 400 km の円軌道 → T ≈ 92.6 分
       静止軌道: T = 1 恒星日 (86 164 s) → a ≈ 42 164 km (高度 ≈ 35 786 km), v ≈ 3.07 km/s
```

### 5. 軌道要素とケプラー方程式 (orbital elements and Kepler's equation)

3次元の軌道は、2つの保存ベクトル(角運動量 h と、近点を向く離心率ベクトル e = (v × h)/μ − r/r)と時刻の基準で決まる。これを6つのスカラーに直したものが**古典軌道要素 (classical orbital elements)** である。a と e が大きさと形、傾斜角 i・昇交点赤経 Ω・近点引数 ω が向き、元期の平均近点角 M₀ が時刻の基準を与える。i, Ω, ω の3つは軌道面座標から赤道座標への3回の回転角(オイラー角)にほかならない (Vallado 2013, Ch. 2)。

時刻 t から軌道上の位置を求めるには、面積速度一定を角度で表した**ケプラー方程式 (Kepler's equation)** を解く。

```text
平均運動 n = √(μ/a³),  平均近点角 M = n (t − t_p)
ケプラー方程式:  M = E − e sin E   (E: 離心近点角)
  E が求まれば  tan(ν/2) = √((1+e)/(1−e)) tan(E/2),   r = a(1 − e cos E)
```

この式は E について超越方程式で、閉形式では解けない。実務ではニュートン法で数回反復して解く。軌道伝播ソフトウェアや GNSS 受信機の衛星位置計算では、毎回この反復が実行されている。

### 6. 摂動: 地球の扁平と太陽同期軌道 (perturbations: J2 and sun-synchronous orbits)

現実の地球は赤道方向に膨らんだ回転楕円体に近く、重力場は逆2乗則から少しずれる。その最大の項を**J2 項**(J2 ≈ 1.0826 × 10⁻³)と呼ぶ。赤道の膨らみが軌道面にトルクをかけるため、角運動量ベクトルはジャイロのように歳差運動し、昇交点赤経 Ω がゆっくり回る (Vallado 2013, Ch. 9)。

```text
dΩ/dt = −(3/2) n J2 (R_⊕/p)² cos i
  i < 90° (順行) で西向きに、i > 90° (逆行) で東向きに回る
太陽同期軌道: dΩ/dt = +360°/365.2422 日 ≈ +0.9856°/日 を満たすように i を選ぶ
  高度 600–800 km では i ≈ 98° 前後
```

太陽同期軌道では、軌道面と太陽の向きの関係が1年を通じてほぼ一定に保たれる。地球観測衛星が毎回同じ地方時に同じ場所の上空を通過し、影の向きや照明条件をそろえて画像を比較できるのはこのためである。摂動は誤差であると同時に、設計者が利用できる「無料の Δv」でもある。ほかに大気抵抗(低軌道の高度を徐々に下げ、軌道寿命を決める)、月と太陽の重力(静止軌道の傾斜角を年に約 0.75–0.95° ずつ増やす)、太陽輻射圧がある。

## 工学への接続 (Engineering Connections)

- **Δv 収支はエネルギーと角運動量の会計である**: 軌道遷移は ε(つまり a)と h(つまり p と面の向き)を推力で書き換える操作である (§3–4)。近点での加速が効率的なのは、速度が大きい場所ほど同じ Δv で大きなエネルギー変化 v·Δv が得られるからで、これがオーベルト効果の正体である。
- **面変更が高価なのは角運動量ベクトルの向きを変えるから**: 軌道面の向きは h の向きそのものであり (§2)、向きを変えるには速度ベクトル全体を回す必要がある。打ち上げ時に射場の緯度と方位で傾斜角を合わせるのが原則になる。
- **軌道伝播と航法の計算核**: ケプラー方程式の数値解法 (§5) と J2 摂動 (§6) は、衛星の軌道予測、GNSS のエフェメリス計算、衝突回避の接近解析の基本部品である。
- **軌道設計は摂動の利用である**: 太陽同期軌道は J2 を、静止軌道の南北保持は月・太陽摂動への対抗を設計に組み込んでいる (§6)。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`engineering/aerospace/orbital-mechanics.md`](../../../engineering/aerospace/orbital-mechanics.md) | ヴィス・ヴィヴァ式・ケプラーの3法則・脱出速度の導出 (§3–4)、軌道要素 (§5)、面変更の高さ (§2)、SSO (§6) |
| [`engineering/aerospace/satellites/`](../../../engineering/aerospace/satellites/) | 太陽同期軌道と J2 歳差 (§6)、衛星の軌道寿命と大気抵抗 |
| [`engineering/aerospace/hls-lunar-architecture.md`](../../../engineering/aerospace/hls-lunar-architecture.md) | 二体近似の限界と月往還軌道 (§1, §6) |
| [`engineering/aerospace/avionics-gnc.md`](../../../engineering/aerospace/avionics-gnc.md) | 誘導・航法で使う軌道伝播 (§5) |
| [`engineering/aerospace/spacex-rocket-engines.md`](../../../engineering/aerospace/spacex-rocket-engines.md) | Δv を推進剤に翻訳する側。必要 Δv の出どころが本回 |
| [`foundations/physics/classical-mechanics/lagrangian-multibody-dynamics.md`](lagrangian-multibody-dynamics.md) | 循環座標と角運動量保存 (§2 の一般論) |

## 参考文献 (References)

- Goldstein, H., Poole, C., & Safko, J. (2002). *Classical Mechanics* (3rd ed.). Addison-Wesley. — 中心力問題、換算質量、ケプラー問題、ベルトランの定理 (Ch. 3)。§1–4 の理論的主参照
- Curtis, H. D. (2020). *Orbital Mechanics for Engineering Students* (4th ed.). Butterworth-Heinemann. — 二体問題・軌道方程式・ケプラー方程式の工学的導出。§3–5 の参照
- Bate, R. R., Mueller, D. D., & White, J. E. (1971). *Fundamentals of Astrodynamics*. Dover. — 二体問題と円錐曲線解、軌道要素の古典的扱い
- Vallado, D. A. (2013). *Fundamentals of Astrodynamics and Applications* (4th ed.). Microcosm Press. — 軌道要素とケプラー方程式の数値解法 (Ch. 2)、J2 摂動と太陽同期軌道 (Ch. 9)。§5–6 の主参照
- Petit, G., & Luzum, B. (Eds.) (2010). *IERS Conventions (2010)*. IERS Technical Note No. 36. Verlag des Bundesamts für Kartographie und Geodäsie. — 地心重力定数 GM = 3.986004418 × 10¹⁴ m³/s² の数値標準
