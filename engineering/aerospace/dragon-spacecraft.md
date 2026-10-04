# Dragon 宇宙船 — 有人機のハードウェア (Dragon Spacecraft: Human-Rating the Hardware)

## 概要 (Overview)

**Dragon** は、SpaceX が NASA の商業輸送プログラムのもとで開発した宇宙船である。貨物型 (Dragon 1, 2010–2020) は民間機として初めて軌道からの回収に成功し、**Crew Dragon (Dragon 2)** は 2020年5月の Demo-2 で**民間開発機として史上初の有人軌道飛行**を達成、以後 ISS への定期有人輸送を担っている[^nasa-ccp]。

ロケット（Falcon 9 / Starship）が「質量を軌道に運ぶ機械」だとすれば、宇宙船は「**人間を生かして往復させる機械**」であり、要求の質が変わる。具体的には、生命維持 (ECLSS)、全飛行域での脱出 (abort)、確実な帰還（熱防護・パラシュート）、そして有人認定 (human-rating) という検証の重さである。本ドキュメントは、Dragon 2 のハードウェアを **SuperDraco による pusher 型アボート**、**自律ドッキング**、**パラシュート系**、**タッチスクリーン化されたコックピット**の設計判断を軸に分析し、2019年の地上爆発事故（SuperDraco 系の逆流着火）から有人機の故障解析を学ぶ。分析深度は Level 3。

> 関連: 熱シールド PICA-X は `reentry-thermal-protection.md`、冗長計算機の設計原理は `avionics-gnc.md`、ハイパーゴリック推進剤の一般論は `spacex-rocket-engines.md` の対極として本書で扱う。

## 方法論 (Methodology)

- 宇宙船を**有人要求から逆算**して読む: 「乗員が死なない」を頂点に、アボート→生命維持→帰還→操縦性の順で設計判断を分解する。
- アボート方式（tractor 型タワー vs pusher 型統合）を、質量・故障モード・運用の多軸で比較する。
- 2019年4月の Crew Dragon 地上爆発を SpaceX/NASA の公式発表に基づいて解剖し、ハイパーゴリック系の故障モードを一般化する。
- 有人認定は NASA 商業乗員輸送プログラム (CCP) の公式情報・ASAP 報告を典拠とする。

## 発見事項 (Findings)

1. **有人機の設計は「アボートから」始まる。** 打ち上げ機の信頼性がどれほど高くても、有人認定は**全上昇域での乗員脱出能力 (full-envelope abort)** を要求する。Dragon 2 の形状・質量・推進系配置は、アボートシステムの成立から逆算されている[^nasa-ccp]。
2. **SuperDraco の pusher 型は「捨てないアボート」。** Apollo/Soyuz の tractor 型（カプセル上部の脱出タワー、正常飛行では投棄）に対し、Dragon 2 は機体側面に埋め込んだ 8基の SuperDraco で機体ごと押し上げる。タワー投棄イベントが消え、アボート系は着水まで機体と共にあり続ける[^superdraco]。
3. **SuperDraco は 3D プリントの実用先駆。** 燃焼室はインコネル系合金の金属積層造形で、1基あたり約 71 kN（Draco (400 N 級) の約180倍）を、着火の確実なハイパーゴリック推進剤 (NTO/MMH) で瞬時に立ち上げる[^superdraco]。
4. **推進着陸は「認定の重さ」に負けた。** Dragon 2 は当初 SuperDraco による推進着陸（脚付き）を構想したが、有人認定の検証負荷を理由にパラシュート海上回収へ回帰した。技術的可能性と認定コストのトレードオフを示す好例[^propulsive]。
5. **ドッキングの自律化が运用を変えた。** Dragon 1 は ISS のアームで掴む berthing 方式だったのに対し、Dragon 2 は国際ドッキング標準 (IDSS) 準拠の機構で**自律ドッキング**する。乗員・地上の作業負荷を下げ、緊急離脱 (undock & depart) も自律で成立する[^idss]。
6. **コックピットは「タッチスクリーン＋物理ボタン」のハイブリッド。** 通常操作はタッチスクリーンに集約しつつ、消火・アボートなど**最重要系統は物理スイッチ**として残す。自律飛行が原則で、手動は多重の縮退モードである。航空機のグラスコックピット化を宇宙船で徹底した構成である[^cockpit]。
7. **2019年の地上爆発は「逆流」が原因だった。** アボートエンジンの地上試験直前、加圧系の逆止弁を高圧の酸化剤 (NTO) が逆流・衝突し、チタン部品が発火して機体が全損した。対策は逆止弁の**バーストディスク（一方向の破裂板）への置換**である。これは「弁を良くする」ではなく「逆流が物理的に不可能な部品に変える」解であり、有人系の設計哲学を象徴する[^anomaly2019]。

