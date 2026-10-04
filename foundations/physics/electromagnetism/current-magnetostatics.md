# 電流と静磁気 (Electric Current and Magnetostatics)

## 概要 (Overview)

電磁気学シリーズの第2回。前回の「止まっている電荷」から「**流れる電荷**」へ進む。**電流 (electric current)** とその微視的な実体、**オームの法則 (Ohm's law)** がなぜ成り立つか(そしていつ破れるか)、電流が発生させる**磁場 (magnetic field)**、磁場が電流に及ぼす**ローレンツ力 (Lorentz force)** までを積み上げる。

この範囲は計算機ハードウェアの「配線」の物理である。配線抵抗と RC 遅延、電源分配の IR ドロップ、ジュール発熱、エレクトロマイグレーション——分析文書が前提にしている配線の制約は、すべて本ドキュメントの範囲に根を持つ。同時に、ローレンツ力はモーターの動作原理であり、EV パワートレインの物理的起点でもある。

## 前提知識 (Prerequisites)

- [`charge-field-potential.md`](charge-field-potential.md) — 電荷、電場、電位
- ベクトルの内積([`../../mathematics/linear-algebra/vectors-matrices-linear-maps.md`](../../mathematics/linear-algebra/vectors-matrices-linear-maps.md) §2)。外積 (cross product) は本文中で導入する

## 本論 (Main Content)

### 1. 電流と電荷保存 (current and charge conservation)

**電流 (current)** `I` は断面を単位時間に通過する電荷量 `I = dQ/dt`(単位: アンペア, `1 A = 1 C/s`)、**電流密度 (current density)** `J` はその単位面積あたりのベクトル版である。電荷保存則は微分形で**連続の式 (continuity equation)** になる (Griffiths 2023, §5.1)。

```text
∂ρ/∂t + ∇·J = 0    (電荷が減る場所からは、その分の電流が流れ出している)
```

回路解析のキルヒホッフ電流則(節点に流れ込む電流の総和はゼロ)は、この式の「電荷が溜まらない定常状態」での言い換えである。

### 2. オームの法則の微視的描像 (microscopic picture of Ohm's law)

金属中の伝導電子は電場で一様に加速され続けるわけではなく、格子欠陥や格子振動に散乱されながら進む。加速と散乱が釣り合った結果、電子集団は電場に比例した平均速度=**ドリフト速度 (drift velocity)** で流れる。平均散乱時間 τ を使うドルーデモデル (Drude model) では次式が成り立つ (Ashcroft & Mermin 1976, Ch. 1)。

```text
J = σE,   σ = n e² τ / m    (n: キャリア密度, m: 電子質量)
```

これが**オームの法則**の微視的な形で、**電気伝導率 (conductivity)** σ(逆数が**抵抗率 (resistivity)** ρ)は材料定数になる。長さ L・断面積 S の一様な導体に積分すればおなじみの巨視的な形になる。

```text
V = IR,   R = ρL / S
```

この導出は同時に法則の適用限界も示している——σ が定数なのは散乱の仕組みが変わらない範囲であり、温度が上がれば格子振動が増えて ρ は上がり(金属の正の温度係数)、半導体ではキャリア密度 n 自体が温度・電場に強く依存するためオームの法則は素直には成り立たない。

散乱で失われるエネルギーは熱になる。単位時間の発熱は**ジュール熱 (Joule heating)** である。

```text
P = IV = I²R = V²/R
```

「配線に電流を流すこと自体が発熱源である」というこの式は、電力輸送の損失(高電圧送電の理由)から、チップの電源網の発熱まで同じ形で効く。

### 3. 磁場とローレンツ力 (magnetic field and the Lorentz force)

電流は周囲に**磁場 (magnetic field)** `B` を作り、磁場は動く電荷に力を及ぼす。力の法則が**ローレンツ力 (Lorentz force)** である (Griffiths 2023, §5.1)。

```text
F = q(E + v × B)
```

ここで `v × B` は**外積 (cross product)**——2つのベクトルの両方に直交し、大きさが `|v||B|sinθ` のベクトル——であり、磁気力は常に速度と直交する。したがって**磁場は仕事をしない**: 電荷の速さは変えず、向きだけを曲げる(サイクロトロン運動)。電流の流れる導線が磁場から受ける力は、これをキャリアについて足し上げた `F = IL × B`(L: 電流方向の長さベクトル)で、**モーターの動作原理はこの1行**である。

### 4. 電流が作る磁場 (magnetic fields from currents)

定常電流が作る磁場は**ビオ・サバールの法則 (Biot–Savart law)** で与えられ、対称性が高い配置では**アンペールの法則 (Ampère's law)** から直ちに求まる (Griffiths 2023, §5.2–5.3)。

```text
∮ B · dl = μ₀ I_enclosed        μ₀ ≈ 4π × 10⁻⁷ H/m (真空の透磁率)
```

代表例を示す。

- **直線電流**: `B = μ₀I / 2πr` — 距離に反比例。隣の配線に磁場が届く=磁気的クロストークの種
- **ソレノイド(コイル)**: 内部で `B = μ₀ n I`(n: 単位長あたり巻数) — 磁場を「濃縮」する装置。インダクタ・トランス・電磁石の原型

ガウスの法則(電場)と対をなす磁場側の法則は `∮ B · dA = 0`——磁力線は必ず閉じ、**磁気単極子 (magnetic monopole) は存在しない**(観測されていない)。「電流ループが磁場を作り、磁場が電流ループに力を及ぼす」という電流⇔磁場の相互作用が見えたところで、次回の電磁誘導——磁場の**変化**が電場を作る——への準備が整う。

## 工学への接続 (Engineering Connections)

- **配線は理想導線ではない**: `R = ρL/S` は、微細化で配線断面積が縮むほど抵抗が増えることを意味する。前回の寄生容量と合わせた RC 遅延が、ロジックの微細化ほどには配線が速くならない理由の一次近似である。
- **電源分配網 (PDN) と IR ドロップ**: 電源からトランジスタまでの配線抵抗は、大電流時に `V = IR` ぶんの電圧降下を生む。「電圧が足りない」障害の物理はオームの法則そのものである。
- **エレクトロマイグレーション (electromigration)**: 高い電流密度 J では、電子流が金属原子を押し流して配線を断線・短絡させる。J(=単位面積あたり)で決まるため、これも微細化とともに厳しくなる配線寿命の制約になる。
- **モーターと発電の物理**: `F = IL × B` がモーター(電流→力)、次回の電磁誘導が発電機(運動→起電力)であり、両者は同じ装置の双方向動作である。EV のトラクションモーターの物理的起点。
- **磁気クロストークとインダクタンス**: 電流ループが磁場を作る以上、あらゆる電流経路は磁気的に結合しうる。この定量化(インダクタンス)は次回導入し、電源のデカップリングや高速信号のリターンパス設計に接続する。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/semiconductor-manufacturing/semiconductor-fabrication.md`](../../../hardware/semiconductor-manufacturing/semiconductor-fabrication.md) | 配線(BEOL)の抵抗率・エレクトロマイグレーション。銅配線・バリアメタルの動機 |
| [`hardware/hw-sw-interface/clock-power-reset.md`](../../../hardware/hw-sw-interface/clock-power-reset.md) | 電源分配・IR ドロップ・ジュール発熱。§2 のオームの法則と P = I²R が前提 |
| [`hardware/hw-sw-interface/bus-interconnect.md`](../../../hardware/hw-sw-interface/bus-interconnect.md) | 配線の RC 遅延とクロストーク。高速化の物理的制約の起点 |
| [`engineering/automotive/ev/tesla-battery-powertrain.md`](../../../engineering/automotive/ev/tesla-battery-powertrain.md) | トラクションモーターの動作原理(ローレンツ力)と大電流経路の I²R 損失 |
| [`engineering/energy/`](../../../engineering/energy/) | 送電の I²R 損失と高電圧化の理由。発電機・変圧器の物理は次回(電磁誘導)で完結 |

## 参考文献 (References)

- Griffiths, D. J. (2023). *Introduction to Electrodynamics* (5th ed.). Cambridge University Press. — 電流・静磁気の主参照
- Purcell, E. M., & Morin, D. J. (2013). *Electricity and Magnetism* (3rd ed.). Cambridge University Press. — 磁場を特殊相対論から導く独自の視点を持つ副読本
- Ashcroft, N. W., & Mermin, N. D. (1976). *Solid State Physics*. Holt, Rinehart and Winston. — ドルーデモデルと電気伝導の主参照
- 砂川重信 (1999). 『理論電磁気学』(第3版) 紀伊國屋書店. — 日本語の標準的教科書
