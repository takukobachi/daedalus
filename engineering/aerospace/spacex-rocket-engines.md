# SpaceX のロケットエンジン — Merlin と Raptor (SpaceX Rocket Engines: Merlin & Raptor)

## 概要 (Overview)

ロケットエンジンは、SpaceX の競争力の源泉である。同社は2つの主力エンジンを持つ。

- **Merlin (マーリン)**: Falcon 9 / Falcon Heavy を飛ばす、ケロシン (RP-1)／液体酸素のエンジン。枯れた**ガスジェネレータサイクル (gas-generator cycle)** を、再使用と量産に振り切って磨き上げた。
- **Raptor (ラプター)**: Starship / Super Heavy 用の、液体メタン (CH₄)／液体酸素エンジン。実用化された世界で唯一の**フルフロー二段燃焼サイクル (full-flow staged combustion, FFSC)** を採用し、燃焼室圧でロケットエンジン史上の最高値を更新し続けている。

本ドキュメントは、前提となる **ツィオルコフスキーのロケット方程式**と**比推力 (Isp)** を確認したうえで、Merlin と Raptor を「サイクルの選択」という設計判断として対比する。とりわけ Raptor の FFSC が「なぜ難しく、なぜ価値があるか」を、ガスジェネレータ／部分二段燃焼との比較で解剖する。分析深度は Level 3。

> 前提知識として、[`aircraft-engineering-fundamentals.md`](aircraft-engineering-fundamentals.md)「ロケットへの橋渡し」節（航空機との差分：自前の酸化剤・推力支配・ロケット方程式）を踏まえる。ノズル・膨張比・Isp の式の理論的導出は [`../../foundations/physics/fluid-dynamics/compressible-flow-nozzles.md`](../../foundations/physics/fluid-dynamics/compressible-flow-nozzles.md) に置く。

## 方法論 (Methodology)

- まず**ロケット方程式**と**比推力**で、エンジン性能が機体能力 (Δv) にどう影響するかを定量化する。
- エンジン**サイクル**を「ターボポンプをどう駆動し、その駆動ガスをどう扱うか」で分類し、ガスジェネレータ→部分二段燃焼→フルフロー二段燃焼の系譜に Merlin と Raptor を位置づける。
- Raptor 1→2→3 の世代変化を、燃焼室圧・推力・推力重量比・製造性の軸で追い、SpaceX の「反復による先鋭化」を読む。
- 推進剤選択（ケロシン vs メタン）を、Isp・再使用（すす）・現地生産 (ISRU)・コストの観点で評価する。
- 数値は SpaceX 公表値・標準的な技術解説に基づき、桁感を優先する（典拠は脚注）。

## 発見事項 (Findings)

1. **エンジンの良さは「比推力 (Isp)」と「推力重量比」の二軸で決まる。** Isp は推進剤を「どれだけ効率よく」使うか（＝Δv）を、推力重量比は「どれだけ軽く力を出すか」を表す。ロケット方程式はこの2軸を機体性能に翻訳する。
2. **サイクル選択が効率の天井を決める。** ガスジェネレータは駆動ガスを捨てるため数%損をする。二段燃焼は駆動ガスも主燃焼室で使い切るため高効率だが、高温・高圧のガスをポンプ後段で扱う難所が生まれる。
3. **Merlin は「枯れた技術 × 量産・再使用」の勝利。** 性能の頂点ではなく、9基クラスタ・エンジンアウト耐性・整備性・低コストで Falcon 9 の再使用を成立させた。
4. **Raptor の FFSC は「効率の理論的上限を、製造性とともに」狙う賭け。** 両推進剤を別々のプリバーナで完全に気化・駆動してから主燃焼室で燃やし切る。実用化は SpaceX が史上初[^raptor-wiki]。
5. **燃焼室圧の更新が Raptor の物語。** Raptor は FFSC を主燃焼室圧 250 bar 級で実用化し、Raptor 2 で 300 bar へ到達した。これは査読誌 (CEAS Space Journal) でも確認される量産エンジン最高水準で、ロシア RD-180 の 267 bar を上回った[^ceas][^ea-raptor]。Raptor 3 は 350 bar・海面推力 280 tf へ到達した[^spacex-raptor3][^techtimes-raptor3]。
6. **メタンへの転換は「再使用と火星」のための選択。** すす (coking) が少なく再使用に向き、火星で現地生産 (ISRU) しやすい。Isp もケロシンより高い。
7. **製造が性能の一部になった。** Raptor 3 は外部ヒートシールドや多数の外部配管を、金属積層造形 (metal AM) と部品統合で内部化し、推力重量比をほぼ倍化した[^techtimes-raptor3]。

