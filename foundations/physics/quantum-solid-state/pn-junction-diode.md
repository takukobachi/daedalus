# pn 接合とダイオード (The pn Junction and Diodes)

## 概要 (Overview)

量子力学・固体物理シリーズの第2回。前回はドーピングで n 型(電子過剰)と p 型(正孔過剰)の半導体を作れることを見た。本回はその2つを**接合する**——すると、接合面に自発的に**空乏層 (depletion region)** と内蔵電位が形成され、電流を一方向にしか流さない**整流 (rectification)** という非線形性が生まれる。

この非線形性が半導体エレクトロニクスの出発点である。抵抗・キャパシタ・インダクタ(電磁気学シリーズ)はすべて線形素子であり、線形回路だけでは増幅もスイッチングも論理演算もできない。pn 接合は「電圧で制御できる非対称性」を初めて提供し、ダイオードとして単独でも(整流・ESD 保護・発光・受光)、MOSFET の構成部品としても(次回)、現代のあらゆるチップの中に文字通り無数に存在する。本回は、キャリアの2つの輸送機構(拡散とドリフト)から空乏層の形成を導き、ダイオード方程式、降伏現象、そして光との相互作用(フォトダイオード・LED・太陽電池)までを扱う。

## 前提知識 (Prerequisites)

- [`band-theory-semiconductors.md`](band-theory-semiconductors.md) — バンド、電子と正孔、ドーピング、フェルミ準位
- [`../electromagnetism/charge-field-potential.md`](../electromagnetism/charge-field-potential.md) — 電場と電位、キャパシタンス
- [`../../mathematics/calculus-analysis/limits-derivatives-integrals.md`](../../mathematics/calculus-analysis/limits-derivatives-integrals.md) — 指数関数(ダイオード方程式)

## 本論 (Main Content)

### 1. キャリア輸送の2機構 — ドリフトと拡散 (drift and diffusion)

半導体中のキャリアは2つの駆動力で動く (Sze & Ng 2007, Ch. 1)。

```text
ドリフト (drift):     電場が押す。 J_drift ∝ E   (オームの法則の起源、第2回既出)
拡散 (diffusion):     濃度勾配が押す。 J_diff ∝ −dn/dx   (濃い方から薄い方へ)
```

拡散はインクが水に広がるのと同じ統計現象であり、電場がなくても濃度差があれば電流が流れる——金属にはなかった、半導体特有の自由度である(金属では電子濃度が一様で拡散項が働かない)。2つの機構は独立ではなく、熱平衡では**アインシュタインの関係** `D/μ = kT/q`(拡散係数と移動度の比が熱電圧 `kT/q ≈ 26 mV @300 K` で結ばれる)を満たす。この熱電圧が、以降のすべての式に現れる自然な電圧スケールになる。

### 2. 接合の形成 — 空乏層と内蔵電位 (junction formation: depletion region and built-in potential)

p 型と n 型を接合した直後を考える。n 側は電子が濃く p 側は薄いから、電子は p 側へ**拡散**する(正孔は逆向き)。拡散したキャリアは相手側で再結合して消え、接合面の両側に**動けないイオン化ドーパント**(n 側に正、p 側に負)が裸で残る——この電荷が電場を作り、その電場がさらなる拡散を**ドリフトで押し返す**。拡散とドリフトが釣り合ったところで平衡に達し、接合面にキャリアの枯れた**空乏層**と、それを横切る**内蔵電位 (built-in potential)** V_bi が確立する。

```text
V_bi = (kT/q)·ln(N_A·N_D/nᵢ²)   — Si の典型値: 0.6〜0.8 V
空乏層幅: W ∝ √(V_bi − V)/N     — ドーピングが濃いほど薄い
```

