# 統計力学と熱雑音 (Statistical Mechanics and Thermal Noise)

## 概要 (Overview)

熱力学・統計力学シリーズの第2回。第1回ではエントロピーを S = k ln W と書き、熱力学の法則を「場合の数の多い状態へ確率的に向かう」という統計的な事実として読んだ。本回はこの見方を計算できる道具にする。中心にあるのは、温度 T の環境と熱をやりとりする系が、エネルギー E の状態を確率 e^{−E/kT} に比例して占めるという**ボルツマン分布 (Boltzmann distribution)** である。その規格化定数である**分配関数 (partition function)** Z から、平均エネルギー・揺らぎ・自由エネルギーといった熱力学量がすべて微分で出てくる。

工学の側では、この一つの分布が三つの限界を与える。第一は半導体のキャリア濃度と電流が e^{−E/kT} に支配されることである。熱電圧 kT/q ≈ 25.9 mV(300 K)はここから来ており、MOSFET のサブスレッショルド傾斜 60 mV/dec の下限もこの分布の帰結である。第二は**熱雑音 (thermal noise)** である。抵抗の両端には電力を加えなくても √(4kTRΔf) の雑音電圧が現れ(ジョンソン=ナイキスト雑音)、標本化容量には √(kT/C) の雑音が残る。ADC の分解能と電力の交換関係はこの雑音で決まる。第三は**ランダウアー限界 (Landauer limit)** である。1 ビットの情報を消去するには最低 kT ln 2 の熱を捨てなければならない。これは計算そのものに課される熱力学的な下限であり、2012 年に実験で確かめられた。本回はこの三つを、等分配則と揺動散逸定理という二つの定理で結びつける。

## 前提知識 (Prerequisites)

- [`laws-entropy-heat-engines.md`](laws-entropy-heat-engines.md) — 温度・第二法則・S = k ln W。本回はその統計的解釈 (§3) を一般化する
- [`../../mathematics/probability-statistics/probability-random-variables.md`](../../mathematics/probability-statistics/probability-random-variables.md) — 確率分布、期待値と分散(揺らぎの計算で使う)
- [`../../information-theory/entropy-source-coding.md`](../../information-theory/entropy-source-coding.md) — シャノンエントロピー。ギブスエントロピーと同形であることを §5 で使う
- [`../electromagnetism/circuit-theory.md`](../electromagnetism/circuit-theory.md) — RC 回路と帯域(熱雑音の帯域制限で使う)

## 本論 (Main Content)

### 1. ボルツマン分布と分配関数 (Boltzmann distribution and the partition function)

孤立系ではすべてのミクロ状態が等確率で現れる(等重率の原理)。注目する小さな系を温度 T の大きな**熱浴 (heat bath)** に接触させると、系がエネルギー E_i の状態 i を占める確率は、熱浴側に残るエネルギーで取れるミクロ状態の数に比例する。熱浴のエントロピーを E_i について1次まで展開し 1/T = ∂S/∂U を使うと、次の形が出る (Kittel & Kroemer 1980, Ch. 3)。

```text
P_i = e^{−E_i/kT} / Z,   Z = Σ_i e^{−E_i/kT}   (分配関数)
  k = 1.380649 × 10⁻²³ J/K (2019 年 SI 改定で定義値)
  300 K で kT ≈ 4.14 × 10⁻²¹ J ≈ 25.9 meV
熱力学量はすべて Z から:
  平均エネルギー  ⟨E⟩ = −∂ ln Z / ∂β        (β = 1/kT)
  エネルギー揺らぎ ⟨ΔE²⟩ = ∂² ln Z / ∂β² = kT² C_V
  ヘルムホルツ自由エネルギー F = −kT ln Z,  S = −∂F/∂T
```

要点は二つある。第一に、確率はエネルギーの**指数関数**で減る。kT の何倍の障壁かが確率を桁で決めるため、「kT に比べて大きいか小さいか」が物理系を読む最初の問いになる。第二に、Z は状態の一覧から作る和にすぎないのに、熱力学のあらゆる量がその対数の微分で求まる。統計力学の仕事の大半は Z を計算することだと言ってよい (Reif 1965, Ch. 6)。

