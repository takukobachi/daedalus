# 圧縮性流れとノズル理論 (Compressible Flow & Nozzle Theory)

## 概要 (Overview)

ロケットエンジンの心臓部は燃焼室だが、**推力を生むのはノズル**である。高温高圧のガスを「先細り→末広がり」というデ・ラバルノズル (de Laval nozzle) に通すと、熱エネルギーが整然と運動エネルギーへ変換され、排気は超音速に達する。なぜ**絞ってから広げる**と速くなるのか、なぜスロットで必ず音速 (M = 1) になるのか、なぜ同じエンジンでも海面用と真空用でノズルの長さが違うのか——本ドキュメントは、これらを圧縮性流れ (compressible flow) の基礎から積み上げて説明する。

到達点は3つの式である: 面積–速度関係（ノズル形状の理由）、チョーク流量（推力の「蛇口」がスロットにある理由）、そして推力式と排気速度（比推力 Isp と分子量・燃焼温度の関係）。`hardware/aerospace/` の SpaceX 分析で結果だけ使ってきた数値（膨張比、海面/真空 Isp 差）の理論的根拠を、ここで供給する。

## 前提知識 (Prerequisites)

- **古典力学** — 運動量保存・エネルギー保存（[`../classical-mechanics/newton-laws-oscillations.md`](../classical-mechanics/newton-laws-oscillations.md)）
- **熱力学** — 第一法則、断熱過程、エンタルピーの概念（[`../thermodynamics-statistical/laws-entropy-heat-engines.md`](../thermodynamics-statistical/laws-entropy-heat-engines.md)）
- **微分積分** — 微小量の関係式の操作（`../../mathematics/calculus-analysis/`）
- ロケット側の文脈: ロケット方程式と Isp（`../../../hardware/aerospace/spacex-rocket-engines.md`）

## 本論 (Main Content)

### 1. 圧縮性とマッハ数 (compressibility and the Mach number)

低速の流れ（自動車、送風機）では空気の密度変化を無視できるが、流速が**音速**に近づくと密度変化が本質になる。音速 `a` は「圧力の乱れが伝わる速さ」であり、理想気体では

```
a = √(γRT)      γ: 比熱比, R: 気体定数(質量基準), T: 温度
```

流速と音速の比が**マッハ数** `M = v/a`。M < 1（亜音速）と M > 1（超音速）で流れの性質は質的に変わる——その分水嶺が、ノズル理論の主役である。

### 2. 面積–速度関係 — なぜ「絞ってから広げる」のか (the area–velocity relation)

摩擦・熱損失を無視した1次元の定常断熱流（等エントロピー流）では、連続の式・運動量の式・状態方程式を組み合わせると、断面積 `A` と流速 `v` の微小変化が次の関係で結ばれる。

```
dA/A = (M² − 1) · dv/v
```

この1本の式がノズルの形を決める。

| 領域 | 加速したい (dv > 0) なら |
|---|---|
| 亜音速 (M < 1) | `M² − 1 < 0` → **dA < 0: 絞る**（先細り） |
| 音速 (M = 1) | `dA = 0`: **断面積が極小の点（スロート）でのみ M=1 になれる** |
| 超音速 (M > 1) | `M² − 1 > 0` → **dA > 0: 広げる**（末広がり） |

- **直観**: 亜音速では密度がほぼ変わらないため、細くすれば速くなる（ホースの先をつまむ）。超音速では速度が上がると**密度が急激に落ちる**ため、同じ質量流量を通すには断面をむしろ広げる必要がある。
- したがって、亜音速の燃焼ガスを超音速まで加速する唯一の定常的な形が、**先細り（加速）→スロート (M=1) →末広がり（さらに加速）** のデ・ラバルノズルである。

```
 燃焼室 ──▶ 先細り部 ──▶ スロート ──▶ 末広がり部 ──▶ 排気
 (M≈0)      (M<1 加速)    (M=1)       (M>1 加速)      (M_e)
 p0, T0     圧力・温度を運動エネルギーへ変換していく     p_e, v_e
```

### 3. チョーク流量 — 推力の蛇口はスロートにある (choked flow)

スロートで M = 1 に達すると、流量はそれ以上増えない（**チョーク**）。チョーク時の質量流量は