## アーキテクチャ (Architecture)

### 機体構成 (Vehicle Layout)

```
        ┌──────────┐
        │ ノーズコーン │ ← 開閉式: ドッキング機構 (IDSS) を保護
        ├──────────┤
        │  与圧区画   │ ← 乗員最大4名(NASA運用) / 貨物
        │ (capsule)  │    ECLSS, コックピット, 座席
        │            │    側面に SuperDraco ×8 (4ポッド×2基)
        │            │    + Draco RCS ×16
        ├──────────┤
        │ 熱シールド  │ ← PICA-X (`reentry-thermal-protection.md`)
        └──────────┘
        ┌──────────┐
        │  トランク   │ ← 非与圧貨物, 太陽電池(側面貼付), フィン
        │  (trunk)   │    アボート時の空力安定にも寄与
        └──────────┘    再突入前に分離・使い捨て
```

### アボート方式の対比 — Tractor vs Pusher (Abort Architectures)

| 観点 | Tractor 型 (Apollo / Soyuz) | Pusher 型 (Dragon 2) |
|---|---|---|
| 配置 | カプセル頂部の脱出タワー | **機体側面に埋込 (SuperDraco ×8)** |
| 正常飛行時 | 上昇途中でタワーを**投棄**（投棄自体が故障モード） | 投棄イベントなし、**全域で保持** |
| カバー範囲 | タワー投棄後は別手段に切替 | **射点から軌道投入直前まで連続** |
| 質量の扱い | 使わなければ捨てる死荷重 | 機体構造と統合（ただし常時搭載） |
| 副次利用 | なし | （当初）推進着陸への転用構想 |
| 実証 | Apollo PA/ LES 試験 | **Pad Abort (2015)・In-Flight Abort (2020)**[^ifa] |

- **In-Flight Abort 試験 (2020-01)**: 最大動圧付近で Falcon 9 から実際に脱出する飛行試験を実施し、分離・姿勢維持・パラシュート降着まで実証した。試験機の Falcon 9 は空力荷重で分解しており、「最悪条件からの脱出」を実機で確認した数少ない例である[^ifa]。

### ドッキング — Berthing から自律ドッキングへ (IDSS Docking)

```
 Dragon 1 (berthing):  接近・停止 → ISS のロボットアームが把持 → 結合
                        （人間とアームが最終工程を担う）
 Dragon 2 (docking) :  GNC が相対航法 (GPS→光学/LIDAR) で接近
                        → IDSS 準拠機構でソフトキャプチャ→剛結合
                        （自律。乗員は監視と手動override のみ）
```

- **IDSS (International Docking System Standard)**: 各国機が相互にドッキングできるよう機構インタフェース（径・捕捉機構・荷重・電力/データ結合）を定めた公開標準。Dragon 2 の NASA Docking System 準拠ポートは、ISS 側の国際ドッキングアダプタ (IDA)（それ自体を Dragon が輸送した）と結合する[^idss]。
- **自律化の含意**: 接近・結合・離脱が機上で完結するため、緊急時の**自律離脱・帰還**が成立する。有人機の安全は「行ける」ことではなく「いつでも帰れる」ことで担保される。

## 技術詳細 (Technical Details)

### SuperDraco — アボート専用エンジンの設計点

- **要求**: アボートは「失敗しつつある巨大ロケットから、1秒を争って十分な距離を取る」機能である。要求は (1) **着火の絶対確実性**、(2) **即応**（コンマ秒での全推力）、(3) 深いスロットリング（軌道離脱用 Draco と共通の推進剤系で成立させる）。
- **解**: 自己着火性 (hypergolic) の四酸化二窒素/モノメチルヒドラジン (NTO/MMH) を採用した。点火器不要で接触即燃焼、常温貯蔵可能で長期待機に向く。8基で計 ~570 kN を発生し、カプセルを数秒で射点から km 級に運ぶ[^superdraco]。
- **製造**: 燃焼室はインコネル系超合金の**金属積層造形 (DMLS)**。複雑な冷却流路を一体成形でき、2014年時点で「印刷したエンジンで人を運ぶ」判断をした先駆例（Raptor 3 の AM 多用の伏線。`spacex-rocket-engines.md`）。
- **代償**: ハイパーゴリック推進剤は毒性・腐食性が強く、地上作業の防護・後処理が重い。「性能の Isp」ではなく「確実性と即応」で推進剤を選んだ、エンジン編とは逆向きの最適化である。

