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
- `timekeeping/` — 時刻同期とクロック（発振器, 原子時計, GNSS 時刻, NTP/PTP …）。
  `software/distributed-systems/` の論理時計に対する「物理時計側」の土台

## 今後のテーマ候補

- 再突入と熱防護 (re-entry & TPS) — 空力加熱、耐熱タイル、アブレーション
- 半導体製造 (semiconductor manufacturing) — リソグラフィ (EUV)、プロセスノード、
  チップレットと先端パッケージング、歩留まり。すべての章の物理的前提
- 無線通信 (wireless & RF) — 変調・OFDM、5G NR、GNSS。`aerospace/` の航法と
  `software/networking/` の物理層側の土台
- ロボティクス (robotics) — アクチュエータ、制御ループ、キネマティクス。
  `automotive/`・`aerospace/` の制御工学の一般化
- データセンターインフラ (datacenter infrastructure) — 給電 (UPS/PDU)、冷却
  (空冷 → 液冷)、Clos トポロジ。`power-generation/` と計算機システムの合流点
- ストレージデバイス物理 (storage device physics) — NAND フラッシュの物理、FTL、
  書込増幅、DRAM セルと emerging memory。`hw-sw-interface/storage-nvm-interface.md` の下の層
- FPGA・再構成可能計算 (FPGA & reconfigurable computing) — LUT とルーティング、
  HLS、SmartNIC/DPU。`gpu-accelerators/` が描く専用化スペクトラムの続き
- 量子コンピューティング (quantum computing) — 量子ビットの物理、誤り訂正、
  極低温制御エレクトロニクス
- 鉄道信号と制御 (railway signaling) — 閉塞、ATC/ETCS/CBTC、フェールセーフ設計。
  `automotive/` の機能安全と対比する第三の輸送ドメイン
