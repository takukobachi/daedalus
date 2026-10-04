# MOSFET と CMOS (The MOSFET and CMOS)

## 概要 (Overview)

量子力学・固体物理シリーズの第3回。前回の pn 接合は「電圧で制御できる非対称性」を与えたが、2端子素子には決定的な限界がある。**入力と出力が分離できない**ことである。計算には「小さな信号で大きな電流を制御し、しかも制御側に電流が流れない」3端子スイッチが要る。それが **MOSFET (metal-oxide-semiconductor field-effect transistor)** である: ゲート電極が絶縁膜越しの**電場だけ**で半導体表面のキャリア濃度を変調し(電界効果)、ソース–ドレイン間の導通を桁で切り替える。

MOSFET を n 型・p 型で対にした **CMOS (complementary MOS)** は、「どちらか一方しか導通しない」構造によって静的電流をほぼゼロにした。スイッチング時にしか電力を消費しないこの性質が、数十億トランジスタの集積を熱的に可能にし、デジタル文明の物理的基盤になった。本回は MOS 構造の物理(反転層としきい値電圧)、トランジスタの I-V 特性、CMOS 論理の動作、そして電力とスケーリングの法則(デナードスケーリングの黄金時代とその終焉、FinFET/GAA への構造進化)までを扱う。「なぜクロック周波数は 2000 年代半ばで頭打ちになったのか」という計算機アーキテクチャ最大の転換点の答が、本回の物理から出てくる。

## 前提知識 (Prerequisites)

- [`band-theory-semiconductors.md`](band-theory-semiconductors.md) — バンド、キャリア、ドーピング、フェルミ準位
- [`pn-junction-diode.md`](pn-junction-diode.md) — 空乏層、バンドの曲がり、60 mV/桁の指数則
- [`../electromagnetism/charge-field-potential.md`](../electromagnetism/charge-field-potential.md) — キャパシタンス(MOS 構造は本質的にキャパシタである)

## 本論 (Main Content)

### 1. MOS 構造 — 電場でキャリアを呼び出す (the MOS capacitor)

ゲート金属 / 酸化膜 (SiO₂) / p 型シリコンの3層構造は、絶縁膜を挟んだキャパシタである。ゲート電圧を変えると、絶縁膜越しの電場が半導体表面のバンドを曲げ(前回 §2 のバンドベンディングの電圧制御版)、表面の状態が3段階に変わる (Sze & Ng 2007, Ch. 4)。

```text
V_G < 0: 蓄積 (accumulation) — 多数キャリア(正孔)が表面に集まる
V_G > 0 (小): 空乏 (depletion) — 正孔が追い払われ、空乏層が伸びる
V_G > V_th: 反転 (inversion) — 少数キャリア(電子)が表面に呼び集められ、
  p 型基板の表面に薄い n 型の層 = 反転層 (inversion layer) ができる
```

**しきい値電圧 (threshold voltage) V_th** は反転層が形成される境目であり、基板ドーピング・酸化膜厚・仕事関数差で設計できる。反転層こそ MOSFET の心臓である。ゲート電圧という「場」だけで、電流を運べる導電チャネルを作ったり消したりできる。ゲートは絶縁膜で隔てられているため直流電流を流さない(理想的には入力電力ゼロ)。pn 接合(電流駆動)との決定的な違いがここにある。

### 2. MOSFET — 電圧制御スイッチ (the transistor)

MOS 構造の両脇に n⁺ のソースとドレインを置くと nMOS トランジスタになる。ゲートが反転層を作れば、ソースとドレインが n 型チャネルでつながる。

```text
遮断 (V_GS < V_th):  チャネルなし。ただしゼロではなく、サブスレッショルド電流
  I ∝ e^{qV_GS/nkT} が残る (n ≈ 1.2–1.5; 理論限界 60 mV/桁 — 前回 §3 と同じ
  ボルツマン因子)
線形領域 (V_DS 小):  チャネルは電圧制御抵抗。 I_D ∝ (V_GS − V_th)·V_DS
飽和領域 (V_DS 大):  ドレイン側でチャネルがピンチオフし、電流が飽和:
  I_D ≈ (μC_ox/2)(W/L)(V_GS − V_th)²  — ゲート電圧が電流を2乗則で決める
```

