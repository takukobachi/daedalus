# マクスウェル方程式と電磁波 (Maxwell's Equations and Electromagnetic Waves)

## 概要 (Overview)

電磁気学シリーズの第4回。ここまでに登場したガウスの法則(第1回)、アンペールの法則(第2回)、ファラデーの法則(第3回)を1組の方程式系に統合する。鍵はマクスウェルによる**変位電流 (displacement current)** の追加である。この1項がアンペールの法則の論理的欠陥を塞ぎ、その帰結として「電場と磁場が互いを生み合いながら空間を伝わる」解、つまり**電磁波 (electromagnetic wave)** が方程式から出てくる。伝播速度は測定可能な2定数から

```text
c = 1/√(μ₀ε₀) ≈ 3.0×10⁸ m/s
```

と計算され、これが光速と一致したことが「光は電磁波である」という統一の根拠になった。WiFi の 2.4 GHz も、光ファイバの赤外光も、カメラが捉える可視光も、EUV リソグラフィの 13.5 nm も、すべて同じ4本の方程式の解であり、違いは周波数だけである。本回はこの統合と、波の基本性質(横波・偏波・スペクトル・エネルギー輸送)までを扱う。

## 前提知識 (Prerequisites)

- [`charge-field-potential.md`](charge-field-potential.md) — 電場、ガウスの法則
- [`current-magnetostatics.md`](current-magnetostatics.md) — 電流、アンペールの法則
- [`induction-inductance.md`](induction-inductance.md) — ファラデーの法則(変化する磁場が電場を作る)
- [`../../mathematics/calculus-analysis/limits-derivatives-integrals.md`](../../mathematics/calculus-analysis/limits-derivatives-integrals.md) — 微分・積分の言葉

## 本論 (Main Content)

### 1. 変位電流 — アンペールの法則の修正 (displacement current)

第2回のアンペールの法則「磁場の周回積分 = 貫く電流 × μ₀」は、定常電流では正しいが、**充電中のキャパシタ**で破綻する。極板間には電流が流れていないのに、周回路の張り方によって「貫く電流」の答が変わってしまう。つまり法則が曖昧になる。

マクスウェルの洞察は、極板間で電流の代わりに**電場が変化している**ことに注目し、電場の時間変化を電流と同格に扱うことだった (Griffiths 2017, §7.3)。

```text
アンペール–マクスウェルの法則:
∮B·dl = μ₀(I + ε₀·dΦ_E/dt)
                └─ 変位電流: 電場変化が磁場を作る
```

これで法則は無矛盾になり、電荷保存則とも整合する。物理的な意味は対称性の完成である: ファラデーの法則が「変化する磁場 → 電場」なら、変位電流は「変化する電場 → 磁場」である。こうして**相互生成のループ**が閉じ、場が電荷や電線から独立して自走できるようになる。

### 2. マクスウェル方程式 (Maxwell's equations)

真空中の4本(積分形)を一覧する (Griffiths 2017, §7.3.3)。

```text
(1) ガウスの法則        ∮E·dA = Q/ε₀          電荷が電場の湧き出し口
(2) 磁場のガウスの法則   ∮B·dA = 0             磁気単極子は存在しない
(3) ファラデーの法則     ∮E·dl = −dΦ_B/dt      変化する磁場が電場を生む
(4) アンペール–マクスウェル ∮B·dl = μ₀I + μ₀ε₀·dΦ_E/dt   電流と変化する電場が磁場を生む
```

これにローレンツ力 `F = q(E + v×B)`(第2回)を加えれば、古典電磁気学は**完結する**。静電気からモーター、無線、光まで、このページの式の外に法則はない。微分形(∇·E = ρ/ε₀ など)は同じ内容の局所的な書き換えであり、多変数の微積分(解析シリーズ第3回のロードマップ)を待って導入する。

### 3. 電磁波の導出 (deriving the electromagnetic wave)

電荷も電流もない真空 (Q = 0, I = 0) を考える。(3) と (4) だけが残り、「変化する E が B を作り、その変化する B が E を作り…」という連鎖が起きる。2式を組み合わせると、E と B のそれぞれが**波動方程式 (wave equation)** を満たすことが示せる。

```text
∂²E/∂x² = μ₀ε₀ · ∂²E/∂t²    →  速度 v = 1/√(μ₀ε₀) で伝わる波
```

μ₀(アンペール力の強さ)と ε₀(クーロン力の強さ)は、どちらも**電磁波と無関係に机上の実験で測れる**定数である。その組み合わせが光速を言い当てたことは、理論が「知らなかったこと」を予言した物理学史上の決定的瞬間であり、ヘルツによる電波の発生・検出 (1887) が実験的裏付けを与えた (Purcell & Morin 2013, Ch. 9)。

### 4. 波の性質 — 横波・偏波・スペクトル (transverse waves, polarization, spectrum)

方程式は波の構造も指定する。電磁波は**横波 (transverse wave)** であり、E と B は互いに直交し、かつ進行方向とも直交する。E⊥B⊥k の右手系で、振幅比は常に `E = cB`。E の振動方向を**偏波 (polarization)** と呼び、直線偏波・円偏波が基本形になる。アンテナの向きが通信品質に影響する理由や、偏光フィルタ・液晶ディスプレイの動作原理はここにある。

波は周波数 f と波長 λ で特徴づけられ、真空中では `c = fλ`。**スペクトル (spectrum)** の全域が同一の物理に従う。

