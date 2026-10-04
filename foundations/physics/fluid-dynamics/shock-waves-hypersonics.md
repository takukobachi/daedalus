# 衝撃波と極超音速流 (Shock Waves & Hypersonic Flow)

## 概要 (Overview)

超音速の流れは、前方に「これから障害物が来る」と知らせる術を持たない——圧力の乱れは音速でしか伝わらず、流れ自身がそれより速いからである。その結果、情報の不連続な壁、すなわち**衝撃波 (shock wave)** が生じる。衝撃波を横切る空気は、分子スケールの厚みの中で急減速・急圧縮・急加熱され、エントロピーが跳ぶ。

本ドキュメントは、衝撃波がなぜ立つのか（波の突っ立ち）、垂直衝撃波・斜め衝撃波・膨張扇の基本関係、そしてマッハ数がさらに上がった**極超音速 (hypersonic, M ≳ 5)** 領域で現れる新しい物理——薄い衝撃層、空力加熱の支配、実在気体効果（解離・電離）——までを積み上げる。到達点は、`hardware/aerospace/reentry-thermal-protection.md` が結果として使った2つの主張——**鈍頭が正解であること**と**加熱率が v³ でスケールすること**——の理論的な足場である。

## 前提知識 (Prerequisites)

- **圧縮性流れの基礎** — マッハ数、等エントロピー流、面積–速度関係（[`compressible-flow-nozzles.md`](compressible-flow-nozzles.md)）
- **熱力学** — 断熱過程、エントロピー、第二法則（[`../thermodynamics-statistical/laws-entropy-heat-engines.md`](../thermodynamics-statistical/laws-entropy-heat-engines.md)）
- 応用側の文脈: 再突入と熱防護（`../../../hardware/aerospace/reentry-thermal-protection.md`）

## 本論 (Main Content)

### 1. なぜ衝撃波が立つのか (why shocks form)

圧力波（音波）の伝播速度は局所音速 `a = √(γRT)` で、**圧縮された（温度が上がった）領域ほど速い**。有限振幅の圧縮波は、波の後方（高温部）が前方に追いつくため波形が前傾していき、ついに**多価**（同じ場所に複数の状態）になろうとする。物理はそれを許さず、波形は分子粘性で支えられる**厚さ数分子自由行程の不連続面**——衝撃波——に収束する。

- **逆は起きない**: 膨張波は後方ほど遅くなるため波形は寝ていき、不連続にはならない（膨張衝撃波が存在しないことは第二法則からも従う——後述）。
- **超音速物体の前に立つ理由**: M > 1 では物体の存在を知らせる圧力情報が上流へ届かない。流れは「知らずに」物体に到達し、直前の衝撃波で一挙に減速・転向させられる。

### 2. 垂直衝撃波 — 不連続の会計 (normal shocks)

衝撃波を挟む上流 (1)・下流 (2) の状態は、質量・運動量・エネルギー保存（**ランキン–ユゴニオ関係**）で結ばれ、理想気体なら上流マッハ数 M₁ だけの関数になる:

| 量 | 変化 | 挙動 |
|---|---|---|
| マッハ数 | M₁ > 1 → **M₂ < 1** | 必ず亜音速へ落ちる |
| 静圧・静温・密度 | **急上昇** | M₁ が大きいほど激しい（p₂/p₁ は M₁² で伸びる） |
| 全温 T₀ | **不変** | エネルギーは保存される（断熱） |
| 全圧 p₀ | **低下** | 仕事に使える圧力が失われる |
| エントロピー | **増大** | 不可逆過程。逆向き（膨張衝撃波）は s 減少となり第二法則が禁止 |

- **全圧損失の意味**: 衝撃波は「エネルギーを失う」のではなく「エネルギーの**質**を落とす」（組織的な運動エネルギー→乱雑な熱）。超音速機のインテークが複数の弱い斜め衝撃波で徐々に減速するのは、1枚の強い垂直衝撃波より全圧損失が小さいからである。

