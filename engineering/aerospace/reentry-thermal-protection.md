# 再突入と熱防護 (Re-entry & Thermal Protection Systems)

## 概要 (Overview)

軌道速度 (~7.8 km/s) で周回する機体は、質量 1 kg あたり約 30 MJ（ガソリン 1 kg の発熱量に迫る）という莫大な運動エネルギーを持つ。再突入 (re-entry) とは、このエネルギーの大部分を**大気との相互作用で熱として捨てながら減速する**過程であり、機体はそのごく一部を受け取るだけで数千度の加熱に晒される。この熱から機体を守るのが**熱防護システム (Thermal Protection System, TPS)** である。

SpaceX は TPS の2つの系譜を実用している。**Dragon** カプセルは NASA Ames 由来のアブレータ **PICA** を自社改良した **PICA-X** で使い捨て熱を凌ぎつつカプセル再使用を成立させ、**Starship** 上段は Space Shuttle の系譜に連なる**再使用型セラミックタイル**を、六角形状・機械固定・ステンレス機体という新しい組み合わせで「最小整備の完全再使用」に挑んでいる。本ドキュメントは、再突入の物理（なぜ鈍頭か、熱はどこから来るか）から説き起こし、TPS の設計空間の中に Dragon と Starship を位置づける。分析深度は Level 3。

> 前提: 軌道エネルギーと Δv は [`orbital-mechanics.md`](orbital-mechanics.md)、Starship の機体側の文脈は [`spacex-starship-superheavy.md`](spacex-starship-superheavy.md)「耐熱防護システム」節。衝撃波・実在気体効果の理論的導出は [`../../foundations/physics/fluid-dynamics/shock-waves-hypersonics.md`](../../foundations/physics/fluid-dynamics/shock-waves-hypersonics.md) に置く。

## 方法論 (Methodology)

- 再突入を**エネルギー収支**（運動エネルギー→大気加熱→機体加熱はその数%）として定式化し、加熱率のスケーリング則（速度の3乗、鈍頭半径の平方根逆数）で設計判断を読む。
- TPS を**ヒートシンク／アブレータ／再使用断熱／能動冷却**の4類型に整理し、ミッション（弾道係数・再突入速度・再使用要求）との対応で選択理由を説明する。
- Dragon (PICA-X) と Starship（セラミックタイル）を、**Space Shuttle の教訓**（接着タイルの整備性、Columbia 事故）との対比で分析する。
- 物理は NACA/NASA の古典的一次資料と標準教科書、機体固有の事実は SpaceX 公式と専門報道に基づき、典拠を脚注に付す。

## 発見事項 (Findings)

1. **再突入の熱は「摩擦」ではなく「圧縮」から来る。** 機体前方の衝撃波が空気を断熱圧縮して数千〜1万 K のプラズマにする。機体への熱伝達は主に衝撃層からの対流加熱で、軌道速度域では輻射加熱も加わる。
2. **鈍頭 (blunt body) は反直感の正解。** 尖った機体ほど衝撃波が機体に密着して熱が流れ込む。鈍い機体は衝撃波を機体から引き離し (detached bow shock)、エネルギーの大半を周囲の大気に捨てる。これは Allen & Eggers が1950年代に示した、再突入体設計の出発点である[^allen]。
3. **加熱率は速度の3乗、総加熱量は減速の仕方で決まる。** よどみ点加熱率は `q̇ ∝ √(ρ/Rn)·v³` でスケールし（Sutton–Graves）[^sutton-graves]、急降下は「短時間×高加熱率」、揚力を使う緩降下は「長時間×低加熱率」のトレードオフになる。
4. **TPS の選択は「捨てるか、貯めるか、遮るか」。** アブレータは材料を熱分解・消耗させて熱を持ち去り（高加熱率に強いが原則使い捨て）、再使用断熱タイルは低熱伝導で熱を遮る（再使用できるが脆く整備が重い）。
5. **PICA-X は「NASA の技術移転×内製化」の成果。** NASA Ames の PICA（Stardust で実証、地球再突入最速の 12.9 km/s に耐えた[^stardust]）を SpaceX がライセンス・改良し、Dragon の再使用可能な熱シールドとして内製した[^pica-nasa]。
6. **Starship のタイルは Shuttle の弱点への回答。** 六角形（直線状の隙間をなくしプラズマの侵入経路を断つ）、**機械式ピン固定**（Shuttle の接着式は1枚ごとに手作業検査が必要だった）、下地がステンレス（アルミより高温に耐え、タイル欠損への裕度が大きい）と、いずれも量産性と整備性を最優先した設計である[^starship-tps]。
7. **SpaceX は「わざとタイルを剥がして」飛ばした。** 飛行試験で意図的にタイルを外し、欠損時の機体挙動とアブレータ製バックアップ層を実データで検証している。TPS でも fly-fail-fix の反復開発が貫かれている[^flight-tests]。

