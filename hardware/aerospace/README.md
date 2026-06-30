# Aerospace

航空・宇宙を題材としたハードウェア分析。固定翼機の古典航空工学（空力・推進・構造・制御）を土台に、ロケット工学、とりわけ **SpaceX** の再使用設計へと深掘りしていく。物理（空力・熱・構造）と制御ソフトウェアの接点を、エンジニアの視点から「なぜこの設計か」で読み解く。

## ドキュメント

### 基礎 (Fundamentals)

- [`aircraft-engineering-fundamentals.md`](aircraft-engineering-fundamentals.md) — 航空機の基礎工学 (aircraft engineering fundamentals)：4つの力、空力（揚力/抗力）、推進（ジェット/プロペラ）、構造・材料、安定と制御、性能
- [`orbital-mechanics.md`](orbital-mechanics.md) — 軌道力学の基礎 (orbital mechanics)：二体問題、ヴィス・ヴィヴァ式、Δv 収支、ホーマン遷移、面変更、打ち上げ損失、多段化（SpaceX 編の物理的土台）

### SpaceX

- [`spacex-rocket-engines.md`](spacex-rocket-engines.md) — ロケットエンジン (rocket engines)：Merlin（ガスジェネレータ）と Raptor（フルフロー二段燃焼）、ロケット方程式/比推力、メタン採用、Raptor 1→2→3 の反復
- [`spacex-reusability.md`](spacex-reusability.md) — 再使用性の工学 (reusability)：Falcon 9 の着陸シーケンス、ホバスラング、グリッドフィン、RTLS/ASDS、再使用の経済性
- [`spacex-starship-superheavy.md`](spacex-starship-superheavy.md) — Starship / Super Heavy：完全再使用、ステンレス鋼、33基クラスタ、Mechazilla 空中キャッチ、飛行試験史

## 今後のトピック

- アビオニクス/GNC (avionics & guidance) — 冗長計算機、フォールトトレランス、誘導制御
- 再突入と熱防護 (re-entry & TPS) — 空力加熱、耐熱タイル
