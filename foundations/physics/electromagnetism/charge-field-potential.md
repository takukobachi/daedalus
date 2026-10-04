# 電荷・電場・電位 (Charge, Electric Field, and Potential)

## 概要 (Overview)

電磁気学 (electromagnetism) シリーズの第1回として、**電荷 (electric charge)** から出発し、**電場 (electric field)**・**電位 (electric potential)**・**キャパシタンス (capacitance)** までの静電気学 (electrostatics) の骨格を積み上げる。

この範囲を最初に置くのは、学習順序として微積分の直後に位置するからだけではない。**計算機の中の「ビット」の物理的実体は、ほぼすべて「どこかに蓄えられた電荷」であり、その蓄え方を定量化する概念がキャパシタンスだからである。** DRAM セルはキャパシタの電荷、NAND フラッシュはフローティングゲートに閉じ込めた電荷、CMOS の消費電力は配線とゲートのキャパシタンスの充放電で決まる。本ドキュメントは、その一連の言明を支える最下層の物理を、法則→概念→定量化の順で構成する。

```mermaid
graph LR
    Q["電荷 q"] -->|"クーロンの法則<br/>(力)"| E["電場 E"]
    E -->|"仕事・エネルギー<br/>E = −∇V"| V["電位 V"]
    Q --> C["キャパシタンス<br/>C = Q / V"]
    V --> C
    C -->|"U = ½CV²"| U["蓄積エネルギー"]
```

## 前提知識 (Prerequisites)

- 高校物理程度の力学(力、仕事、エネルギー)
- 微積分の初歩(偏微分、積分。勾配 ∇ は本文中で導入する)
- ベクトルの内積 — [`../../mathematics/linear-algebra/vectors-matrices-linear-maps.md`](../../mathematics/linear-algebra/vectors-matrices-linear-maps.md) の §2

## 本論 (Main Content)

### 1. 電荷とクーロンの法則 (charge and Coulomb's law)

**電荷 (electric charge)** は物質の基本的な属性で、正・負の2種類があり、同符号は反発し異符号は引き合う。電荷は**量子化 (quantization)** されており、観測される電荷は**電気素量 (elementary charge)** の整数倍で現れる。電気素量は 2019 年の SI 改定で定義値となった。

```text
e = 1.602 176 634 × 10⁻¹⁹ C (クーロン)   — 定義値 (BIPM 2019)
```

また電荷は**保存量 (conserved quantity)** である——閉じた系の総電荷は変化せず、移動・再配置するだけである。半導体で「電子と正孔 (hole)」を対で生成・再結合として扱えるのは、この保存則の帰結である。