### 3. 斜め衝撃波と膨張扇 (oblique shocks & expansion fans)

流れが凹角（壁が流れ側へ折れる）に出会うと**斜め衝撃波**が立ち、流れは波面で折れ曲がる。凸角では連続的な**膨張扇（プラントル–マイヤー膨張）** が開き、流れは等エントロピーに加速・転向する。

```
   凹角 (圧縮)                  凸角 (膨張)
   ──────╲  斜め衝撃波          ──────╮ 膨張扇（扇状の特性線群）
     M₁ → ╲╲                    M₁ →  ╰─╮
            ╲╲→ M₂ < M₁ (減速)         ╰──→ M₂ > M₁ (加速)
      不連続・エントロピー増         連続・等エントロピー
```

- **転向角と波角の関係 (θ-β-M)**: 同じ転向角に対し弱解と強解があり、通常は弱解が実現する。転向角には M₁ ごとの**最大値**があり、それを超える角度（＝鈍い物体）では衝撃波が付着できず、**離脱衝撃波 (detached bow shock)** として物体前方に立つ。
- **超音速飛行の抗力の正体**: 衝撃波が運ぶエネルギーは**造波抗力 (wave drag)** として機体に課金される。超音速機の細長い形状・薄い翼・エリアルールは、すべて衝撃波を弱くする（転向角を小さくする）ための設計である（`../../../hardware/aerospace/aircraft-engineering-fundamentals.md` の高速側の延長）。

### 4. 極超音速 — 量ではなく質が変わる (the hypersonic regime, M ≳ 5)

マッハ5前後を境に、単なる「速い超音速」ではない新しい性質が支配的になる:

1. **薄い衝撃層 (thin shock layer)**: 衝撃波背後の密度が桁で上がるため、衝撃波は機体表面に薄く張り付く。衝撃層内に機体まわりの流れが凝縮され、境界層と衝撃波が干渉し始める。
2. **マッハ数独立の原理 (Mach number independence)**: M が十分大きいと、圧力係数など無次元量が M にほぼ依存しなくなる。設計上は「M20 と M25 の空力はほぼ同じ」——再突入体の空力設計が広い速度域で1つの形で済む理由。
3. **空力加熱の支配**: 抗力より熱が設計を決める。よどみ点加熱率は `q̇ ∝ √(ρ/Rn)·v³`（Sutton–Graves。`reentry-thermal-protection.md` で使用）——エネルギー流束が ρv³ に比例することの表れで、**速度の3乗**が極超音速の過酷さの根源である。
4. **実在気体効果 (real gas effects)**: 衝撃層温度が数千 K を超えると、空気は理想気体をやめる——振動励起、O₂ の解離 (~2,500 K〜)、N₂ の解離 (~4,000 K〜)、さらに電離。解離はエネルギーを化学結合に吸い込み、衝撃層温度は理想気体の予測より下がる一方、**原子状酸素**は表面材料と反応し（TPS の触媒性・酸化の問題）、電離は**通信ブラックアウト**のプラズマ鞘を作る。
5. **輻射加熱**: 高温衝撃層自体が光り、対流とは別経路で機体を加熱する。軌道再突入 (~8 km/s) では対流が支配的だが、月帰還 (~11 km/s) 以上では輻射の寄与が急増する。

### 5. 鈍頭理論の再訪 — 衝撃波で熱を捨てる (blunt bodies revisited)

`reentry-thermal-protection.md` の鈍頭体理論（Allen & Eggers）は、本ドキュメントの言葉で言い直せる:

- 尖鋭体: 弱い付着斜め衝撃波 → エントロピー増（＝空気側への散逸）が小さい → 運動エネルギーの多くが**境界層の摩擦**すなわち機体表面経由で熱化する。
- 鈍頭体: 強い離脱衝撃波 → 通過する空気が**衝撃波自体で大量にエントロピー生成**（高温化）→ エネルギーの大半は機体に触れずに後流へ流される。加えて Rn が大きいほど `q̇ ∝ 1/√Rn` で加熱率が下がる。
- つまり鈍頭とは「**衝撃波という散逸装置を機体の代わりに働かせる**」設計である。不連続でエントロピーを生む衝撃波の「損失」（第2節）が、再突入では防護の道具に反転する。