自由エネルギー F = U − TS は、温度一定の環境で系が取り出せる仕事の上限である。系は F を最小にする状態へ向かい、エネルギーを下げる傾向 (U) とエントロピーを稼ぐ傾向 (TS) の折り合いを温度が決める。高温ほどエントロピーが勝つので、秩序だった状態(磁化、結晶)は温度を上げると崩れる。

### 2. 量子統計: フェルミ=ディラックとボース=アインシュタイン (quantum statistics)

同種粒子が区別できないことを取り入れると、1粒子状態の占有数の平均は粒子の種類で分かれる (Kittel & Kroemer 1980, Ch. 6)。

```text
フェルミ粒子 (電子): f(E) = 1 / (e^{(E−μ)/kT} + 1)   — フェルミ=ディラック分布
ボース粒子 (光子・フォノン): n(E) = 1 / (e^{(E−μ)/kT} − 1)  — ボース=アインシュタイン分布
E − μ ≫ kT では両者とも e^{−(E−μ)/kT} (ボルツマン近似) に一致する
```

μ は**化学ポテンシャル (chemical potential)** で、半導体ではフェルミ準位と呼ばれる。シリコンの伝導帯の電子濃度が n ≈ N_c e^{−(E_c−E_F)/kT} と書けるのは、伝導帯の底がフェルミ準位から数 kT 以上離れていてボルツマン近似が使えるからである。真性キャリア濃度が e^{−E_g/2kT} で決まり温度に極めて敏感なこと、pn 接合の電流が e^{qV/kT} で立ち上がること、MOSFET のサブスレッショルド電流が1桁変わるのにゲート電圧が少なくとも (kT/q) ln 10 ≈ 60 mV 要ることは、どれもこの指数の帰結である (Sze & Ng 2007, Ch. 1)。ボース側の帰結としてはプランクの黒体放射則(第1回 §5 のシュテファン=ボルツマン則の出どころ)と、固体の熱容量を担うフォノンがある。

### 3. 等分配則と kT/C 雑音 (equipartition and kT/C noise)

古典的な系で、エネルギーが座標または運動量の2乗で書ける自由度は、熱平衡でそれぞれ平均 kT/2 のエネルギーをもつ。これが**等分配則 (equipartition theorem)** で、ボルツマン分布のガウス積分から直接出る (Reif 1965, Ch. 7)。

```text
E = ½ a x² の自由度 → ⟨½ a x²⟩ = ½ kT
容量 C の蓄積エネルギー ½ C V² に適用:  ⟨½ C V²⟩ = ½ kT
⟹ V_rms = √(kT/C)
  例: C = 1 pF, T = 300 K → V_rms ≈ 64 µV
```

この結果は抵抗値によらない。スイッチの抵抗 R は雑音の源だが、R が大きいほど雑音密度は増え、同時に RC 帯域は狭まり、両者が打ち消し合う。標本化回路の雑音を下げる手段は C を大きくすることしかなく、雑音を半分(+6 dB、1 bit 相当)にするには C を 4 倍にし、同じ速度で充電するために電流も 4 倍にする必要がある。ADC の高分解能域で「1 bit ごとに電力 4 倍」が成り立つ理由はこの1行にある。

### 4. ジョンソン=ナイキスト雑音と揺動散逸定理 (Johnson–Nyquist noise and fluctuation–dissipation)

1928 年、Johnson は温度 T の抵抗が、材料によらず T と R に比例する雑音電圧を出すことを測定で示した。同じ号で Nyquist は2つの抵抗を伝送線路でつないだ系に熱平衡と等分配則を当てはめ、その大きさを導いた (Johnson 1928; Nyquist 1928)。

```text
開放端雑音電圧のパワースペクトル密度 (片側, hf ≪ kT):
  S_V(f) = 4kTR  [V²/Hz]     ⟹ V_n,rms = √(4kTRΔf)
  例: R = 1 kΩ, T = 300 K → 4.07 nV/√Hz;  R = 50 Ω → 0.91 nV/√Hz
整合負荷へ渡せる雑音電力: P_n = kTΔf  — −174 dBm/Hz (290 K) の出どころ
抵抗 R と容量 C の並列で全帯域を積分すると ∫4kTR/(1+(2πfRC)²) df = kT/C  (§3 と一致)
```