バンド図では、平衡の条件は**フェルミ準位が全体で一直線に揃う**こと(前回 §4 で予告)であり、その帰結としてバンドが接合部で曲がる (band bending)。「異種材料を接合するとフェルミ準位が揃うようにバンドが曲がり、界面に電位障壁ができる」——この1文は pn 接合に限らず、金属–半導体接合(ショットキー障壁)、MOS 構造(次回)、ヘテロ接合まで、デバイス物理全体を貫く原理である。空乏層は電荷を隔てた絶縁領域なので**キャパシタンス**を持ち、電圧で幅が変わるため可変容量(バラクタ)として振る舞う——VCO の周波数チューニングの実体である。

### 3. バイアスと整流 — ダイオード方程式 (bias and rectification: the diode equation)

外部電圧 V を p 側に正で印加(順バイアス)すると、内蔵電位の障壁が `V_bi − V` に下がり、障壁を越えられるキャリアがボルツマン因子 `e^{qV/kT}` で指数的に増える。逆バイアスでは障壁が上がり、流れは飽和値に張り付く。まとめたものが**ショックレーのダイオード方程式**である (Shockley 1949; Sze & Ng 2007, Ch. 2)。

```text
I = I_s·(e^{qV/kT} − 1)
順方向: 26 mV ごとに電流が e 倍 (60 mV/桁) — 事実上のスイッチ
逆方向: I ≈ −I_s (微小な飽和電流。ただし温度に指数依存 — 前回 §3 のリーク)
```

「0.7 V で急に流れ出す」という実務の記憶則は、この指数関数を線形軸で見たときの見かけであり、本当の物理は**全域で指数**である。60 mV/桁という急峻さは、わずかな電圧差で電流を4〜5桁変えられることを意味し、これがスイッチとしての半導体の優秀さの根源になる(次回の MOSFET のサブスレッショルド係数にそのまま再登場する)。

### 4. 降伏 — 逆方向の限界 (breakdown)

逆バイアスを深くすると、ある電圧で電流が急増する**降伏 (breakdown)** が起きる。機構は2つある (Sze & Ng 2007, Ch. 2)。

```text
なだれ降伏 (avalanche): 空乏層の強電場でキャリアが加速され、格子に衝突して
  電子·正孔対を連鎖生成。低濃度・厚い空乏層で支配的
ツェナー降伏 (Zener): 空乏層が極薄 (高濃度ドープ) になり、価電子帯から
  伝導帯へ量子トンネリングで直接遷移。次々回のトンネル効果の初登場
```

降伏は本質的に破壊ではない——電流を制限すれば可逆であり、**設計資源**として使われる: ツェナーダイオードは基準電圧源に、なだれ増倍はアバランシェフォトダイオード (APD) や SPAD(単一光子検出)の増幅機構に、そして ESD 保護回路は「静電気サージを保護ダイオードの降伏で安全に逃がす」構造であり、すべての IC の I/O パッドに標準装備されている。一方、制限なき降伏はジュール熱で接合を溶かす——パワーデバイスの絶対最大定格の物理的根拠である。

### 5. 光との相互作用 — フォトダイオード・太陽電池・LED (optoelectronic devices)

pn 接合は光と電気の変換器でもある。前回 §「工学への接続」の光吸収(光子エネルギー > Eg で電子·正孔対生成)に、本回の空乏層電場が組み合わさる。

```text
受光 (フォトダイオード / 太陽電池): 空乏層で光が対を生成 → 内蔵電場が
  電子と正孔を逆方向へ掃き分ける → 光電流。逆バイアスで動かせば高速な
  検出器 (フォトダイオード)、バイアスなしで電力を取り出せば太陽電池
発光 (LED / 半導体レーザー): 順バイアスで注入した電子と正孔が再結合し、
  Eg にほぼ等しいエネルギーの光子を放出。発光波長は材料の Eg が決める
  (GaN 系 ~2.8–3.4 eV: 青・紫外, InGaN: 緑, AlGaInP: 赤)
```

受光と発光は同じ接合物理の逆過程である。ただし Si は間接遷移型バンド構造のため発光効率が極めて低く、LED・レーザーには直接遷移型の化合物半導体 (GaAs, GaN, InP) が使われる——「なぜ照明とレーザーだけシリコンでないのか」の答である。イメージセンサの画素はフォトダイオード+読み出し回路であり、CMOS プロセスとの親和性(Si で作れる)が CCD から CMOS センサへの移行を支えた。

