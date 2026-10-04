# 回路理論 (Circuit Theory)

## 概要 (Overview)

電磁気学シリーズの第5回。ここまでの4回で電磁場の完全な法則(マクスウェル方程式)が揃ったが、場の方程式を解いてコンピュータの基板を設計する者はいない。実務は**回路理論 (circuit theory)**(場の理論を「素子と接続」の代数に圧縮した近似体系)で回っている。本回の主題はこの圧縮がいつ・なぜ成立するかである。

鍵は**集中定数近似 (lumped-element approximation)**: 回路の物理寸法が信号の波長より十分小さければ、場の空間分布を無視して、エネルギーの蓄積・散逸を R・L・C という「点」に集中させてよい。このとき電荷保存則はキルヒホッフの電流則に、ファラデーの法則は電圧則に退化し、微分方程式(解析シリーズ第2回)と複素数の代数(インピーダンス)だけで回路が解ける。逆に言えば、**周波数が上がり波長が寸法に近づくと近似は成り立たなくなり**、次回の伝送線路(分布定数の世界)が必要になる。本回はその境界線まで、直流解析・テブナン等価・交流のフェーザとインピーダンス・共振と Q 値を積み上げる。

## 前提知識 (Prerequisites)

- [`charge-field-potential.md`](charge-field-potential.md) — 電位、キャパシタンス
- [`current-magnetostatics.md`](current-magnetostatics.md) — 電流、抵抗、ジュール熱
- [`induction-inductance.md`](induction-inductance.md) — インダクタンス、ファラデーの法則
- [`maxwell-equations-em-waves.md`](maxwell-equations-em-waves.md) — 波長 λ = c/f(近似の適用条件で使う)
- [`../../mathematics/calculus-analysis/ordinary-differential-equations.md`](../../mathematics/calculus-analysis/ordinary-differential-equations.md) — 1階・2階線形 ODE、減衰比

## 本論 (Main Content)

### 1. 集中定数近似 (the lumped-element approximation)

場の理論から回路理論への圧縮は、3つの理想化からなる (Agarwal & Lang 2005, Ch. 1)。

```text
(1) 回路寸法 ℓ ≪ 波長 λ = v/f — 信号変化が回路全体に「瞬時に」行き渡る
(2) 素子の外に電磁エネルギーを蓄えない — E 場は C の中、B 場は L の中に閉じ込める
(3) 配線は理想導体 — 抵抗・遅延ゼロ

目安: ℓ > λ/10 になったら集中定数は疑わしい
  50 Hz 送電 (λ = 6000 km) → 大陸規模でも回路理論が有効
  3 GHz 基板上 (λ ≈ 5 cm, 基板中) → 数 mm の配線でもう怪しい
```

同じ「電気の問題」でも、電力系統が回路理論で設計でき、高速シリアルリンクが伝送線路を要求するのは、**周波数と寸法の比**が違うからである。この1つの無次元量が、以降のシリーズの分水嶺になる。

### 2. キルヒホッフの法則 (Kirchhoff's laws)

集中定数近似の下で、マクスウェル方程式は2つの代数則に退化する。

```text
KCL (電流則): 各ノードで Σ流入電流 = 0   ← 電荷保存 (ノードに電荷が溜まらない)
KVL (電圧則): 各閉路で Σ電圧降下 = 0     ← ファラデーの法則で dΦ_B/dt ≈ 0 の場合
```

KVL の由来には注意する価値がある。閉路を貫く磁束が変化していれば `∮E·dl = −dΦ_B/dt ≠ 0` であり、**KVL は厳密には成り立たない**(第3回の誘導起電力はまさにこの項である)。実務で「グラウンドループが拾うノイズ」「大電流ループの di/dt が誘起する電圧」として現れる現象は、KVL の破れ=集中定数近似 (2) の破れとして統一的に理解できる。ループ面積の最小化という EMC 設計の鉄則は、この破れ項 dΦ_B/dt を小さくする操作に他ならない。

