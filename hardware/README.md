# Hardware

テーマ別に整理したハードウェア分析。ソフトウェアに繋がる層を中心としつつ、物理的な工学システム（航空・自動車・電力など）まで対象を広げていく。

## 構成

- `cpu-architecture/` — CPU内部（ISA, キャッシュ, メモリ順序, 仮想記憶 …）
- `hw-sw-interface/` — HW・SW境界（MMIO, DMA, 割り込み, ABI …）
- `automotive/` — 自動車（車載ネットワーク, ECU …）

## 調査ロードマップ

今後調査すべきトピックは [`research-roadmap.md`](research-roadmap.md) に優先順位と根拠つきで一元管理している。主な拡張方向:

- 既存領域の深掘り — 動的実行機構 (パイプライン, 分岐予測)、キャッシュコヒーレンス、バス/相互接続 (PCIe, AXI, CXL)
- 横断トピック — ハードウェアセキュリティ (TEE, side channel)、信頼性・故障モデル (ECC, 冗長設計)
- 新規ドメイン — 航空機 (aircraft, avionics)、電力・発電 (power generation)