| 帯域 | 周波数 / 波長 | Daedalus での出現場所 |
|------|-------------|---------------------|
| 電波 (radio) | ~kHz–GHz | WiFi 2.4/5/6 GHz、Bluetooth、GNSS |
| マイクロ波 | 1–300 GHz | 5G ミリ波、原子時計の遷移 (Cs: 9.19 GHz) |
| 赤外 | ~THz | 光ファイバ通信 (1310/1550 nm)、リモコン |
| 可視光 | 430–770 THz | カメラ、ディスプレイ |
| 紫外–X線 | >PHz | EUV リソグラフィ (13.5 nm) |

周波数はまた情報伝送の資源でもある。搬送波周波数が高いほど広い変調帯域が取れる(WiFi が 2.4 GHz から 6 GHz へ、無線バックホールがミリ波へ向かう根本理由)。一方で高周波ほど直進性が強く回折しにくく、障害物・大気減衰に弱い。このように、到達性と容量のトレードオフがスペクトルの物理から直接出てくる。

### 5. エネルギー輸送とアンテナ (energy transport and antennas)

電磁波はエネルギーを運ぶ。単位面積・単位時間あたりのエネルギー流は**ポインティング・ベクトル (Poynting vector)**

```text
S = (1/μ₀) E×B    [W/m²]   (大きさの時間平均 = 放射強度)
```

で与えられる (Griffiths 2017, §8.1)。点源からの放射は距離 r の球面に広がるため、強度は **1/r² で減衰**する。これは、受信電力が距離の2乗(実環境ではそれ以上)で落ちるという、無線リンク設計の出発点(自由空間伝搬損失)である。

放射の源は**加速する電荷**である。静止電荷は電場のみ、等速の電荷は電流として磁場も作るが、どちらも波を放たない。電荷を振動させて初めて放射が起きる。アンテナとは「電荷を効率よく加速し続ける導体構造」であり、効率よく共振する寸法が波長で決まる(半波長ダイポールで λ/2。2.4 GHz なら約 6 cm で、WiFi アンテナの実寸に当たる)。同じ物理が意図しない場所で起きるのが **EMI (電磁干渉)** である: 高速信号の立ち上がり(大きな dV/dt, dI/dt)を持つ配線・スリットは意図せぬアンテナとして放射する。

## 工学への接続 (Engineering Connections)

- **無線通信の物理層の全部**: 搬送波・変調・偏波・アンテナ寸法・伝搬損失・帯域と周波数の関係など、WiFi/Bluetooth/GNSS/5G の物理層仕様書に出てくる概念は、すべて本回の方程式の帰結である。
- **「信号は場として伝わる」**: 配線上の信号は電子の移動(ドリフト速度は mm/s 級、第2回)ではなく、導体周りの**電磁場**として媒質中の光速(FR-4 基板で約 c/2)で伝わる。信号伝播遅延・クロック分配・伝送線路(次回)の理解はこの描像の転換から始まる。
- **EMC 設計**: 放射は加速電荷から (§5)。エッジレートの管理、リターンパスの確保、シールドは「意図せぬアンテナを作らない・閉じ込める」ための対策であり、規制 (FCC/CISPR) 適合の物理的根拠。
- **周波数選択というアーキテクチャ判断**: 到達性(低周波)と容量(高周波)のトレードオフ (§4) は、無線規格の周波数割り当てからチップ間光インターコネクトの波長選択まで、通信システム設計の第一の分岐点になる。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/peripherals/wifi-architecture.md`](../../../hardware/peripherals/wifi-architecture.md) | 2.4/5/6 GHz の伝搬特性、OFDM の搬送波、アンテナ・MIMO の物理的基盤 |
| [`hardware/peripherals/bluetooth-pairing-protocol.md`](../../../hardware/peripherals/bluetooth-pairing-protocol.md) | 2.4 GHz ISM 帯の共有、到達距離と送信電力 (1/r² 減衰) |
| [`hardware/hw-sw-interface/bus-interconnect.md`](../../../hardware/hw-sw-interface/bus-interconnect.md) | 信号は場として伝わる——伝播遅延・インピーダンス(次回)の前提となる描像 |
| [`hardware/peripherals/camera-image-sensors.md`](../../../hardware/peripherals/camera-image-sensors.md) | 可視光 = 数百 THz の電磁波。波長がセンサ画素・カラーフィルタ設計を規定 |
| [`hardware/peripherals/display-pipeline.md`](../../../hardware/peripherals/display-pipeline.md) | 液晶の動作原理は偏波 (§4) の制御 |
| [`hardware/timekeeping/clocks-time-synchronization.md`](../../../hardware/timekeeping/clocks-time-synchronization.md) | Cs 原子時計のマイクロ波遷移 (9.19 GHz)、GNSS 信号の光速伝播と測位 |
| [`hardware/semiconductor-manufacturing/semiconductor-fabrication.md`](../../../hardware/semiconductor-manufacturing/semiconductor-fabrication.md) | リソグラフィの解像度は波長で律速——EUV 13.5 nm への移行の物理的必然 |

## 参考文献 (References)

- Griffiths, D. J. (2017). *Introduction to Electrodynamics* (4th ed.). Cambridge University Press. DOI: 10.1017/9781108333511 — 変位電流・波動方程式・ポインティングベクトルの主参照
- Purcell, E. M., & Morin, D. J. (2013). *Electricity and Magnetism* (3rd ed.). Cambridge University Press. — 電磁波の導出と物理的描像の主参照
- Feynman, R. P., Leighton, R. B., & Sands, M. (1964). *The Feynman Lectures on Physics, Vol. II*. Addison-Wesley. — 場の描像・放射の直観の副読本(https://www.feynmanlectures.caltech.edu/ で公開)
- Pozar, D. M. (2011). *Microwave Engineering* (4th ed.). Wiley. — スペクトル利用・アンテナ・伝搬の工学側参照
