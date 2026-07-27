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

### アビオニクス (Avionics)

- [`avionics-gnc.md`](avionics-gnc.md) — アビオニクスと GNC (avionics & GNC)：GNC ループ（航法/誘導/制御）、冗長計算機とフォールトトレランス（Shuttle・777・A320・Falcon 9）、Byzantine 障害、凸最適化着陸誘導、航空宇宙バス (1553/AFDX)、Apollo AGC

## 今後のテーマ候補

- Starlink と衛星バス (satellite bus) — ホールスラスタ、レーザー衛星間リンク、フェーズドアレイ。ロケットではなく衛星側のハードウェアなので、着手時はサブディレクトリ（例: `satellites/`）を切る
- 燃焼安定性 (combustion instability) — 音響モードと燃焼の結合、噴射器設計、F-1 の教訓。エンジン編の深掘り