```
ṁ = A* · p0 / √(T0) · Γ(γ)      A*: スロート断面積, p0/T0: 燃焼室全圧/全温
                                  Γ(γ): γ だけで決まる定数
```

- **含意 1**: 下流（ノズル出口側・外気圧）を何をどう変えても、**流量は燃焼室条件とスロート面積だけで決まる**。エンジンのスロットリングとは p0（燃焼室圧）の制御である。
- **含意 2**: 燃焼室圧 p0 を上げると、同じ流量を**より小さいスロート**で流せる。Raptor が燃焼室圧に執着する理由の一つがここにある（後述の膨張比の項も参照）。

### 4. 推力式と最適膨張 (the thrust equation and optimum expansion)

ノズルを含む制御体積の運動量収支から、推力は

```
F = ṁ·v_e + (p_e − p_a)·A_e
      │          │
      │          └ 圧力項: 出口圧 p_e と外気圧 p_a の差 × 出口面積
      └ 運動量項: 質量流量 × 排気速度
```

- **最適膨張**: 固定の外気圧 p_a に対し、推力が最大になるのは **p_e = p_a**（出口圧＝外気圧）のとき。
  - **過小膨張 (underexpansion)**: p_e > p_a。膨張しきれず、エネルギーを排気の圧力として捨てている（真空中の短いノズル）。
  - **過膨張 (overexpansion)**: p_e < p_a。広げすぎで圧力項が負になる。さらに進むと**ノズル内で流れが剥離**し、非対称な横荷重が生じて危険——地上で真空用ノズルを全開にできない理由。
- **膨張比** `ε = A_e/A*`（出口面積/スロート面積）が p_e を決める設計変数である。外気圧は高度で変わるが ε は固定なので、**1つのノズルは1つの高度でしか最適にならない**。

### 5. 排気速度と Isp — 何が「燃費」を決めるか (exhaust velocity and Isp)

等エントロピー膨張のエネルギー収支（エンタルピー→運動エネルギー）から、理想排気速度は

```
v_e = √( 2γ/(γ−1) · R̄T0/M̄ · [1 − (p_e/p0)^((γ−1)/γ)] )
             │        │              │
             │        │              └ 膨張の深さ: p0/p_e が大きいほど速い
             │        └ 燃焼温度 T0 / 平均分子量 M̄ — v_e ∝ √(T0/M̄)
             └ ガス物性
```

比推力は `Isp = v_e/g0`（最適膨張時）。この式が推進剤化学のほぼすべてを説明する。

- **v_e ∝ √(T0/M̄)**: 燃焼温度が高く、**排気の分子量が軽い**ほど速い。水素/酸素 (H₂O 主体, M̄ 小) が最高の Isp を持ち、ケロシン (CO₂ など重い分子) が低く、**メタンがその中間**に来る序列は、ここから直接出る。
- **p0/p_e の対数的な効き**: 燃焼室圧を上げ、深く膨張させるほど v_e は伸びる——ただし平方根内の `1−(p_e/p0)^…` は頭打ちがあり、圧力比を上げる利得は逓減する。
- **仮定の注意**: この式は「凍結流」（膨張中に組成が変わらない）近似で、実際は解離・再結合（平衡流）で数%変わる。設計計算では化学平衡コード (CEA 等) を使う。ここでは**傾向を支配する構造**が要点である。

### 6. 海面ノズルと真空ノズル (sea-level vs vacuum nozzles)

同じエンジンでも、働く高度でノズルが変わる。

| | 海面用 | 真空用 |
|---|---|---|
| 外気圧 p_a | ~1 bar | ~0 |
| 最適な膨張比 ε | **小さい**（広げすぎると過膨張・剥離） | **大きい**（いくら広げても良い。制約は質量と長さ） |
| 例 (概算・公表値) | Merlin 1D: ε ≈ 16 / Raptor 海面型: ε ≈ 34 | Merlin Vacuum: ε ≈ 165 / Raptor Vacuum: ε ≈ 80 級 |
| Isp への効果 | 低め（浅い膨張） | 同系エンジンで数十秒高い |