## アーキテクチャ (Architecture)

### エネルギー収支 — 再突入とは何か (Energy Budget)

```
軌道上の機体 (7.8 km/s)
  運動エネルギー ≈ 30 MJ/kg
        │
        ▼  大気との相互作用（衝撃波による断熱圧縮）
  ┌────────────────────────────────┐
  │  エネルギーの大半 → 周囲の大気へ │  ← 鈍頭形状の狙い:
  │  （加熱された空気を後方に残す）  │     衝撃波を機体から引き離し
  │                                │     「熱を空気に捨てる」
  │  数 % → 機体表面へ（対流+輻射） │  ← これを TPS が受け持つ
  └────────────────────────────────┘
        │
        ▼
  地表付近 (数百 m/s) — パラシュート/動力降下で終端
```

- **Falcon 9 第1段との違い**: 第1段の再突入は ~2 km/s 級で、エネルギーは軌道再突入の約 1/15（v² に比例）。だからこそ第1段は再突入燃焼＋耐熱塗装で凌げるが、**軌道速度から帰る上段・カプセルには本格的な TPS が必須**になる（[`spacex-reusability.md`](spacex-reusability.md)）。

### なぜ鈍頭か (Blunt Body Theory)

```
 尖鋭体 (sharp)                鈍頭体 (blunt)
      ／                    ─────────
     ／← 衝撃波が機体に付着        ⌒⌒⌒  ← 離脱衝撃波 (bow shock)
    ／   境界層へ熱が流入       ┌─────┐   機体との間に高温層を挟み
   ／                          │ 機体 │   熱の大半を空気側に残す
```

Allen & Eggers (NACA) は、弾道再突入体への伝熱が**機体の抗力形状に反比例する**こと、つまり抗力の大きい鈍頭ほど機体に入る熱の割合が減ることを解析的に示した[^allen]。以来、Mercury から Dragon まで、カプセルはすべて鈍頭である。よどみ点の対流加熱率は近似的に

```
q̇ ≈ k · √(ρ/Rn) · v³        (Sutton–Graves の相関式)
```

| 記号 | 意味 | 設計への含意 |
|---|---|---|
| `ρ` | 大気密度 | 高高度で減速するほど加熱率は下がる |
| `Rn` | 機体先端の曲率半径 | **鈍い（Rn 大）ほど加熱率が下がる** |
| `v` | 速度 | **3乗**で効く——軌道再突入が桁違いに過酷な理由 |

で与えられる[^sutton-graves]。月帰還 (~11 km/s) や Stardust (12.9 km/s) では、これに衝撃層プラズマからの**輻射加熱**が重畳する[^anderson]。

### TPS の設計空間 (TPS Taxonomy)

| 類型 | 原理 | 代表例 | 再使用 | 適用域 |
|---|---|---|---|---|
| ヒートシンク | 金属塊に熱を貯める | 初期 ICBM, Mercury 初期案 | △ | 低総加熱量のみ |
| **アブレータ** | 材料の熱分解・昇華で熱を持ち去る | Apollo (Avcoat), **Dragon (PICA-X)** | 原則× | **高加熱率に最強** |
| **再使用断熱** | 低熱伝導材で熱を遮り輻射で捨てる | Shuttle タイル (LI-900), **Starship タイル** | ○ | 中加熱率×再使用要求 |
| 能動冷却 | 冷媒で熱を運び去る（発汗冷却など） | 実用例なし（Starship 初期検討[^transpiration]） | ○ | 研究段階 |

**選択の論理**: カプセル（弾道係数が小さく1回きりの高加熱）はアブレータ、再使用機（何度も帰る）は再使用断熱を選ぶ。ただし後者は Shuttle が示したとおり**整備コストとの戦い**になる。

## 技術詳細 (Technical Details)

### PICA-X — Dragon の熱シールド

**PICA (Phenolic Impregnated Carbon Ablator)** は NASA Ames が開発した低密度アブレータで、炭素繊維基材にフェノール樹脂を含浸させた材料。軽量（~0.27 g/cm³）ながら高加熱率に耐え、Stardust サンプルリターンカプセル（12.9 km/s、史上最速の人工物の地球再突入）で実証された[^stardust][^tran]。

SpaceX は NASA からライセンスを受けて **PICA-X** として自社改良・内製化した。

- **改良点**: 製造性とコストの改善（NASA との技術移転協力のもと、大幅に安価に製造できると SpaceX は説明する[^pica-nasa]）。単一の大型シールドではなくタイル状に分割して貼る。
- **再使用性**: アブレータでありながら、LEO 帰還（~7.5 km/s 級）では消耗が浅く、**同一シールドでの複数回再使用**を成立させている（Dragon 2 のカプセル再使用の一部）。アブレータ=完全使い捨て、という等式を運用で崩した点が重要。
- **月帰還マージン**: PICA-X は月軌道からの帰還速度にも耐える設計余裕を持つとされる[^pica-nasa]。