## 工学への接続 (Engineering Connections)

- **再突入機の形状設計**: 離脱衝撃波を保つ鈍頭、加熱率と Rn のトレードオフ、マッハ数独立を使った広速度域の空力設計——カプセル (Dragon) から揚力体 (Starship のベリーフロップ) までの形状の語彙を与える。
- **TPS 要求の源泉**: 対流/輻射の配分、原子状酸素との表面反応（触媒加熱）、解離による衝撃層温度の変化は、TPS 材料（アブレータ/タイル）の選定と設計マージンを直接決める（`../../../hardware/aerospace/reentry-thermal-protection.md`）。
- **超音速域の制御翼**: Falcon 9 のグリッドフィン（`../../../hardware/aerospace/spacex-reusability.md`）は、格子面それぞれに斜め衝撃波系を作ることで、遷音速〜超音速で失速的な挙動をしにくい制御面として働く——衝撃波の理解が着陸誘導の空力を支える例。
- **ノズルとの往復**: 過膨張ノズル内の流れ剥離（`compressible-flow-nozzles.md`）は、ノズル内に立つ衝撃波と境界層の干渉問題であり、本ドキュメントの垂直/斜め衝撃波の応用である。
- **通信ブラックアウト**: 電離が作るプラズマ鞘の遮蔽は、再突入中のテレメトリ設計（Starlink 経由の上面リンク）の物理的前提である。

## Daedalus内の応用 (Applications within Daedalus)

- [`../../../hardware/aerospace/reentry-thermal-protection.md`](../../../hardware/aerospace/reentry-thermal-protection.md) — 鈍頭体理論・Sutton–Graves 則・ブラックアウトの理論的土台（本ドキュメントが供給）
- [`../../../hardware/aerospace/spacex-reusability.md`](../../../hardware/aerospace/spacex-reusability.md) — グリッドフィンの超音速空力、再突入燃焼の意味（動圧・加熱の緩和）
- [`../../../hardware/aerospace/aircraft-engineering-fundamentals.md`](../../../hardware/aerospace/aircraft-engineering-fundamentals.md) — 亜音速空力からの接続（造波抗力、面積則）
- [`compressible-flow-nozzles.md`](compressible-flow-nozzles.md) — 等エントロピー流の対極としての衝撃波、過膨張剥離
- [`../thermodynamics-statistical/laws-entropy-heat-engines.md`](../thermodynamics-statistical/laws-entropy-heat-engines.md) — エントロピー増大則（膨張衝撃波の不在、全圧損失の意味）

## 参考文献 (References)

- Anderson, J. D. (2020). *Modern Compressible Flow: With Historical Perspective* (4th ed.). McGraw-Hill — 衝撃波関係式・θ-β-M・プラントル–マイヤー膨張の標準教科書。
- Anderson, J. D. (2019). *Hypersonic and High-Temperature Gas Dynamics* (3rd ed.). AIAA — 極超音速流・実在気体効果・空力加熱の標準教科書。
- Liepmann, H. W., & Roshko, A. (1957). *Elements of Gasdynamics*. Wiley (Dover 復刻) — 気体力学の古典。波の突っ立ちと衝撃波形成の明快な導出。
- Allen, H. J., & Eggers, A. J. (1958). A Study of the Motion and Aerodynamic Heating of Ballistic Missiles Entering the Earth's Atmosphere at High Supersonic Speeds. *NACA Report 1381* — 鈍頭体理論の原典（`reentry-thermal-protection.md` と共通）。
- Sutton, K., & Graves, R. A. (1971). A General Stagnation-Point Convective-Heating Equation for Arbitrary Gas Mixtures. *NASA TR R-376* — よどみ点加熱の相関式。