- **第1段と第2段で Isp が違う理由**はエンジンの優劣ではなく、主に**外気圧と許される膨張比の差**である（Falcon 9 の Merlin と MVac、Starship の海面 Raptor と RVac の使い分け）。
- **燃焼室圧と海面ノズルの関係**: p0 が高いほど、同じ出口圧 p_e を保ったまま**より大きな ε を海面で使える**（p_e = p0 × f(ε) なので）。Raptor の高燃焼室圧（`spacex-rocket-engines.md`）は、チョーク流量の項の「小さいスロート」と合わせて、海面 Isp の底上げに直結している。
- **高度補償の研究**: 1つのノズルで全高度に適応するエアロスパイクなどの概念があるが、冷却・重量の難しさから実用機では「高度別に2種類のノズルを積む」現実解が支配的である。

## 工学への接続 (Engineering Connections)

- **ノズルは「熱→運動」の変換器であり、効率の上限は熱力学が決める**。燃焼室温度 T0 を上げる（≒カルノー的な高温源の確保）、分子量を下げる、深く膨張させる——Isp 向上の手段はすべて v_e の式の中にある。エンジンサイクルの工夫（`spacex-rocket-engines.md` の FFSC）は、この式の p0 と T0 を上げるための機械側の努力と読める。
- **設計は「1点最適 × 制約」**: 膨張比は打ち上げプロファイル上の代表高度への最適化であり、第1段は剥離限界、上段は質量・全長が実効的な制約になる。多段化（`../../../hardware/aerospace/orbital-mechanics.md`「なぜ多段か」）は、ノズルの高度最適化問題でもある。
- **チョークという設計の錨**: 「下流条件に流量が依存しない」性質は、エンジン制御を燃焼室圧制御に単純化し、複数エンジンのクラスタ（9基/33基）でも各エンジンの流量が互いに干渉しない土台を与える。
- **ジェットエンジンとの対比**: 航空機のジェット（`../../../hardware/aerospace/aircraft-engineering-fundamentals.md`）は通常、先細りノズルのみ（排気は M ≤ 1 付近）。ロケットが末広がり部を持つのは、外気を使わず自前の高圧ガスを**真空近くまで膨張させ切る**必要があるからである。

## Daedalus内の応用 (Applications within Daedalus)

- [`../../../hardware/aerospace/spacex-rocket-engines.md`](../../../hardware/aerospace/spacex-rocket-engines.md) — Merlin/Raptor の Isp・燃焼室圧・推力の数値の理論的根拠（本ドキュメントの v_e 式・チョーク流量・膨張比がその土台）
- [`../../../hardware/aerospace/orbital-mechanics.md`](../../../hardware/aerospace/orbital-mechanics.md) — ロケット方程式の v_e がここで決まる。多段化の必然性との接続
- [`../../../hardware/aerospace/spacex-starship-superheavy.md`](../../../hardware/aerospace/spacex-starship-superheavy.md) — 海面3基＋真空3基という Starship のエンジン構成の理由
- [`../../../hardware/aerospace/aircraft-engineering-fundamentals.md`](../../../hardware/aerospace/aircraft-engineering-fundamentals.md) — ジェット推進（亜音速ノズル）との対比
- [`../thermodynamics-statistical/laws-entropy-heat-engines.md`](../thermodynamics-statistical/laws-entropy-heat-engines.md) — 「熱を仕事に変える」一般論。ノズルはその連続流版

## 参考文献 (References)

- Anderson, J. D. (2020). *Modern Compressible Flow: With Historical Perspective* (4th ed.). McGraw-Hill — 圧縮性流れ・面積–速度関係・ノズル流れの標準教科書。
- Sutton, G. P., & Biblarz, O. (2016). *Rocket Propulsion Elements* (9th ed.). Wiley — 推力式、理想排気速度、膨張比設計、剥離限界などロケット応用の標準典拠。
- [NASA Glenn Research Center — Isentropic Flow / Nozzle Design (educational pages)](https://www.grc.nasa.gov/www/k-12/airplane/) — 等エントロピー流とノズルの入門解説。Accessed: 2026-07-26.
- Gordon, S., & McBride, B. J. (1994). Computer Program for Calculation of Complex Chemical Equilibrium Compositions and Applications. *NASA RP-1311* — 化学平衡計算 (CEA)。凍結流近似を超える実務計算の典拠。