### Starship のタイル — Shuttle の教訓への回答

Space Shuttle のシリカタイル (LI-900) は再使用 TPS を史上初めて実用化したが、**約2.4万枚が1枚ずつ形状の異なる接着式**で、飛行毎の検査・補修が運用コストを支配した。Starship のタイルはこの教訓への直接の回答である[^starship-tps]。

| 観点 | Shuttle (LI-900) | Starship |
|---|---|---|
| 形状 | ほぼ全枚が個別形状 | **六角形の共通形状**を大量生産 |
| 固定 | 接着（1枚ごとに手検査） | **機械式ピン固定**（交換が速い） |
| 隙間の走り | 直線状の隙間が生じうる | **六角配置で直線経路を断つ**（プラズマの吹き込み対策） |
| 下地構造 | アルミ（~150 °C で軟化）＋断熱 | **ステンレス**（高温裕度が大きい） |
| 欠損への裕度 | 小（Columbia 事故の教訓） | 下地の耐熱＋**アブレータ製バックアップ層**で多重化[^flight-tests] |

- **下地がステンレスであることの意味**: TPS は単体では設計できず、**「タイル＋機体構造」の系**で耐熱裕度が決まる。アルミ機体の Shuttle はタイル欠損が即致命傷になりえたが、ステンレスの Starship は下地自体がある程度の過熱に耐え、欠損許容 (damage tolerance) の設計余地が生まれる（材料選定の連関は [`spacex-starship-superheavy.md`](spacex-starship-superheavy.md)「なぜステンレス鋼か」）。
- **発汗冷却の検討**: 開発初期には、ステンレス表面から液体（メタン等）を滲出させる**発汗冷却 (transpiration cooling)** も検討されたが、複雑さからタイル方式に落ち着いた。「まず枯れた原理を量産性で磨く」判断である[^transpiration]。
- **反復検証**: 飛行試験では**意図的にタイルを除去した箇所**を設けて欠損時の熱流入を実測し、タイル下のアブレータ層や取付方式を世代ごとに変えている[^flight-tests]。

### ベリーフロップ — 空力減速としての落下姿勢

Starship 上段は、再突入後の大気圏内を**機体を水平に寝かせた「腹打ち」姿勢 (belly-flop)** で落下する。

```
      再突入 (タイル面を進行方向へ)
         │  迎角 ~60–70° で滑空的に減速
         ▼
      ベリーフロップ降下
   ┌───────────────┐
   │ ⇦ 機体を水平に │  4枚のフラップ (前2+後2) で
   │    寝かせ投影   │  ピッチ/ロール/ヨーを制御
   │    面積を最大化 │  （スカイダイバーの姿勢制御と同型）
   └───────────────┘
         │  終端速度まで減速 (~舵は空力のみ)
         ▼
      着陸フリップ (landing flip)
         │  Raptor 点火 + ヘッダタンクの推進剤で
         ▼  垂直姿勢へ回頭 → 着陸燃焼
      垂直着陸 / タワーキャッチ
```

- **狙い**: 投影面積を最大化して**空力だけで終端速度まで減速**し、着陸燃焼に必要な推進剤（＝ペイロード犠牲）を最小化する。翼で滑空する Shuttle と、逆噴射に頼る Falcon 9 の中間に位置する第3の解。
- **フリップの難所**: 着陸直前の急回頭はタンク内推進剤の挙動（スロッシュ・吸込み不良）と結合し、初期の試験機 (SN8–SN15) の失敗の多くがここで起きた。専用の**ヘッダタンク**で着陸用推進剤を確保する設計は [`propellant-fluid-systems.md`](propellant-fluid-systems.md) で扱う。

### 通信ブラックアウトと Starlink

再突入中、機体周囲のプラズマ鞘 (plasma sheath) が電波を遮る**通信ブラックアウト**が生じるのが従来の常識だった。Starship の飛行試験では、プラズマの薄い機体上面側から**上空の Starlink 衛星群へ**リンクを張ることで、再突入中の映像・テレメトリをほぼ連続的に取得している。自社コンステレーションが試験データ取得のインフラとして働く好例である[^flight-tests]。

## 信頼性考察 (Reliability Considerations)

> 本分野の「セキュリティ考察」は、TPS では**欠損許容と検査性**として読み替える。