### 2019年地上爆発 — 逆流と titanium fire (The 2019 Anomaly)

2019年4月20日、In-Flight Abort 試験に先立つ SuperDraco の地上静的燃焼試験の準備中、Crew Dragon 試験機 (Demo-1 帰還機) が爆発・全損した。

- **機構**: SuperDraco 点火の約100ミリ秒前、加圧系ヘリウムの起動時に、**逆止弁 (check valve) を透過して配管に漏れ出ていた液体 NTO のスラグ**が高圧で弁へ叩き込まれ、チタン製部品に高速衝突。高圧酸化剤環境下でチタンが発火し（titanium fire。チタンは高圧 NTO 中で衝撃着火しうる）、系全体の爆発に至った[^anomaly2019]。
- **対策**: 逆止弁を**バーストディスク (burst disk)** に置換。破裂板は開くまで完全密封であり、「漏れる可能性のある可動弁」を「漏れが物理的にありえない受動部品」に変えた。あわせて手順・検証を強化[^anomaly2019]。
- **教訓**: (1) 「逆止弁は逆流させない」というカタログ上の機能を、有人系では**物理的不可能性**まで格上げする、(2) 材料適合性（チタン×高圧 NTO）は静的な相性表ではなく**動的条件（衝撃・圧力）込み**で評価する、(3) 事故が地上試験で起きたこと自体が、飛行前に故障モードを洗い出す試験戦略の成果でもある。これは AMOS-6・CRS-7（`rocket-structures-tanks.md`）と並ぶ、SpaceX の「高価な授業」の三例目である。

### パラシュートと帰還 (Parachutes & Recovery)

- **構成**: ドローグ2傘で姿勢を安定・初期減速し、メイン4傘で着水速度まで落とす。メインは1傘喪失でも安全着水できる冗長設計。
- **Mark 3 への改良**: 開発中の落下試験失敗を受け、リザーブライン材を強度に優れる Zylon へ変え、縫製・荷重配分を再設計した **Mark 3** に更新。NASA と共同で数十回規模の試験を重ねて認定した。「枯れた技術」と見られがちなパラシュートが、実は認定の最難関の一つだった[^parachute]。
- **海上回収**: 着水後は回収船が捕捉。当初構想の推進着陸（SuperDraco で陸上へ）は、認定検証の負荷と熱シールド貫通部（脚）のリスクを嫌って断念された[^propulsive]。これは Starship の動力着陸（`spacex-starship-superheavy.md`）とは逆の判断であり、「有人認定の重さ」が技術選択を曲げる実例である。

### ECLSS とコックピット (Life Support & Cockpit)

- **ECLSS (Environmental Control and Life Support System)**: 与圧維持（N₂/O₂）、CO₂ 除去、温湿度制御、火災検知・消火、与圧服インタフェース（急減圧時は服が個別のバックアップ与圧を担う）。ISS 往復（数日〜半年係留）の規模では、再生型（水・酸素の再利用）ではなく**消耗品ベース**で成立させ、複雑さを避けている。
- **コックピット**: 3面の大型タッチスクリーンに飛行情報・操作を集約し、**アボート起動・消火など生死に直結する操作は物理スイッチ**として独立させる。飛行は原則自律（打ち上げ〜ドッキング〜帰還まで乗員操作なしで完結可能）で、手動操縦は縮退モードとする。これは「人間は監視者であり最後のバックアップ」という設計思想である[^cockpit]。`avionics-gnc.md` の冗長計算機構成（民生部品×多数決）は Dragon にも通底する。

## 安全考察 (Safety Considerations)

> 有人機では「セキュリティ考察」は文字どおり**乗員安全 (crew safety)** である。

