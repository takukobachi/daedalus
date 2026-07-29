# 航空宇宙 (Aerospace)

航空・宇宙を題材としたハードウェア分析。固定翼機の古典航空工学（空力・推進・構造・制御）を土台に、ロケット工学、とりわけ **SpaceX** の再使用設計へと深掘りしていく。物理（空力・熱・構造）と制御ソフトウェアの接点を、エンジニアの視点から「なぜこの設計か」で読み解く。

## ドキュメント

### 基礎 (Fundamentals)

- [`aircraft-engineering-fundamentals.md`](aircraft-engineering-fundamentals.md) — 航空機の基礎工学 (aircraft engineering fundamentals)：4つの力、空力（揚力/抗力）、推進（ジェット/プロペラ）、構造・材料、安定と制御、性能
- [`orbital-mechanics.md`](orbital-mechanics.md) — 軌道力学の基礎 (orbital mechanics)：二体問題、ヴィス・ヴィヴァ式、Δv 収支、ホーマン遷移、面変更、打ち上げ損失、多段化（SpaceX 編の物理的土台）
- ノズル理論の理論編は [`../../foundations/physics/fluid-dynamics/compressible-flow-nozzles.md`](../../foundations/physics/fluid-dynamics/compressible-flow-nozzles.md)（圧縮性流れ、デ・ラバルノズル、膨張比と Isp）に置く

### SpaceX

- [`spacex-rocket-engines.md`](spacex-rocket-engines.md) — ロケットエンジン (rocket engines)：Merlin（ガスジェネレータ）と Raptor（フルフロー二段燃焼）、ロケット方程式/比推力、メタン採用、Raptor 1→2→3 の反復
- [`spacex-reusability.md`](spacex-reusability.md) — 再使用性の工学 (reusability)：Falcon 9 の着陸シーケンス、ホバスラング、グリッドフィン、RTLS/ASDS、再使用の経済性
- [`spacex-starship-superheavy.md`](spacex-starship-superheavy.md) — Starship / Super Heavy：完全再使用、ステンレス鋼、33基クラスタ、Mechazilla 空中キャッチ、飛行試験史
- [`dragon-spacecraft.md`](dragon-spacecraft.md) — Dragon 宇宙船 (Dragon spacecraft)：SuperDraco と pusher 型アボート、IDSS 自律ドッキング、パラシュート、ECLSS、タッチスクリーンコックピット、2019年地上爆発の故障解析
- [`spacex-manufacturing.md`](spacex-manufacturing.md) — 製造哲学 (manufacturing)：垂直統合、削除の5段階プロセス、金属積層造形、McGregor 全数試験、Starfactory、CRS-7 とサプライチェーン統制

### ロケット機体システム (Vehicle Systems)

SpaceX 3部作（エンジン・再使用・Starship）を機体サブシステムの断面で深掘りする続編。

- [`reentry-thermal-protection.md`](reentry-thermal-protection.md) — 再突入と熱防護 (re-entry & TPS)：鈍頭体理論、加熱のスケーリング則、アブレータvs再使用タイル、PICA-X、Starship 六角タイル、ベリーフロップ、Columbia の教訓
- [`rocket-structures-tanks.md`](rocket-structures-tanks.md) — 構造とタンク (structures & tanks)：タンク＝主構造、内圧安定、共通隔壁、Al-Li 2195×摩擦攪拌接合 vs 301 ステンレス、COPV、CRS-7/AMOS-6 故障解析、スロッシュ・ポゴ
- [`propellant-fluid-systems.md`](propellant-fluid-systems.md) — 推進剤と流体システム (propellants & fluid systems)：サブクール高密度化、load-and-go、自生加圧 vs ヘリウム加圧、沈底・ヘッダタンク、軌道上推進剤移送、火星 ISRU

### 地上系と飛行安全 (Ground Segment & Flight Safety)

- [`launch-ground-systems.md`](launch-ground-systems.md) — 射点と地上系 (launch & ground systems)：Stage 0 の思想、タンクファームと過冷却、水デリュージと音響荷重、IFT-1 射点破壊と水冷鋼板デッキ、水平統合 vs タワー統合
- [`range-safety-fts.md`](range-safety-fts.md) — 飛行安全と自律飛行中断 (range safety & AFSS)：飛行回廊・破壊限界線・瞬時落下点、FTS の判断系と執行系、コマンド式→自律式の転換、Starship 飛行試験での FTS 作動

### エンジンの深掘り (Engine Deep-Dives)

- [`combustion-instability.md`](combustion-instability.md) — 燃焼安定性 (combustion instability)：レイリー基準、チャギング/バジング/スクリーチ、F-1 の1,000回試験とボム試験、バッフルと音響キャビティ、中型多数クラスタとガス-ガス噴射の安定上の利得

### 衛星 (Satellites)

- [`satellites/`](satellites/README.md) — 衛星バス・コンステレーション（ペイロード側）
  - [`satellites/starlink-satellite-bus.md`](satellites/starlink-satellite-bus.md) — Starlink：フラットパネルとスタック展開、Kr/Ar ホールスラスタ、フェーズドアレイ、レーザー衛星間リンク、自律衝突回避、デブリ・天文考察
  - [`satellites/electric-propulsion.md`](satellites/electric-propulsion.md) — 電気推進：F = 2ηP/v_e の三角関係、ホール vs イオン、磁気シールディング、PPU、フライトヘリテージ

### ミッションと有人化 (Missions & Human Spaceflight)

- [`hls-lunar-architecture.md`](hls-lunar-architecture.md) — HLS と月着陸アーキテクチャ (lunar architecture)：給油キャンペーン、NRHO の Δv 収支、Apollo LM との対比、真空専用機の引き算、噴煙とレゴリス、GAO/OIG のリスク評価
- [`crewed-starship.md`](crewed-starship.md) — 有人 Starship の認定問題 (crew certification)：脱出系/信頼性設計/頻度実証の3パラダイム、Shuttle 遡及 PRA の実数、rule of three による必要飛行回数、段階的有人化の経路

### アビオニクス (Avionics)

- [`avionics-gnc.md`](avionics-gnc.md) — アビオニクスと GNC (avionics & GNC)：GNC ループ（航法/誘導/制御）、冗長計算機とフォールトトレランス（Shuttle・777・A320・Falcon 9）、Byzantine 障害、凸最適化着陸誘導、航空宇宙バス (1553/AFDX)、Apollo AGC

## 今後のテーマ候補

衛星側の候補（デブリ・軌道交通管理、衛星の電力・熱設計）は
`satellites/README.md` のロードマップに置く。衝撃波・極超音速の理論編は
`foundations/physics/fluid-dynamics/shock-waves-hypersonics.md` として整備済み。

- 火星往還アーキテクチャ (Mars architecture) — ISRU 収支の全体設計、地球帰還までの閉じた質量・電力収支。`hls-lunar-architecture.md` と `propellant-fluid-systems.md` の合流点
- 打ち上げ市場の経済分析 (launch economics) — 価格弾力性、需要の誘発、競合（再使用機の追随勢）。`spacex-reusability.md` の経済節の産業視点への拡張