## アーキテクチャ (Architecture)

### 前提 — ロケット方程式と比推力 (Rocket Equation & Specific Impulse)

ロケットの到達能力は **ツィオルコフスキーのロケット方程式**で表される。

```
Δv = v_e · ln(m0 / mf) = Isp · g0 · ln(m0 / mf)
```

| 記号 | 意味 |
|---|---|
| `Δv` | 速度増分（軌道投入に必要な「予算」。LEO までおよそ 9.4 km/s） |
| `v_e` | 有効排気速度 |
| `Isp` | 比推力 (specific impulse, 秒)。`v_e = Isp · g0` |
| `m0 / mf` | 初期質量／空虚質量比（推進剤をどれだけ積むか） |

要点は **対数 (ln)** にある。質量比を増やしても Δv は対数的にしか伸びないため、`v_e`（＝Isp）を上げることと、`mf`（構造・エンジンの乾燥質量）を減らすことが本質的に効果を持つ。**Isp はエンジンの燃費、推力重量比は機体の軽さ**であり、この2つがエンジン設計の双子の目標である。

### エンジンサイクルの分類 (Engine Cycles)

ロケットエンジンの難所は「高圧の推進剤を主燃焼室へ送り込む**ターボポンプ**を、何で回すか」にある。その駆動ガスの扱いでサイクルが分かれる。

```
[1] ガスジェネレータ (open cycle) — Merlin
    一部の推進剤を別燃焼器で燃やしタービンを回し、その排ガスは「捨てる」
    ┌─ プリバーナ ─▶ タービン ─▶ 排気（捨てる＝損失）
    │                 ↳ ポンプ駆動
    主流 ─────────────────────────▶ 主燃焼室 ─▶ ノズル
    ◎ 単純・堅牢・安価   ✗ 駆動ガス分の効率損（数%）

[2] 部分二段燃焼 (staged combustion) — RD-180, SSME 等
    駆動ガス（酸素過剰 or 燃料過剰の一方）をタービン後に主燃焼室へ送り、燃やし切る
    ◎ 高効率   ✗ 高温ガスがポンプ系を腐食/脆化させる難所

[3] フルフロー二段燃焼 (full-flow staged combustion, FFSC) — Raptor
    酸素過剰・燃料過剰の「2つの」プリバーナを持ち、
    “全ての”推進剤をタービンに通してから主燃焼室で完全燃焼
    ◎ 最高効率・タービン低温化・ポンプ間シール緩和   ✗ 設計・製造が最難関
```

#### フルフロー二段燃焼 (FFSC) の核心

Raptor の FFSC は2つのプリバーナを持つ[^raptor-wiki]。

- **酸素過剰プリバーナ (oxygen-rich preburner)**: 大量の LOX と少量のメタンを燃やし、**酸素リッチの高温ガス**で酸素側ターボポンプを駆動。
- **燃料過剰プリバーナ (fuel-rich preburner)**: 大量のメタンと少量の LOX を燃やし、**燃料リッチの高温ガス**でメタン側ターボポンプを駆動。

両者のガスは**いずれも主燃焼室に入り、そこで完全に燃え切る**。これが「フルフロー」の意味であり、推進剤を1滴も捨てない。利点は次のとおりである。

1. **効率最大化**: 駆動ガスを捨てないため、原理的に最も高い Isp/圧力を狙える。
2. **タービンの低温化**: 全流量でタービンを回すので、同じ仕事でもガス温度を低く保てる→部品寿命・再使用に有利。
3. **ポンプ間シールの緩和**: 燃料側ターボポンプには燃料リッチガス、酸素側には酸素リッチガスが流れるため、両者を隔てる「燃料と酸素が混じると爆発する」シールの難易度が下がる（同種流体同士の界面で済む）。