設計者が握るパラメータが式に現れている: W/L(トランジスタの幅と長さの比=レイアウトで決める駆動能力)、C_ox(酸化膜容量=プロセスが決める)、μ(移動度=材料とひずみ工学)。pMOS は全極性を反転した鏡像で、正孔の移動度が電子の約半分のため、同じ駆動力には約2倍の W を要する。これが、標準セルで pMOS が太い理由である (Taur & Ning 2009, Ch. 3–4)。

### 3. CMOS — 相補対という発明 (the complementary pair)

nMOS(入力 High で導通)と pMOS(入力 Low で導通)を積み重ねたインバータが CMOS 論理の原子である。

```text
入力 High: nMOS on / pMOS off → 出力は GND へ引かれる (Low)
入力 Low:  nMOS off / pMOS on → 出力は VDD へ引かれる (High)
どちらの定常状態でも、VDD から GND への直流経路が存在しない
```

**静的電力がほぼゼロ**である。この1点が、CMOS が他のあらゆる論理族(抵抗負荷、バイポーラ TTL/ECL、nMOS のみ)を駆逐した理由である。出力は常に VDD か GND へ低インピーダンスで駆動されるため信号レベルが再生され(ノイズ耐性)、ゲートは容量負荷なので論理を段数深く連ねられる。スイッチング速度は「pMOS/nMOS が負荷容量 C_L(次段のゲート容量+配線容量)を充放電する時間」で決まり、遅延 ∝ C_L·VDD/I_D となる。§2 の駆動電流の式が、そのままゲート遅延・最大クロック周波数の式になる (Weste & Harris 2011, Ch. 4)。

### 4. 電力の3成分とデナードスケーリング (power and Dennard scaling)

CMOS チップの消費電力は3成分に分解される。

```text
動的電力:      P_dyn = α·C·V²·f   (α: スイッチング率。充放電1回で CV² を消費)
貫通電流:      遷移の瞬間だけ nMOS と pMOS が同時に導通
リーク電力:    サブスレッショルド (e^{−qV_th/nkT}) + ゲート酸化膜トンネリング
```

デナードスケーリング (Dennard et al. 1974) は「寸法・電圧を 1/k に縮めれば、速度 k 倍・集積度 k² 倍・**電力密度一定**」という黄金律だった。これが 2000 年代半ばに破綻する。原因は V_th を下げられなくなったことにある。サブスレッショルドリークは V_th に対して指数的(60 mV/桁の物理限界はボルツマン統計そのもので、プロセス改良では破れない)であり、V_th を下げれば漏れ、下げなければ VDD も下げられず動的電力が落ちない。加えて酸化膜は数原子層まで薄くなり量子トンネリングで漏れ始めた(high-k 材料への移行で対処)。結果が**電力の壁**: 周波数を上げると V² の壁に当たるため、クロックは ~4–5 GHz で頭打ちになり、増え続けるトランジスタは「もう1つのコア」に振り向けられた。マルチコア化・ダークシリコン・DVFS(電圧周波数制御)という現代アーキテクチャの前提条件は、すべてこの物理に由来する (Taur & Ning 2009; Weste & Harris 2011, Ch. 5)。

### 5. 構造進化 — プレーナから FinFET・GAA へ (structural evolution)

微細化でチャネル長 L が短くなると、ドレインの電場がチャネル全体に染み出し、ゲートの支配力が落ちる。これが**短チャネル効果 (short-channel effects)** で、V_th の低下、サブスレッショルド勾配の劣化、ゲートで切れないリーク経路をもたらす。対策の歴史は「**ゲートがチャネルを囲む面を増やす**」一本道である。

```text
プレーナ (〜28 nm 世代): ゲートは上面の1面のみ
FinFET (〜16 nm 以降):   チャネルを立てたヒレ (fin) にし、3面から制御
GAA / ナノシート (〜3 nm 以降): チャネルを完全に取り囲む (gate-all-around)
```