- **Columbia の教訓 (STS-107, 2003)**: 打ち上げ時に外部タンクの断熱材片が翼前縁の強化炭素 (RCC) を破損し、再突入時の高温ガス侵入で機体喪失に至った。事故調査委員会 (CAIB) は、TPS 損傷の**検査・修復手段の欠如**と、損傷リスクの組織的過小評価を指摘した[^caib]。TPS は「貼れれば終わり」ではなく、**損傷検知→評価→修復までを含む系**として設計しなければならない。
- **Starship への写像**: 機械固定タイルの迅速交換、下地ステンレスの耐熱裕度、アブレータバックアップ層は、いずれも「**欠損は起こる**」前提の多重防護であり、Columbia 型の単一欠損致命性を下げる方向の設計である。一方、軌道上での検査・修復手段は Shuttle 同様に未確立で、完全再使用の残る難所である。
- **アブレータの残厚管理**: PICA-X の再使用は「消耗が浅い」ことに依存するため、飛行後の残厚検査と再使用判定基準が信頼性の鍵になる（航空機の損傷許容設計と同型の運用課題）。

## 結論 (Conclusions)

再突入は「摩擦で熱くなる」のではない。**軌道エネルギーを大気に捨てる過程で、その数%が機体に流れ込む**のであり、TPS 設計とは「熱をどこに捨てるか」の設計である。鈍頭形状は熱を空気に捨て、アブレータは材料に捨てさせて持ち去り、再使用タイルは遮って輻射で捨てる。

SpaceX の2つの解はこの設計空間の合理的な両端にある。**Dragon/PICA-X** は NASA の実証済みアブレータを内製化し、「使い捨て材料の再使用運用」という現実解でカプセル再使用を成立させた。**Starship のタイル**は Shuttle が残した宿題（1枚ずつ違う接着タイルの整備地獄と欠損の致命性）に、六角共通形状・機械固定・ステンレス下地・アブレータ多重化で答えようとしている。完全再使用の成否は、エンジンでも着陸制御でもなく、**「軌道速度から何度でも、最小整備で帰ってくる」この熱防護**が握っている。

---

## 参考文献 (References)

> 物理は NACA/NASA の古典的一次資料と標準教科書、機体固有の事実は SpaceX 公式・専門報道に基づく。

**一次資料・査読研究 (Primary & Peer-reviewed)**

[^allen]: Allen, H. J., & Eggers, A. J. (1958). A Study of the Motion and Aerodynamic Heating of Ballistic Missiles Entering the Earth's Atmosphere at High Supersonic Speeds. *NACA Report 1381* — 鈍頭体理論の原典。伝熱が抗力形状に反比例することを解析的に示した。

[^sutton-graves]: Sutton, K., & Graves, R. A. (1971). A General Stagnation-Point Convective-Heating Equation for Arbitrary Gas Mixtures. *NASA TR R-376* — よどみ点対流加熱の相関式 `q̇ ∝ √(ρ/Rn)·v³` の標準的典拠。

[^tran]: Tran, H. K., et al. (1997). Phenolic Impregnated Carbon Ablators (PICA) as Thermal Protection Systems for Discovery Missions. *NASA TM-110440* — PICA の材料開発の一次資料。

[^stardust]: [NASA — Stardust Sample Return Capsule](https://www.nasa.gov/mission/stardust/) — 12.9 km/s（人工物の地球再突入最速）を PICA 熱シールドで凌いだ実証。Accessed: 2026-07-26.

[^caib]: Columbia Accident Investigation Board (2003). *CAIB Report Volume I*. NASA — TPS 損傷の検査・修復手段の欠如と組織要因の指摘。

[^anderson]: Anderson, J. D. (2019). *Hypersonic and High-Temperature Gas Dynamics* (3rd ed.). AIAA — 衝撃層の実在気体効果・対流/輻射加熱の標準教科書。

**公式・専門報道 (Official & Specialist reporting)**

[^pica-nasa]: [NASA Spinoff — Advanced Heat Shield Technology (PICA-X)](https://spinoff.nasa.gov/Spinoff2013/t_2.html) ／ [SpaceX — Dragon](https://www.spacex.com/vehicles/dragon/) — NASA Ames との技術移転による PICA-X の内製化、低コスト化、月帰還級の設計余裕。Accessed: 2026-07-26.

[^starship-tps]: [SpaceX — Starship（公式車両ページ・アップデート）](https://www.spacex.com/vehicles/starship) — 六角セラミックタイル、機械固定、ステンレス機体との組み合わせ。Accessed: 2026-07-26.

[^transpiration]: [Elon Musk on Starship heat shield development — SpaceNews / 公開発言の報道 (2019)](https://spacenews.com/) — ステンレス機体＋発汗冷却の初期検討から、タイル方式への収束。Accessed: 2026-07-26.

[^flight-tests]: [SpaceX — Starship Flight Test updates](https://www.spacex.com/updates/) — 意図的なタイル除去試験、タイル下のアブレータバックアップ層、再突入中の Starlink 経由テレメトリ取得。Accessed: 2026-07-26.