代償は、酸素リッチガス（金属を激しく酸化させる）に耐える材料と、2系統のプリバーナ・ターボポンプを協調制御する難しさ。**SpaceX はこれを量産・再使用レベルで実用化した初の組織**である[^raptor-wiki]。

### 主要諸元の比較 (Specifications)

| 項目 | Merlin 1D | Raptor 1 | Raptor 2 | Raptor 3 |
|---|---|---|---|---|
| 推進剤 | RP-1 / LOX | CH₄ / LOX | CH₄ / LOX | CH₄ / LOX |
| サイクル | ガスジェネレータ[^f9official] | FFSC | FFSC | FFSC |
| 海面推力 | ~845 kN (190,000 lbf)[^f9official] | ~185 tf[^ea-raptor] | ~230 tf[^ea-raptor] | ~280 tf[^spacex-raptor3] |
| 主燃焼室圧 | ~97 bar 級 | ~250 bar[^ceas] | **~300 bar**[^ceas] | **~350 bar**[^techtimes-raptor3] |
| 海面 Isp | 282 s[^merlin] | — | ~327 s 級 | ~350 s[^techtimes-raptor3] |
| 真空 Isp | 311 s[^merlin] | — | — | — |
| 乾燥質量 | — | ~2,080 kg | ~1,630 kg | <1,525 kg[^techtimes-raptor3] |
| 搭載機 | Falcon 9 / Heavy | Starship 初期 | Starship V2 | Starship V3 / Super Heavy |

> Raptor 2 の 300 bar は当時のあらゆる量産エンジンを超える主燃焼室圧で、RD-180（267 bar）を上回った[^ceas]。Raptor 3 はさらに 350 bar へ。チャンバ圧の上昇は、同じノズルでより多くの推進剤を燃やし、推力と効率を同時に押し上げる。
>
> **注（Raptor 3 推力の幅）**: Raptor 3 の海面推力は、SpaceX 公表・報道で **250〜330 tf** の幅で語られている。2024年8月の SpaceX 公表は 280 tf[^spacex-raptor3]、公式 Starship V3 更新ページは 250 tf 級、V3 統合後の報道では 330 tf 級とされる[^v3]。これは開発の進行と機体統合に伴う更新の途上にあるためで、本表の 280 tf は「実証された代表値」として扱い、一意に確定した最終値ではない点に留意する。燃焼室圧（350 bar）や Isp（~350 s）はより安定して引用される。

### Merlin と Falcon 9 のクラスタ (Clustering & Engine-Out)

Falcon 9 第1段は **Merlin 1D を9基**束ね、海面で合計 約7,605 kN (1,710,000 lbf) を出す（オクタウェブ配置）[^f9official]。設計思想は次のとおりである。

- **エンジンアウト耐性 (engine-out capability)**: 9基のうち**最大2基が停止しても**ミッションを継続できる冗長性[^merlin]。多発化は信頼性を「数」で買う設計。
- **再使用への適合**: 着陸時は中央1基（または3基）を再着火して減速する。深い**スロットリング**と複数回**再着火**ができることが、Merlin を再使用エンジンたらしめる。
- **量産**: 1機で10基（第1段9＋第2段1基の真空型）使うため、年間数百基を製造する。**ガスジェネレータの単純さ**が量産・低コスト・整備性に直結する。

## 技術詳細 (Technical Details)

### 推進剤選択 — ケロシン vs メタン (Propellant Choice)

| 観点 | RP-1 (ケロシン, Merlin) | CH₄ メタン (Raptor) |
|---|---|---|
| Isp | やや低い | 高い（密度比 Isp も良好） |
| すす (coking) | 多い→再使用時の洗浄・配管詰まり | **少ない→再使用に好適** |
| 密度 | 高い（タンク小型化に有利） | 中（液水よりずっと密、液酸と温度域が近い） |
| 保管温度 | 常温付近 | 極低温（液酸と近く、共通設備で扱える） |
| 現地生産 (ISRU) | 困難 | **火星で CO₂ + H₂O から合成可能（サバティエ反応）** |
| コスト | 安い | 安い・豊富 |