- **確率要求と設計の対応**: NASA の商業乗員輸送は、ミッション喪失・乗員喪失確率 (LOC) の定量要求を課す（ISS ミッションで 1/270 級とされる）。アボートの全域カバー、パラシュート冗長、自律離脱能力は、この数値要求を満たすための独立の防護層 (layers of protection) として積まれている[^nasa-ccp]。
- **「使われない系」の検証**: アボートもバーストディスクも、正常飛行では一度も働かない。働かない系の信頼性は試験でしか作れず、Pad Abort・In-Flight Abort・パラシュート数十回という試験群が実質的な認定根拠になった。「試験で安全を買う」構図は FTS (`range-safety-fts.md`) と同型である。
- **毒性推進剤の地上安全**: ハイパーゴリック系は回収・整備時の残留推進剤処理が乗員・作業員双方のハザードになる。着水回収後の機体セーフ化手順は、設計段階から運用安全と一体で決まる。

## 結論 (Conclusions)

Dragon 2 は「ロケットの上に載る貨物」ではなく、**それ自体が完結した有人飛行システム**である。設計の頂点にあるのはアボートであり、SuperDraco の pusher 型統合・ハイパーゴリックの即応性・全域カバーは、すべて「打ち上げ機は失敗しうる」という前提から逆算されている。一方で推進着陸の断念とパラシュートへの回帰は、有人認定という制度的重力が技術の野心を現実解へ引き戻す力学を示した。

2019年の地上爆発とその対策（逆止弁をバーストディスクへ）は、SpaceX の設計哲学が有人系でどう現れるかを凝縮している: **部品を改良するのではなく、故障モードが物理的に存在しない構成に変える**。CRS-7・AMOS-6・2019年事故と並べたとき、3つの「高価な授業」はいずれも高圧流体系の界面で起き、いずれも「系の削除・単純化」で答えられた。この哲学が、月・火星へ人を運ぶ次の機体 (Starship) の有人化、つまりアボートを持たない設計をどう認定するかという、まだ答えのない問いへ引き継がれていく。

---

## 参考文献 (References)

> NASA 商業乗員輸送プログラムと SpaceX の公式発表を一次資料とし、標準文書・専門報道で補う。

**一次資料 (Primary — 公式)**

[^nasa-ccp]: [NASA — Commercial Crew Program](https://www.nasa.gov/commercial-crew-program/) — Demo-2 (2020-05-30) による民間機初の有人軌道飛行、有人認定要求（LOC 確率、full-envelope abort）、運用体制。Accessed: 2026-07-26.

[^superdraco]: [SpaceX — Dragon（公式車両ページ）](https://www.spacex.com/vehicles/dragon/) — SuperDraco ×8（約 71 kN/基）、Draco ×16、pusher 型アボート、DMLS 積層造形燃焼室。Accessed: 2026-07-26.

[^anomaly2019]: [SpaceX — Update: In-Flight Abort Static Fire Anomaly Investigation (2019-07-15)](https://www.spacex.com/updates/) — 逆止弁への NTO スラグ衝突とチタン発火という原因、バーストディスクへの置換という対策の一次発表。Accessed: 2026-07-26.

[^ifa]: [NASA/SpaceX — In-Flight Abort Test (2020-01-19)](https://www.nasa.gov/commercial-crew-program/) — 最大動圧域での実機脱出試験。分離から着水までの実証。Accessed: 2026-07-26.

[^idss]: [International Docking System Standard (IDSS) — Interface Definition Document](https://www.internationaldockingstandard.com/) — ドッキング機構の公開標準。Dragon 2 の NASA Docking System と ISS 側 IDA の適合根拠。Accessed: 2026-07-26.

**技術解説・報道 (Technical explainers & reporting)**

[^propulsive]: [SpaceX drops plans for powered Dragon landings — SpaceNews (2017-07)](https://spacenews.com/spacex-drops-plans-for-powered-dragon-landings/) — 推進着陸の断念と認定負荷・熱シールド貫通リスクの理由。Accessed: 2026-07-26.

[^parachute]: [NASA — Crew Dragon Parachute Testing (Mark 3)](https://blogs.nasa.gov/commercialcrew/) — 落下試験での不具合と Zylon 化・再設計、認定までの試験キャンペーン。Accessed: 2026-07-26.

[^cockpit]: [NASA/SpaceX — Crew Dragon displays and manual flying（Demo-2 関連公式説明・乗員報告）](https://www.nasa.gov/commercial-crew-program/) — タッチスクリーン主体＋重要系統の物理スイッチ、自律飛行原則と手動縮退モード。Accessed: 2026-07-26.

**本リポジトリ内参照**

- `reentry-thermal-protection.md` — PICA-X 熱シールド
- `avionics-gnc.md` — 冗長計算機・fail-safe 設計
- `rocket-structures-tanks.md` — CRS-7 / AMOS-6（高圧流体系の故障解析の姉妹例）
