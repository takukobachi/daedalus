# 非圧縮性流れと境界層 (Incompressible Flow and Boundary Layers)

## 概要 (Overview)

流体力学シリーズの第3回。第1回と第2回は、音速に近い・音速を超える流れで密度が大きく変わる圧縮性流れを扱った。本回は逆に、マッハ数がおよそ 0.3 より小さく密度をほぼ一定とみなせる**非圧縮性流れ (incompressible flow)** を扱う。旅客機の離着陸、ドローン、自動車、配管の冷却水、データセンターの空調はすべてこの領域にある。航空機の分析 ([`engineering/aerospace/aircraft-engineering-fundamentals.md`](../../../engineering/aerospace/aircraft-engineering-fundamentals.md)) は L = ½ρV²SC_L、クッタ・ジュコフスキーの定理、境界層の剥離による失速を使った。本回はそれらの理論的な出どころを示す。

中心にある問いは、「粘性は小さいのに、なぜ抵抗も揚力も粘性なしでは説明できないのか」である。(1) 連続の式とオイラー方程式から、流線に沿った**ベルヌーイの定理**を導き、その適用条件を明確にする。(2) 粘性を加えたナビエ=ストークス方程式と**レイノルズ数**を導入し、模型試験が成り立つ理由(相似則)を示す。(3) 粘性を完全に無視した理論では抗力がゼロになってしまう**ダランベールのパラドックス**を、Prandtl の**境界層 (boundary layer)** の考え方で解消する。(4) 層流から乱流への遷移と、逆圧力勾配による**剥離 (separation)** を扱う。(5) 最後に、揚力が循環から生じること、その循環の大きさを粘性(クッタ条件)が決めること、抗力が摩擦と圧力の2系統に分かれることを、境界層の言葉でまとめる。

## 前提知識 (Prerequisites)

- [`compressible-flow-nozzles.md`](compressible-flow-nozzles.md) — 連続の式・運動量・エネルギー保存、マッハ数。本回はその M → 0 の極限から始める
- [`../classical-mechanics/newton-laws-oscillations.md`](../classical-mechanics/newton-laws-oscillations.md) — 運動量保存と第3法則(揚力の反作用としての吹き下ろし)
- [`../../mathematics/calculus-analysis/multivariable-calculus.md`](../../mathematics/calculus-analysis/multivariable-calculus.md) — 発散・回転・線積分(循環の定義で使う)
- [`../../mathematics/calculus-analysis/ordinary-differential-equations.md`](../../mathematics/calculus-analysis/ordinary-differential-equations.md) — 相似解が ODE に落ちる手続き (Blasius 解)

## 本論 (Main Content)

### 1. 非圧縮の条件とベルヌーイの定理 (incompressibility and Bernoulli's equation)

密度の相対変化はおよそ Δρ/ρ ≈ M²/2 の大きさをもつ。M = 0.3 で約 5% なので、それ以下では密度を一定と置いても工学上の誤差は小さい (Anderson 2017, Ch. 3)。このとき質量保存は ∇·v = 0(体積が保存する)となる。粘性を無視した運動方程式(**オイラー方程式 (Euler equation)**)を、定常流で1本の流線に沿って積分すると次式が得られる。

```text
p + ½ ρ V² + ρ g z = 一定   (流線に沿って)
  p: 静圧,  ½ρV²: 動圧 q,  p + ½ρV²: 全圧 (よどみ点圧)
適用条件: 定常・非粘性・非圧縮・同一流線上 (渦なし流れなら流れ場全体で一定)
```

ベルヌーイの定理は流体の微小部分に対する運動エネルギーの定理である。圧力の低い方へ押されて加速し、高い方へ向かうと減速する。ピトー管が全圧と静圧の差から速度 V = √(2(p₀ − p)/ρ) を測れるのはこの式による。揚力・抗力が q = ½ρV² に比例する形で書かれるのも、空気力の大きさが動圧で測られるからである。注意すべきは、ベルヌーイの定理が圧力と速度を**関係づける**だけで、なぜ翼の上面で流れが速くなるのかは説明しないことである。それを決めるのは §5 の循環である。

