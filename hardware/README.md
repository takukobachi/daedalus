# Hardware

テーマ別に整理したハードウェア分析。ソフトウェアに繋がる層を中心としつつ、物理的な工学システム（航空・自動車・電力など）まで対象を広げていく。

## 構成

- `cpu-architecture/` — CPU内部（ISA, キャッシュ, メモリ順序 …）
- `hw-sw-interface/` — HW・SW境界（MMIO, DMA, 割り込み, ABI …）

## 今後のテーマ候補

- 航空機 (aircraft) — 飛行制御 (flight control)、推進 (propulsion)、航空電子工学 (avionics)
- 自動車 (automotive) — パワートレイン (powertrain)、車載ネットワーク (in-vehicle network)、ECU
- 電力・発電 (power generation) — 発電方式、送配電 (transmission & distribution)、電力変換 (power conversion)
