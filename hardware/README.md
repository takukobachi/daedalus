# Hardware

テーマ別に整理したハードウェア分析。ソフトウェアに繋がる層を中心としつつ、物理的な工学システム（航空・自動車・電力など）まで対象を広げていく。

## 構成

- `cpu-architecture/` — CPU内部（ISA, キャッシュ, メモリ順序, 仮想記憶 …）
- `hw-sw-interface/` — HW・SW境界（MMIO, DMA, 割り込み, ABI …）
- `automotive/` — 自動車（車載ネットワーク, ECU …）
- `power-generation/` — 電力・発電（発電方式, 電力系統, 系統制御 …）
- `aerospace/` — 航空・宇宙（航空機の基礎工学, ロケット, SpaceX …）
- `gpu-accelerators/` — GPU・アクセラレータ（SIMT, メモリ階層/HBM, Tensor Core …）。
  `cpu-architecture/` の対照編で、`software/ml-systems/` と対で読む
- `hardware-security/` — ハードウェアセキュリティ（サイドチャネル, 一時的実行攻撃,
  Rowhammer, TEE, secure boot …）。`cpu-architecture/` の投機実行・キャッシュ機構と
  ソフトウェア側セキュリティの橋渡し

## 今後のテーマ候補

- 再突入と熱防護 (re-entry & TPS) — 空力加熱、耐熱タイル、アブレーション
- 半導体製造 (semiconductor manufacturing) — リソグラフィ (EUV)、プロセスノード、
  チップレットと先端パッケージング、歩留まり。すべての章の物理的前提
- 無線通信 (wireless & RF) — 変調・OFDM、5G NR、GNSS。`aerospace/` の航法と
  `software/networking/` の物理層側の土台
- ロボティクス (robotics) — アクチュエータ、制御ループ、キネマティクス。
  `automotive/`・`aerospace/` の制御工学の一般化