SpaceX が Raptor でメタンへ移ったのは、**(1) すすが少なく再使用に向く、(2) 火星で燃料を現地生産できる、(3) Isp が高い**という3点が、長期の「再使用＋火星」戦略に合致するため。Merlin（ケロシン）は Falcon の枯れた資産として併存させ、新規開発リスクを Raptor に集中させた。

### 再生冷却とスロットリング (Regenerative Cooling & Throttling)

- **再生冷却 (regenerative cooling)**: 燃焼室・ノズルの壁内チャネルに低温の推進剤（Raptor ではメタン）を流して壁を冷やし、その推進剤を予熱してプリバーナへ送る。熱を捨てずに回収する手法[^ea-raptor]。Raptor 3 は外部ヒートシールドを廃し、露出面すべてを一体の再生冷却で覆う設計に進化した[^techtimes-raptor3]。
- **スロットリングと再着火**: 着陸（Falcon 9）やホバリング相当の減速には、推力を深く絞り（throttle down）、複数回**再着火**できることが必須。化学・タービン・冷却を協調させて広い作動域を確保する。

### 設計の反復 — Raptor 1 → 2 → 3 (Iterative Refinement)

SpaceX の特徴は「飛ばして壊して直す」反復にある。Raptor での経過を以下に示す。

- **1→2**: スロート（ノズル喉部）を拡げて流量を増やし推力を ~185→~230 tf に、燃焼室圧を 250→300 bar へ。部品点数と製造コストを削減[^ea-raptor]。
- **2→3**: 350 bar・280 tf へ。外部配管・センサ・ヒートシールドを**金属積層造形 (metal AM) と部品統合**で内部化し、推力重量比をほぼ倍に、製造の複雑さを大幅削減[^techtimes-raptor3]。

**「性能向上」と「製造簡素化」を同時に達成する**ことが Raptor 開発の核心であり、これがエンジンの大量生産（Super Heavy 1機で33基）を可能にしている。

## セキュリティ・信頼性考察 (Reliability & Failure Modes)

> 航空・宇宙分野では「セキュリティ」を**信頼性・故障モード**として読み替える（[`aircraft-engineering-fundamentals.md`](aircraft-engineering-fundamentals.md) と同様）。

| 故障モード | 機構 | 対策・設計思想 |
|---|---|---|
| **エンジン停止** | 燃焼不安定、ポンプ故障、配管漏れ | **多発化による冗長**（Falcon 9 は2基喪失まで許容[^merlin]）。再着火・スロットリングで運用包絡を確保 |
| **酸素リッチ環境の腐食** | FFSC の酸素過剰プリバーナで金属が激しく酸化 | 耐酸化合金、内部の流路設計、AM による一体成形で接合部削減 |
| **燃焼不安定 (combustion instability)** | 高圧燃焼室での圧力振動が共振・破壊に至る | インジェクタ設計、地上での反復試験、高い燃焼室圧ほど難度増 |
| **再使用に伴う劣化** | 熱サイクル・すす・疲労 | メタン採用（低すす）、低温タービン（FFSC の利点）、点検・整備性の作り込み |

エンジンの信頼性は「壊れない部品」ではなく、**(1) 多発冗長、(2) 反復試験で解消した既知の故障モード、(3) 再使用に耐える材料・冷却**の組み合わせで作られる。航空機の「冗長化＋型式証明」と同じ思想が、より過酷な熱・圧力環境で先鋭化している。

## 結論 (Conclusions)

Merlin と Raptor は、同じ SpaceX の中で対照的な設計判断を体現する。

- **Merlin** は「枯れたガスジェネレータサイクル」を量産・再使用・低コストへ振り切り、**性能の頂点ではなく運用の経済性**で Falcon 9 の再使用を成立させた。
- **Raptor** は「フルフロー二段燃焼」という最難関サイクルを世界で初めて実用化し、**燃焼室圧と推力重量比を反復で更新**しながら、Starship の完全再使用という野心を支える。

両者を貫くのは、ロケット方程式が突きつける2つの命題、すなわち**Isp（効率）を上げ、乾燥質量（構造）を下げる**ことへの執着である。次は、このエンジンを基に **Falcon 9 の再使用性の工学**（着陸シーケンス・グリッドフィン・経済性）を深掘りし、エンジンの「再着火・スロットリング」がどう着陸を成立させるかを見る。