### 2. 粘性、ナビエ=ストークス方程式、レイノルズ数 (viscosity, Navier–Stokes, and the Reynolds number)

**粘性 (viscosity)** は隣り合う流体層の間で運動量をやりとりする性質である。空気や水のような**ニュートン流体 (Newtonian fluid)** では、せん断応力が速度勾配に比例する。これを運動方程式に加えたものが**ナビエ=ストークス方程式 (Navier–Stokes equations)** である (White 2011, Ch. 4)。

```text
せん断応力: τ = μ du/dy     (μ: 粘性係数 [Pa·s]、ν = μ/ρ: 動粘性係数 [m²/s])
  空気 (海面 15 °C): ν ≈ 1.46 × 10⁻⁵ m²/s,   水 (20 °C): ν ≈ 1.0 × 10⁻⁶ m²/s
非圧縮 NS:  ρ (∂v/∂t + (v·∇)v) = −∇p + μ ∇²v
無次元化すると係数は1つだけ残る:
  Re = ρ V L / μ = V L / ν    (慣性力 / 粘性力)
```

**レイノルズ数 (Reynolds number)** は、慣性による運動量の輸送と粘性による拡散の比である。形が相似で Re(圧縮性があればマッハ数も)が等しい2つの流れは、無次元化すれば同じ解をもつ。小さな模型を風洞で試験して実機の C_L・C_D を予測できるのは、この**力学的相似 (dynamic similarity)** による。目安として、旅客機の主翼は巡航で Re ≈ 10⁷ 程度、小型ドローンのプロペラは 10⁴–10⁵ 程度であり(本稿の概算)、同じ「翼」でも境界層の振る舞いが大きく異なる。

### 3. ダランベールのパラドックスと境界層 (d'Alembert's paradox and the boundary layer)

Re が大きいなら粘性項を落としてよさそうに見える。ところが、非粘性・渦なしの理論で物体まわりの定常流を解くと、物体にはたらく抗力は厳密にゼロになる。これが**ダランベールのパラドックス (d'Alembert's paradox)** で、現実の物体が必ず抵抗を受けることと矛盾する (Anderson 2017, Ch. 3)。

1904 年に Prandtl はこれを解消した。粘性は流れ全体では小さくても、壁の上では流体が壁に貼りつく(**滑りなし条件 (no-slip condition)**)ため、壁のごく近くに速度が 0 から外部流の速度 U まで急変する薄い層ができる。この**境界層**の中では速度勾配が大きいので、粘性は決して無視できない。その外側では非粘性理論がよく成り立つ。流れ場を「薄い粘性層 + 外側の非粘性流」に分けるこの考え方が、近代空気力学の出発点になった (Schlichting & Gersten 2017, Ch. 1)。

```text
平板の層流境界層 (Blasius の相似解):
  境界層厚さ  δ₉₉ ≈ 5.0 x / √Re_x            (Re_x = U x / ν)
  局所摩擦係数 c_f = τ_w / (½ρU²) = 0.664 / √Re_x
  例 (本稿の試算): 空気 U = 50 m/s, x = 1 m → Re_x ≈ 3.4 × 10⁶, δ₉₉ ≈ 2.7 mm (層流と仮定した場合)
乱流境界層 (1/7 乗則の経験式):  δ ≈ 0.37 x / Re_x^{1/5},  c_f ≈ 0.059 / Re_x^{1/5}
```

境界層は 1 m 進んでも数 mm にしかならない。それでも、壁面摩擦による抗力はこの層の中の速度勾配で決まり、次節の剥離もこの層で起きる。

### 4. 遷移と剥離 (transition and separation)

境界層は下流へ進むにつれて厚くなり、局所レイノルズ数がある値を超えると、小さな擾乱が増幅されて**乱流 (turbulent flow)** に遷移する。平板では Re_x ≈ 5 × 10⁵ が教科書的な目安だが、外部流の乱れや表面粗さによって 10⁵ から 10⁶ 台まで変わる。管内流れでは Re_D ≈ 2300 が層流の上限の目安である (White 2011, Ch. 6–7)。乱流境界層は渦によって外側の速い流体を壁の近くまで運ぶので、壁面の速度勾配が大きく、摩擦抗力は層流より大きい。その一方で、壁近くの流体が運動量を多くもつので剥離しにくい。