2つの点電荷 `q₁, q₂` が距離 `r` を隔てて及ぼし合う力は**クーロンの法則 (Coulomb's law)** で与えられる (Griffiths 2023, §2.1)。

```text
F = (1 / 4πε₀) · q₁q₂ / r²        ε₀ ≈ 8.854 × 10⁻¹² F/m (真空の誘電率)
```

逆2乗則・中心力・重ね合わせ可能という3つの性質が、以降のすべての構造(ガウスの法則、ポテンシャルの存在)を決めている。

### 2. 電場 (electric field)

クーロンの法則は「離れた電荷が直接力を及ぼす」形をしているが、電磁気学はこれを2段階に分けて読む: **電荷は空間に「場 (field)」を作り、他の電荷はその場から力を受ける。**

```text
E(r) = F / q    (その点に置いた微小試験電荷が受ける、単位電荷あたりの力)
```

場という見方は静電気学の範囲では計算の書き換えに過ぎないが、時間変動を含めた瞬間に本質になる——電荷を急に動かしたとき、遠方の電荷が受ける力は即座には変わらず、変化は場の波(電磁波)として有限速度で伝わる。場は「実在」であり、エネルギーと運動量を運ぶ (Feynman, Leighton & Sands 1964, Vol. II, Ch. 1)。この転回はシリーズ後半(マクスウェル方程式と電磁波)で回収する。

複数の電荷が作る電場は各電荷の作る電場のベクトル和である(**重ね合わせの原理 (superposition principle)**)。これは電磁気学が「線形な理論」であることの表明で、線形代数の言葉がそのまま通用する土台でもある。

#### ガウスの法則 (Gauss's law)

任意の閉曲面を貫く電場の総流束 (flux) は、内部の総電荷だけで決まる (Griffiths 2023, §2.2)。

```text
∮ E · dA = Q_enclosed / ε₀
```

逆2乗則と等価だが、対称性の高い配置(球・円筒・平面)では電場を積分一発で求められる実用的な形であり、後にマクスウェル方程式の第1式としてそのまま登場する。

### 3. 電位 (electric potential)

静電場は**保存力場 (conservative field)** であり、電荷を動かす仕事が経路によらない。したがって各点に「単位電荷あたりの位置エネルギー」=**電位 (electric potential)** `V` が定義でき、電場は電位の**勾配 (gradient)** で書ける。

```text
E = −∇V        ∇V = (∂V/∂x, ∂V/∂y, ∂V/∂z)
```

単位はボルト (volt): `1 V = 1 J/C`。電位はスカラーであり、ベクトルの電場より計算・測定・直観のすべてで扱いやすい。回路解析が電流と**電圧**(=2点間の電位差)だけで閉じるのは、この「ベクトル場をスカラー1個に圧縮できる」性質の恩恵である。

**物理的な意味を持つのは電位差 (potential difference) だけ**で、基準点(接地, ground)の取り方は任意である点が重要である。デジタル回路の H/L も、電源レールに対する電位差の閾値判定であり、「絶対的な電圧」なるものは存在しない。基準(グラウンド)の設計が回路設計の中心課題の1つになるのはこのためである。

### 4. 導体と静電遮蔽 (conductors and electrostatic shielding)

**導体 (conductor)** は自由に動ける電荷を持つ物質である。静電平衡では、内部に電場が残っていれば電荷が動いてそれを打ち消すため、

1. 導体内部の電場はゼロ
2. 電荷はすべて表面に分布する
3. 導体表面は等電位面 (equipotential surface) になる

が成り立つ (Purcell & Morin 2013, §3.1–3.2)。帰結として、導体で囲まれた空洞は外部の静電場から遮蔽される(**ファラデーケージ (Faraday cage)**)。シールドケーブル・筐体シールド・コネクタのシェルなど、EMI (electromagnetic interference) 対策の原型がここにある(時間変動する場に対する遮蔽は完全ではなく、周波数と表皮効果の話としてシリーズ後半で扱う)。

### 5. キャパシタンス (capacitance)

導体に電荷 `Q` を与えると電位 `V` が上がり、静電気学の線形性から両者は比例する。この比例係数が**キャパシタンス (capacitance)** である。

```text
C = Q / V    [F (ファラド) = C/V]
```

`C` は形状と材質だけで決まる幾何学的な量である。代表例の平行平板キャパシタ(面積 `A`、間隔 `d`、比誘電率 εᵣ の誘電体を挟む)では

```text
C = ε₀ εᵣ A / d
```

となり (Griffiths 2023, §2.5)、「広く・薄く・高誘電率に」がキャパシタンスを稼ぐ3つの手段だと分かる。蓄えられるエネルギーは

```text
U = ½ C V² = Q² / 2C
```

**誘電体 (dielectric)** は電場中で分極 (polarization) して実効的な電場を弱め、同じ電荷でも電位を下げる=キャパシタンスを εᵣ 倍に増やす。半導体産業で SiO₂ (εᵣ ≈ 3.9) を基準に、それより大きい絶縁膜を **high-k**、小さい層間絶縁膜を **low-k** と呼ぶ (Sze & Ng 2007, Appendix G)。ゲートには「薄くしてもリークしない high-k」を、配線間には「寄生容量を減らす low-k」を使う——同じ式 `C = ε₀εᵣA/d` の、増やしたい側と減らしたい側である。

## 工学への接続 (Engineering Connections)

- **ビット=蓄えた電荷**: DRAM セルは「キャパシタに電荷があるか否か」、NAND フラッシュは「フローティングゲートに閉じ込めた電荷の量」でビットを表現する。`C = Q/V` は「セルを小さくすると保持できる電荷が減り、読み出しマージンとリーク耐性が落ちる」というスケーリングの根本制約そのもの。
- **消費電力**: CMOS 論理の動的電力は、ノードのキャパシタンス C を電圧 V まで充放電する繰り返しで決まり、`P_dyn ≈ α C V² f` に従う (Weste & Harris 2011, §5.1)。電圧を下げることが二乗で効く理由は `U = ½CV²` に既に書かれている。
- **寄生容量と速度**: 配線・トランジスタのあらゆる導体対は意図せぬキャパシタ(寄生容量, parasitic capacitance)を形成し、充電が終わるまで信号は遷移しない。「配線が長い=遅い」の一次的な物理は RC 充電である。
- **遮蔽と基準電位**: ファラデーケージが EMI シールドの原型であり、「意味を持つのは電位差だけ」がグラウンド設計・シングルエンド/差動伝送の出発点になる。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/cpu-architecture/dram-memory-controllers.md`](../../../hardware/cpu-architecture/dram-memory-controllers.md) | DRAM セル=1T1C のキャパシタ電荷。リーク→リフレッシュ必須という同文書の前提が §5 の物理 |
| [`hardware/storage-devices/nand-flash-ftl.md`](../../../hardware/storage-devices/nand-flash-ftl.md) | フローティングゲートへの電荷注入・保持がビットの実体。多値化 (MLC/TLC/QLC) は電荷量の分解能の問題 |
| [`hardware/hw-sw-interface/clock-power-reset.md`](../../../hardware/hw-sw-interface/clock-power-reset.md) | 動的電力 CV²f と DVFS。電圧スケーリングが二乗で効く根拠 |
| [`hardware/semiconductor-manufacturing/semiconductor-fabrication.md`](../../../hardware/semiconductor-manufacturing/semiconductor-fabrication.md) | high-k ゲート絶縁膜・low-k 層間絶縁膜。§5 の誘電体の工学的使い分け |
| [`hardware/hardware-security/side-channels-tee-secure-boot.md`](../../../hardware/hardware-security/side-channels-tee-secure-boot.md) | Rowhammer は隣接行アクセスによるセル電荷のリーク誘発。電力解析は CV²f がデータ依存に揺らぐことの悪用 |

## 参考文献 (References)

- Griffiths, D. J. (2023). *Introduction to Electrodynamics* (5th ed.). Cambridge University Press. — 静電気学の主参照
- Purcell, E. M., & Morin, D. J. (2013). *Electricity and Magnetism* (3rd ed.). Cambridge University Press. — 導体・遮蔽と物理的直観の副読本
- Feynman, R. P., Leighton, R. B., & Sands, M. (1964). *The Feynman Lectures on Physics, Vol. II*. Addison-Wesley. [オンライン版](https://www.feynmanlectures.caltech.edu/) — Accessed: 2026-07-11
- 砂川重信 (1999). 『理論電磁気学』(第3版) 紀伊國屋書店. — 日本語の標準的教科書
- BIPM (2019). *The International System of Units (SI Brochure)* (9th ed.). [https://www.bipm.org/en/publications/si-brochure](https://www.bipm.org/en/publications/si-brochure) — Accessed: 2026-07-11。電気素量の定義値
- Sze, S. M., & Ng, K. K. (2007). *Physics of Semiconductor Devices* (3rd ed.). Wiley. — 誘電率・絶縁膜の物性値
- Weste, N. H. E., & Harris, D. M. (2011). *CMOS VLSI Design: A Circuits and Systems Perspective* (4th ed.). Addison-Wesley. — 動的電力 αCV²f の主参照