雑音電力 kTΔf は無線受信機の雑音指数と感度計算の基準になる。熱雑音は白色(周波数によらない)でガウス分布に従うので、シャノンの通信路容量で仮定する AWGN の物理的な実体でもある。

Nyquist の結果はより一般的な**揺動散逸定理 (fluctuation–dissipation theorem)** の最初の例である (Callen & Welton 1951)。エネルギーを散逸させる機構(抵抗、粘性、摩擦)は、熱平衡では必ず揺らぎの源にもなり、両者の大きさは kT で結ばれる。抵抗は電流を熱に変えるのと同じ微視的な衝突によって雑音を出す。逆に、散逸のない理想的なリアクタンス (L, C) は雑音を生まない。したがって低雑音設計とは、信号経路に置く損失を減らすことである。冷却 (T を下げる) も効くため、電波望遠鏡や量子計算機の初段増幅器は極低温に置かれる。

### 5. ランダウアー限界: 情報消去の熱力学的コスト (Landauer's limit)

ギブスエントロピー S = −k Σ p_i ln p_i は、シャノンエントロピー H = −Σ p_i log₂ p_i に k ln 2 を掛けたものと同じ形をしている。1 ビットのメモリ(0 と 1 が等確率)を確定した状態 0 に**消去 (erase)** する操作は、メモリのエントロピーを k ln 2 だけ減らす。第二法則は全体のエントロピーが減ることを許さないので、その分を熱として環境に捨てなければならない (Landauer 1961)。

```text
1 ビット消去の最小散逸熱:  Q ≥ kT ln 2
  300 K で ≈ 2.87 × 10⁻²¹ J (≈ 0.018 eV)
比較 (本稿の試算): 1 回の論理遷移に 1 fJ = 10⁻¹⁵ J を要するとすれば、限界の約 3.5 × 10⁵ 倍
```

この限界は論理演算一般ではなく、**情報を捨てる操作**にだけ課される。AND ゲートは2入力から1出力を作るので、入力の情報を失い、原理的に散逸を伴う。一方、入力を出力から復元できる**可逆計算 (reversible computing)** は、原理的には散逸ゼロに近づけられる (Bennett 1982)。マクスウェルの悪魔が第二法則を破れない理由も、悪魔が測定結果を記録したメモリをいつか消去しなければならず、そのコストが得た仕事を打ち消すことで説明される。

ランダウアー限界は長く思考実験の対象だったが、Bérut ら (2012) は光ピンセットで捕捉したコロイド粒子を二重井戸ポテンシャル中の1ビットメモリとして使い、消去を十分ゆっくり行うと平均散逸熱が kT ln 2 に漸近することを測定した。現在の CMOS は限界から5桁以上離れており、エネルギーの大部分は配線容量の充放電に使われている。したがってこの限界が当面の設計を縛るわけではない。それでも、「計算はなぜ熱を出すのか」に対する、装置によらない最終的な答えはこの式である。

## 工学への接続 (Engineering Connections)