静電的支配の回復が目的であり、「n nm」というノード名がもはや物理寸法でなくなった時代でも、この構造進化の論理は一貫している。チャネルを囲む幾何学の追求はやがて積層(CFET: nMOS と pMOS の縦積み)へ向かう。製造(リソグラフィ・エッチング・成膜)の各文書で起きている工程の複雑化は、この静電気学的要請の裏面である。

## 工学への接続 (Engineering Connections)

- **DVFS の物理的根拠**: 動的電力の V² 依存 (§4) により、周波数と電圧を連動して下げると電力は約3乗で落ちる。これが、OS のガバナ・ブースト制御・サーマルスロットリングの背後にある1本の式である。
- **SRAM/DRAM のセル設計**: SRAM の6トランジスタセルは CMOS インバータ2個の交差結合 (§3)、DRAM は1トランジスタ+1キャパシタでアクセストランジスタは §2 のスイッチである。メモリ階層の面積・速度・リークの特性差はセル構造の物理に還元される。
- **電力サイドチャネルの起源**: 動的電力がスイッチング率 α に比例する (§4) ということは、**消費電力がデータに依存する**ということである。これが電力解析攻撃 (DPA) が成立する物理的根拠であり、対策(マスキング、電力平坦化)はこの依存の切断を狙う。
- **アナログ/デジタルの共存**: 同じトランジスタが、デジタルでは遮断/飽和のスイッチ、アナログでは飽和領域の2乗則 (§2) を使う増幅器になる。SerDes のイコライザ(電磁気学第6回)や PLL のチャージポンプは、デジタルチップの中のアナログ MOSFET である。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/semiconductor-manufacturing/semiconductor-fabrication.md`](../../../hardware/semiconductor-manufacturing/semiconductor-fabrication.md) | ゲート形成・high-k/メタルゲート・FinFET/GAA 工程 (§1, §5) が作ろうとしている物理構造 |
| [`hardware/hw-sw-interface/clock-power-reset.md`](../../../hardware/hw-sw-interface/clock-power-reset.md) | 電力3成分と DVFS・クロックゲーティング (§4)、V_th とリークのトレードオフ |
| [`hardware/cpu/cache-memory-hierarchy.md`](../../../hardware/cpu/cache-memory-hierarchy.md) | SRAM セル=交差結合インバータ (§3)。キャッシュ容量とリークの関係 |
| [`hardware/cpu/pipeline-superscalar-ooo.md`](../../../hardware/cpu/pipeline-superscalar-ooo.md) | 周波数の頭打ち (§4) が ILP 追求からマルチコアへの転換を強制した物理的背景 |
| [`hardware/cpu/dram-memory-controllers.md`](../../../hardware/cpu/dram-memory-controllers.md) | DRAM セルのアクセストランジスタとリーク→リフレッシュ (§2, §4) |
| [`hardware/security/side-channels-tee-secure-boot.md`](../../../hardware/security/side-channels-tee-secure-boot.md) | 電力解析攻撃の物理的起源 = α·C·V²·f のデータ依存性 (工学への接続 §3) |
| [`hardware/accelerators/gpu-simt-architecture.md`](../../../hardware/accelerators/gpu-simt-architecture.md) | ダークシリコン時代の設計選択——電力バジェット内で演算器を並べる (§4) |

## 参考文献 (References)

- Sze, S. M., & Ng, K. K. (2007). *Physics of Semiconductor Devices* (3rd ed.). Wiley. DOI: 10.1002/0470068329 — MOS 構造・MOSFET 特性の主参照
- Taur, Y., & Ning, T. H. (2009). *Fundamentals of Modern VLSI Devices* (2nd ed.). Cambridge University Press. — 短チャネル効果・スケーリング理論の主参照
- Weste, N. H. E., & Harris, D. M. (2011). *CMOS VLSI Design* (4th ed.). Addison-Wesley. — CMOS 論理・電力・回路設計側の主参照
- Dennard, R. H., Gaensslen, F. H., Yu, H.-N., Rideout, V. L., Bassous, E., & LeBlanc, A. R. (1974). Design of ion-implanted MOSFET's with very small physical dimensions. *IEEE Journal of Solid-State Circuits*, 9(5), 256–268. DOI: 10.1109/JSSC.1974.1050511 — スケーリング則の原典