**剥離**は、圧力が下流へ向かって上がる区間(**逆圧力勾配 (adverse pressure gradient)**)で起きる。境界層の底の流体は粘性で運動量を失っているので、圧力の坂を登りきれずに止まり、やがて逆流して壁から離れる。剥離が起きると、物体の後ろに圧力の回復しない大きな後流 (wake) ができ、前後の圧力差による**圧力抗力 (pressure drag)** が急増する (Schlichting & Gersten 2017, Ch. 2)。

- **翼の失速**: 迎角を上げると上面後半の逆圧力勾配が強くなり、ある迎角で上面の境界層が大規模に剥離して C_L が急減する。失速が速度ではなく迎角で決まるのは、圧力分布の形が迎角で決まるからである。
- **ゴルフボールのディンプル**: 表面の凹凸で境界層を早めに乱流にすると、剥離点が後ろへ下がって後流が細くなる。摩擦は増えるが圧力抗力が大きく減り、全抗力が下がる。球や円柱の抗力係数が Re ≈ 2–4 × 10⁵ 付近で急落する**抗力危機 (drag crisis)** と同じ現象である。
- **ボルテックスジェネレータ・スラット**: 翼面の小さな突起や前縁の隙間から運動量の大きい流れを境界層に供給し、剥離を遅らせる。
- **カルマン渦列**: 円柱の後ろでは剥離した渦が左右交互に放出され、その周波数はストローハル数 St = f D / U ≈ 0.2 でほぼ決まる。電線の風切り音や構造物の渦励振の原因になる。

### 5. 揚力と抗力の由来 (the origin of lift and drag)

翼まわりの流れを「一様流 + 循環 Γ」で表すと、**クッタ=ジュコフスキーの定理 (Kutta–Joukowski theorem)** により、単位翼幅あたりの揚力は L' = ρ V Γ になる。循環 Γ = ∮ v·dl は翼を囲む閉曲線に沿った速度の線積分で、上面の流れが速く下面が遅いことを一つの数で表したものである (Anderson 2017, Ch. 3–4)。

ところが非粘性理論では Γ の値が一意に決まらず、どの Γ も方程式を満たしてしまう。現実には、鋭い後縁を回り込む流れは粘性のため維持できず、流れは後縁から滑らかに離れる。この**クッタ条件 (Kutta condition)** を課すと Γ が一つに決まる。薄翼理論では次の結果が得られる。

```text
薄翼理論:  C_L = 2π (α − α_L0)     (α: 迎角 [rad]、α_L0: ゼロ揚力角)
  揚力傾斜 2π /rad ≈ 0.11 /deg — 実測の翼型でもおおむね 0.1 /deg 前後
翼の始動時: 後縁から循環 −Γ の始動渦が流れ去り、翼まわりに +Γ が残る (ケルビンの循環定理)
```