## 工学への接続 (Engineering Connections)

- **CMOS の中の「見えないダイオード」**: CMOS 構造は本質的に pn 接合の集合体であり、ウェル・基板・拡散層の間に寄生ダイオード・寄生サイリスタが必ず形成される。ラッチアップ(寄生サイリスタの点弧による電源短絡)対策と ESD 保護設計 (§4) は、「使っていない接合」の管理という形でレイアウト設計の必須項目になっている。
- **パワーエレクトロニクスの構成要素**: インバータ・DC-DC コンバータの還流経路はダイオード(または MOSFET のボディダイオード——これも寄生 pn 接合)が担う。順方向降下 ~0.7 V × 大電流 = 損失、という §3 の直接の帰結が、低損失化(同期整流、SiC ショットキー)の動機である。
- **接合容量と速度の限界**: 空乏層容量 (§2) はスイッチング時に充放電を要し、ダイオード・トランジスタの動作速度を制限する。「微細化 = 接合を小さく = 容量を減らす」がスケーリングの速度メリットの一部を説明する。
- **温度センサとしての接合**: ダイオード方程式の温度依存性 (V_F が約 −2 mV/°C) は、CPU のオンダイ温度センサ (thermal diode) の動作原理そのもの——サーマルスロットリングの測定端は pn 接合である。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/semiconductor-manufacturing/semiconductor-fabrication.md`](../../../hardware/semiconductor-manufacturing/semiconductor-fabrication.md) | 接合形成=イオン注入・拡散・アニールの目的物。ウェル構造と寄生接合の管理 |
| [`hardware/peripherals/camera-image-sensors.md`](../../../hardware/peripherals/camera-image-sensors.md) | 画素=フォトダイオード (§5)。空乏層の掃き分けが量子効率と速度を決める |
| [`hardware/peripherals/display-pipeline.md`](../../../hardware/peripherals/display-pipeline.md) | LED/OLED バックライト・自発光画素の発光原理 (§5)、Eg と色の関係 |
| [`engineering/automotive/ev/tesla-battery-powertrain.md`](../../../engineering/automotive/ev/tesla-battery-powertrain.md) | インバータの還流ダイオード・ボディダイオード、SiC 化による損失削減 (工学への接続 §2) |
| [`engineering/energy/battery-energy-storage.md`](../../../engineering/energy/battery-energy-storage.md) | 太陽電池 (§5) とパワーコンディショナの整流・変換段 |
| [`hardware/hw-sw-interface/clock-power-reset.md`](../../../hardware/hw-sw-interface/clock-power-reset.md) | サーマルダイオードによる温度測定 (工学への接続 §4)、ESD 保護と電源クランプ |
| [`hardware/security/side-channels-tee-secure-boot.md`](../../../hardware/security/side-channels-tee-secure-boot.md) | レーザー・電圧グリッチによるフォールト注入の物理的入口は接合の光感受性 (§5)・降伏 (§4) |

## 参考文献 (References)

- Sze, S. M., & Ng, K. K. (2007). *Physics of Semiconductor Devices* (3rd ed.). Wiley. DOI: 10.1002/0470068329 — 接合形成・ダイオード方程式・降伏の主参照
- Streetman, B. G., & Banerjee, S. K. (2015). *Solid State Electronic Devices* (7th ed.). Pearson. — キャリア輸送・バンド図の教育的主参照
- Shockley, W. (1949). The theory of p-n junctions in semiconductors and p-n junction transistors. *Bell System Technical Journal*, 28(3), 435–489. DOI: 10.1002/j.1538-7305.1949.tb03645.x — ダイオード方程式の原典
- Pierret, R. F. (1996). *Semiconductor Device Fundamentals*. Addison-Wesley. — 空乏近似の導出の副読本