---

## 参考文献 (References)

> 一次資料（SpaceX 公式・査読論文）を優先し、技術解説（Everyday Astronaut 等）と報道で補う。

**一次資料 (Primary — 公式・査読)**

[^f9official]: [SpaceX — Falcon 9（公式車両ページ）](https://www.spacex.com/vehicles/falcon-9/) — 第1段は Merlin 9基、各基 海面推力 845 kN (190,000 lbf)、合計 7,605 kN (1,710,000 lbf)。ガスジェネレータサイクル、単軸ターボポンプ、再生冷却の銅合金ライナ、再着火可能な Merlin Vacuum。Accessed: 2026-06-30. 関連: [Falcon User's Guide (2025-05-09, PDF)](https://www.spacex.com/assets/media/falcon-users-guide-2025-05-09.pdf)。

[^spacex-raptor3]: [SpaceX 公式 (X) — Raptor 3 sea-level variant: Thrust 280 tf](https://x.com/SpaceX/status/1819772716339339664) — Raptor 3 海面推力 280 tf。Accessed: 2026-06-30.

[^ceas]: Springer / CEAS Space Journal: [Selection of propulsion characteristics for systematic assessment of future European RLV-options](https://link.springer.com/article/10.1007/s12567-024-00564-w) (2024). — 査読誌。Raptor をフルフロー二段燃焼（FFSC）と記載し、燃焼室圧を少なくとも250 bar、最新の Raptor 2 で 300 bar と整理。Accessed: 2026-06-30.

[^merlin]: [SpaceX Merlin — Wikipedia](https://en.wikipedia.org/wiki/SpaceX_Merlin)（公式値を集約）— Isp 282 s(SL)/311 s(vac)、ガスジェネレータ、9基クラスタ・最大2基喪失耐性。一次値は[^f9official]を優先。Accessed: 2026-06-30.

[^raptor-wiki]: [SpaceX Raptor — Wikipedia](https://en.wikipedia.org/wiki/SpaceX_Raptor) — フルフロー二段燃焼サイクル、酸素過剰／燃料過剰の2プリバーナ構成（一次資料への索引として使用）。Accessed: 2026-06-30.

[^v3]: [SpaceX — Starship V3（公式アップデート）](https://www.spacex.com/updates/starship-v3) ／ 報道: [Scientific American — Watch SpaceX launch Starship V3](https://www.scientificamerican.com/article/watch-spacex-launch-starship-v3-the-tallest-and-most-powerful-rocket-yet/)。 — Raptor 3 推力の公表値の幅（海面 250〜330 tf）と V3 機体諸元。Accessed: 2026-06-30.

**二次資料 (Secondary — 技術解説・報道)**

[^ea-raptor]: [Raptor 1 vs Raptor 2: What did SpaceX change? — Everyday Astronaut](https://everydayastronaut.com/spacex-raptor-engine-comparison/)（Musk へのインタビューに基づく技術解説）— 推力 ~185→~230 tf、スロート拡大、再生冷却、RD-180 (267 bar) 比較。Accessed: 2026-06-30.

[^techtimes-raptor3]: [SpaceX Raptor 3 Engines and the Next Step in Starship Propulsion — New Space Economy](https://newspaceeconomy.ca/2026/04/16/spacex-raptor-3-engines-and-the-next-step-in-starship-propulsion/) ／ [NextBigFuture](https://www.nextbigfuture.com/2025/09/205681.html) — Raptor 3 は 350 bar・280 tf・Isp ~350 s・乾燥質量<1,525 kg、外部ヒートシールド廃止・一体再生冷却・金属積層造形による推力重量比ほぼ倍化。Accessed: 2026-06-30.

その他の学術的典拠（査読前プレプリント含む、原理の確認に使用）を以下に挙げる。

- *Conceptual Design and Performance Analysis of a SpaceX Raptor Rocket Engine* (ResearchGate, 2025). — FFSC の性能解析。
- *Preliminary design of a Raptor-like engine* (ResearchGate, 2022). — Raptor 類似 FFSC エンジンの予備設計。