したがって、揚力は非粘性の式 (L' = ρVΓ) で計算できるが、その大きさを決めているのは粘性である。運動量の見方では、翼は循環によって空気を下向きに曲げ(吹き下ろし)、その反作用として上向きの力を受ける。両者は同じ現象の別の記述である。「上面の経路が長いので速くなる」という等時間通過説は誤りで、実際には上面の流れは下面より先に後縁へ到達する (NASA Glenn Research Center)。

抗力は境界層の言葉で整理できる。**摩擦抗力 (skin-friction drag)** は壁面せん断応力 τ_w の積分で、境界層が層流か乱流かで決まる。**圧力抗力 (form drag)** は剥離による後流の大きさで決まる。流線形の物体は剥離を後ろへ追いやって圧力抗力を抑える形であり、鈍い物体では圧力抗力が支配的になる。有限翼幅の翼ではさらに、翼端渦による**誘導抗力 (induced drag)** が加わる。これは揚力をつくるために必然的に生じる抗力で、C_L²/(π e AR) に比例する。

## 工学への接続 (Engineering Connections)

- **航空機の空力設計は境界層の制御である**: 失速迎角、高揚力装置、層流翼型による摩擦抗力の削減は、剥離と遷移の位置をどこに置くかの設計である (§4)。巡航の L/D を上げる努力の多くは、摩擦抗力 (§3) と誘導抗力 (§5) の削減に向かう。
- **小型機・ドローンは低 Re の世界にある**: Re が 10⁴–10⁵ 程度では層流剥離泡ができやすく、翼型の性能が旅客機の翼から予想されるより大きく落ちる (§2, §4)。マルチロータのプロペラ効率や騒音の議論はこの領域の空気力学である。
- **冷却と配管は管内流れの問題である**: 液冷のコールドプレートや配管の圧力損失は、層流と乱流で摩擦の法則が変わる (§4)。熱伝達係数も乱流で大きくなるため、ポンプ電力と冷却性能が交換関係になる。
- **相似則が試験とシミュレーションをつなぐ**: 風洞試験と CFD の検証は Re とマッハ数をそろえることで成り立つ (§2)。高 Re の乱流を直接計算するのは今も計算量的に困難で、乱流モデルの選択が CFD の精度を左右する。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`engineering/aerospace/aircraft-engineering-fundamentals.md`](../../../engineering/aerospace/aircraft-engineering-fundamentals.md) | 揚力方程式と動圧 (§1)、クッタ=ジュコフスキーとクッタ条件 (§5)、失速と境界層剥離・高揚力装置 (§4)、摩擦・形状・誘導抗力 (§5) |
| [`engineering/aerospace/uav-multirotor-systems.md`](../../../engineering/aerospace/uav-multirotor-systems.md) | 低 Re のプロペラ・ロータ空力 (§2, §4) |
| [`engineering/datacenters/datacenter-cooling.md`](../../../engineering/datacenters/datacenter-cooling.md) | 空調と液冷配管の流れ、層流・乱流と圧力損失 (§4) |
| [`engineering/aerospace/reentry-thermal-protection.md`](../../../engineering/aerospace/reentry-thermal-protection.md) | 境界層の遷移と加熱率の増大 (§3–4 の高速版) |
| [`foundations/physics/fluid-dynamics/compressible-flow-nozzles.md`](compressible-flow-nozzles.md) | M → 0 の極限としての非圧縮流 (§1)、ノズル内の剥離 (§4) |
| [`foundations/physics/fluid-dynamics/shock-waves-hypersonics.md`](shock-waves-hypersonics.md) | 衝撃波と境界層の干渉、造波抗力との対比 (§5) |

## 参考文献 (References)

- Anderson, J. D. (2017). *Fundamentals of Aerodynamics* (6th ed.). McGraw-Hill. — 非圧縮流とベルヌーイの定理、ダランベールのパラドックス、循環とクッタ=ジュコフスキーの定理 (Ch. 3)、薄翼理論 (Ch. 4)。§1, §3, §5 の主参照
- White, F. M. (2011). *Fluid Mechanics* (7th ed.). McGraw-Hill. — 粘性とナビエ=ストークス方程式 (Ch. 4)、管内流れの遷移 (Ch. 6)、境界層と物体まわりの流れ (Ch. 7)。§2, §4 の参照
- Schlichting, H., & Gersten, K. (2017). *Boundary-Layer Theory* (9th ed.). Springer. DOI: 10.1007/978-3-662-52919-5 — Prandtl の境界層概念、Blasius 解、遷移と剥離の標準的参照。§3–4 の主参照
- Prandtl, L. (1904). Über Flüssigkeitsbewegung bei sehr kleiner Reibung. *Verhandlungen des III. Internationalen Mathematiker-Kongresses*, Heidelberg, 484–491. — 境界層概念の原典
- Blasius, H. (1908). Grenzschichten in Flüssigkeiten mit kleiner Reibung. *Zeitschrift für Mathematik und Physik*, 56, 1–37. — 平板層流境界層の相似解
- [NASA Glenn Research Center — Beginner's Guide to Aeronautics: Equal Transit Theory Interactive](https://www1.grc.nasa.gov/beginners-guide-to-aeronautics/foilw1/) — Accessed: 2026-11-28. 等時間通過説が誤りである理由の解説