- **熱電圧 kT/q がデバイスの尺度を決める**: ダイオード電流の指数、バイポーラトランジスタの gm = I_C/(kT/q)、MOSFET のサブスレッショルド傾斜 60 mV/dec(300 K)の下限は、どれもボルツマン因子から来る (§1–2)。電源電圧を下げてもしきい値を比例して下げられない(リーク電流が増える)という CMOS 微細化の壁は、この 60 mV/dec が温度だけで決まり、構造では縮まないことの帰結である。
- **ADC の電力と分解能の交換は kT/C が決める**: 標本化容量を 4 倍にするごとに雑音が半分になり、電力も 4 倍になる (§3)。高分解能 ADC の性能指数の上限が kT で書かれる理由である。
- **受信機の感度は kTB から始まる**: −174 dBm/Hz に帯域と雑音指数を足したものが受信感度の床になる (§4)。低雑音増幅器の設計は初段の損失を減らし利得を確保するという、揺動散逸定理を実務に置き換えた作業である。
- **計算のエネルギーには下限がある**: kT ln 2 はビット消去の物理的下限で (§5)、データセンターの電力問題を最終的に制約する。ただし現状の支配項は配線とデータ移動であり、限界との差は5桁以上ある。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/mixed-signal/adc-dac-converters.md`](../../../hardware/mixed-signal/adc-dac-converters.md) | 標本化容量の kT/C 雑音と「1 bit ごとに電力 4 倍」(§3)、8kT·SNR のエネルギー限界の出どころ |
| [`foundations/physics/quantum-solid-state/band-theory-semiconductors.md`](../quantum-solid-state/band-theory-semiconductors.md) | フェルミ=ディラック分布とキャリア濃度 n ∝ e^{−(E_c−E_F)/kT} (§2) |
| [`foundations/physics/quantum-solid-state/pn-junction-diode.md`](../quantum-solid-state/pn-junction-diode.md) | ダイオード電流の e^{qV/kT} と熱電圧 (§2) |
| [`foundations/physics/quantum-solid-state/mosfet-cmos.md`](../quantum-solid-state/mosfet-cmos.md) | サブスレッショルド傾斜 (kT/q) ln 10 ≈ 60 mV/dec とリーク電流 (§2) |
| [`foundations/information-theory/channel-capacity-coding.md`](../../information-theory/channel-capacity-coding.md) | AWGN 通信路の雑音 = 熱雑音 kTB (§4)、情報エントロピーと熱力学エントロピーの対応 (§5) |
| [`foundations/physics/electromagnetism/transmission-lines-signal-integrity.md`](../electromagnetism/transmission-lines-signal-integrity.md) | Nyquist の導出が使う整合伝送線路 (§4)、50 Ω 系の雑音床 |
| [`engineering/datacenters/datacenter-power.md`](../../../engineering/datacenters/datacenter-power.md) | 計算のエネルギーの物理的下限 kT ln 2 と、現状の電力との桁差 (§5) |

## 参考文献 (References)

- Kittel, C., & Kroemer, H. (1980). *Thermal Physics* (2nd ed.). W. H. Freeman. — ボルツマン分布・分配関数 (Ch. 3)、量子統計 (Ch. 6)。§1–2 の主参照
- Reif, F. (1965). *Fundamentals of Statistical and Thermal Physics*. McGraw-Hill. — 分配関数から熱力学量を導く手順 (Ch. 6)、等分配則 (Ch. 7)、ジョンソン雑音
- Sze, S. M., & Ng, K. K. (2007). *Physics of Semiconductor Devices* (3rd ed.). Wiley. — 半導体のキャリア統計 (Ch. 1)。§2 の工学側参照
- Johnson, J. B. (1928). Thermal Agitation of Electricity in Conductors. *Physical Review*, 32(1), 97–109. DOI: 10.1103/PhysRev.32.97
- Nyquist, H. (1928). Thermal Agitation of Electric Charge in Conductors. *Physical Review*, 32(1), 110–113. DOI: 10.1103/PhysRev.32.110
- Callen, H. B., & Welton, T. A. (1951). Irreversibility and Generalized Noise. *Physical Review*, 83(1), 34–40. DOI: 10.1103/PhysRev.83.34
- Landauer, R. (1961). Irreversibility and Heat Generation in the Computing Process. *IBM Journal of Research and Development*, 5(3), 183–191. DOI: 10.1147/rd.53.0183
- Bennett, C. H. (1982). The Thermodynamics of Computation — a Review. *International Journal of Theoretical Physics*, 21(12), 905–940. DOI: 10.1007/BF02084158
- Bérut, A., Arakelyan, A., Petrosyan, A., Ciliberto, S., Dillenschneider, R., & Lutz, E. (2012). Experimental verification of Landauer's principle linking information and thermodynamics. *Nature*, 483, 187–189. DOI: 10.1038/nature10872
- [NIST — Kelvin: Boltzmann Constant](https://www.nist.gov/si-redefinition/kelvin-boltzmann-constant) — Accessed: 2026-11-28. k = 1.380649 × 10⁻²³ J/K の定義値