### 3. 直流解析と等価回路 (DC analysis and equivalent circuits)

KCL/KVL とオームの法則から、直流回路は連立一次方程式に帰着する(回路シミュレータの節点解析は文字通り `Ax = b` であり、線形代数第4回の疎 LU の応用先である)。手計算の道具として最重要なのは**テブナンの定理 (Thévenin's theorem)** である。

```text
線形回路の任意の2端子は、電圧源 V_th + 直列抵抗 R_th に等価
(双対: ノートン等価 = 電流源 + 並列抵抗)
```

「複雑なものを、外から見た振る舞いが同じ単純なものに置き換える」という、インターフェースの背後を抽象化するこの操作は、工学全体の設計思想の原型である。実用上は**出力インピーダンス**という概念そのものであり、「電池の内部抵抗」「レギュレータの出力インピーダンス」「ドライバの駆動能力」はすべて R_th の言い換えである。負荷への最大電力伝送が `R_load = R_th` で起きること(最大電力伝送定理)は、次回のインピーダンス整合の直流版にあたる。

### 4. 交流回路 — フェーザとインピーダンス (AC: phasors and impedance)

線形回路に正弦波 `cos(ωt)` を入れると、定常状態では**同じ周波数の正弦波**が振幅と位相だけ変わって出てくる(線形性の帰結)。ならば振幅と位相の組=複素数1個で信号を表せる。これが**フェーザ (phasor)** 表示で、オイラーの公式 `e^{jωt}`(解析第1回 §5)により微分が `×jω` に置き換わる。

```text
インピーダンス Z = V/I (複素数):
  抵抗:     Z_R = R          (位相そのまま、エネルギーを散逸)
  インダクタ: Z_L = jωL       (電流が 90° 遅れ。高周波で「重い」)
  キャパシタ: Z_C = 1/(jωC)   (電流が 90° 進み。高周波で「軽い」)
```

微分方程式が**複素数のオームの法則**に退化し、直列・並列・テブナンなど直流の道具がすべてそのまま使える。これがフェーザ法の威力である。周波数を変数と見ると、分圧回路はそのまま**フィルタ**になる: RC ローパス(カットオフ `f_c = 1/2πRC`)、ハイパス、そして L と C を組めば §5 の共振へ。「回路は周波数ごとに違う顔を持つ」という周波数領域の視点は、フーリエ解析(数学シリーズの候補)で一般の信号に拡張される。

交流電力はフェーザの内積として `P = V·I·cos φ`(cos φ: 力率, power factor)と書け、位相差 φ を持つ成分は仕事をしない**無効電力 (reactive power)** として往復する。送電網が無効電力を管理し力率改善を求めるのは、この往復分も導体を流れてジュール損を生むからである。こうして第2回のジュール熱と本節が電力系統でつながる。

### 5. 共振と Q 値 (resonance and Q factor)

L と C を組むと、蓄えたエネルギーが電場と磁場の間を往復する**共振**が起きる。直列 RLC では次の関係が成り立つ。

```text
共振周波数: ω₀ = 1/√(LC)   (Z_L と Z_C が打ち消し合い、|Z| が最小 = R のみ)
Q 値:      Q = ω₀L/R = (1/R)√(L/C)   (蓄積エネルギー / 1周期の散逸 × 2π)
帯域幅:    Δω = ω₀/Q   (Q が高いほど鋭い共振)
```

これは解析第2回の2階 ODE の複素インピーダンス版であり、`Q = 1/2ζ` で減衰比と結ばれる。高 Q は選択性・低損失(水晶振動子 Q ~10⁴–10⁶、原子時計の共振 Q ~10¹⁰)、低 Q は広帯域・高減衰であり、用途が Q を決める。設計上の注意は**意図しない共振**である: 電源分配網 (PDN) では、デカップリングコンデンサの C と実装の寄生 L が共振回路を形成し、特定周波数でインピーダンスが跳ね上がる**反共振 (anti-resonance)** ピークを作る。容量の異なるコンデンサを並べて周波数帯を分担させる PDN 設計は、この Z(ω) プロファイルを目標カーブ以下に抑える作業である (Bogatin 2018, Ch. 13)。

## 工学への接続 (Engineering Connections)

- **電源分配網 (PDN) 設計**: 「電源インピーダンスを全帯域で目標値以下に」という PDN 設計仕様は、§4–5 の Z(ω) の言葉でしか書けない。デカップリングコンデンサの選定・配置は応用された共振理論である。
- **RC 遅延とオンチップ配線**: 配線抵抗とゲート容量が作る RC ローパス (§4) がデジタル信号のエッジを鈍らせる。プロセス微細化で配線遅延がゲート遅延を追い越した、という計算機アーキテクチャの転換点は回路理論の1行に由来する。
- **等価回路によるモデリング**: バッテリ(開放電圧 + 内部抵抗 + RC 網)、水晶振動子、スピーカー、さらには熱回路(熱抵抗・熱容量)まで、「測定に合う R・L・C の網」への還元 (§3) は分野横断のモデリング手法として機能する。
- **回路シミュレータの数学的正体**: SPICE = 節点解析 (KCL) で方程式を組み、直流は疎 LU、交流はフェーザ(複素連立方程式)、過渡は硬い ODE の陰解法で解く。本回と数学シリーズ(線形代数第4回・解析第2回)の合流点に立つソフトウェアである。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/hw-sw-interface/clock-power-reset.md`](../../../hardware/hw-sw-interface/clock-power-reset.md) | PDN と反共振 (§5)、デカップリング設計、RC 遅延と電源シーケンス (§4) |
| [`hardware/hw-sw-interface/bus-interconnect.md`](../../../hardware/hw-sw-interface/bus-interconnect.md) | 集中定数近似の限界 (§1) がバス速度の壁——伝送線路(次回)への動機 |
| [`engineering/energy/generation-grid-fundamentals.md`](../../../engineering/energy/generation-grid-fundamentals.md) | 交流電力・力率・無効電力 (§4)、送電網が回路理論で設計できる理由 (§1) |
| [`engineering/energy/hvdc-interconnection.md`](../../../engineering/energy/hvdc-interconnection.md) | 交流送電の無効電力問題 (§4) が HVDC を選ばせる背景 |
| [`engineering/automotive/ev/tesla-battery-powertrain.md`](../../../engineering/automotive/ev/tesla-battery-powertrain.md) | バッテリ等価回路モデル (§3)、インバータのフィルタ設計 (§4) |
| [`hardware/peripherals/audio-subsystem.md`](../../../hardware/peripherals/audio-subsystem.md) | アナログフィルタ (§4)、スピーカーの等価回路と共振 (§5) |
| [`hardware/timekeeping/clocks-time-synchronization.md`](../../../hardware/timekeeping/clocks-time-synchronization.md) | 水晶振動子の高 Q 共振 (§5) が周波数安定度の源 |

## 参考文献 (References)

- Agarwal, A., & Lang, J. H. (2005). *Foundations of Analog and Digital Electronic Circuits*. Morgan Kaufmann. — 集中定数近似の位置づけの主参照 (MIT 6.002 の教科書)
- Nilsson, J. W., & Riedel, S. A. (2019). *Electric Circuits* (11th ed.). Pearson. — 直流・交流解析・テブナン等価の標準的主参照
- Horowitz, P., & Hill, W. (2015). *The Art of Electronics* (3rd ed.). Cambridge University Press. — 実務側の設計直観の副読本
- Bogatin, E. (2018). *Signal and Power Integrity — Simplified* (3rd ed.). Prentice Hall. — PDN・反共振・実装寄生の工学側主参照
