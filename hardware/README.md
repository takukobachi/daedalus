# ハードウェア (Hardware)

テーマ別に整理したハードウェア分析。ソフトウェアに繋がる層を中心としつつ、物理的な工学システム（航空・自動車・電力など）まで対象を広げていく。

## 構成

### 計算機システム (Computing Systems)

CPU の内部から SoC の外界までを、ソフトウェアに接続する層の順に積み上げる。

- `cpu-architecture/` — CPU内部（ISA, キャッシュ, メモリ順序, 仮想記憶 …）
- `gpu-accelerators/` — GPU・アクセラレータ（SIMT, メモリ階層/HBM, Tensor Core …）。
  `cpu-architecture/` の対照編で、`software/ml-systems/` と対で読む
- `hw-sw-interface/` — HW・SW境界（MMIO, DMA, 割り込み, ABI …）
- `peripherals-connectivity/` — 周辺デバイスと接続（入力デバイス/HID, オーディオ
  サブシステム, ディスプレイ, USB, Bluetooth …）。`hw-sw-interface/` の一段上でSoCと
  外界をつなぐ層。`software/android/`・`software/ios/` の入力・オーディオ・表示
  スタックと対で読む
- `storage-devices/` — ストレージデバイス物理（NAND フラッシュ, FTL, 書込増幅 …）。
  `hw-sw-interface/storage-nvm-interface.md` の下の層で、`cpu-architecture/` の
  DRAM 分析と対になる永続側の媒体物理
- `hardware-security/` — ハードウェアセキュリティ（サイドチャネル, 一時的実行攻撃,
  Rowhammer, TEE, secure boot …）。`cpu-architecture/` の投機実行・キャッシュ機構と
  ソフトウェア側セキュリティの橋渡し
- `timekeeping/` — 時刻同期とクロック（発振器, 原子時計, GNSS 時刻, NTP/PTP …）。
  `software/distributed-systems/` の論理時計に対する「物理時計側」の土台

### 物理工学ドメイン (Physical Engineering Domains)

計算機の外に広がる工学システム。物理プラントと制御ソフトウェアの接点を軸に読む。

- `automotive/` — 自動車（車載ネットワーク, ECU, 機能安全, EV …）
- `power-generation/` — 電力・発電（発電方式, 電力系統, 系統制御 …）
- `aerospace/` — 航空・宇宙（航空機の基礎工学, ロケット, SpaceX …）

## 今後のテーマ候補

新しいトップレベル領域を生みうる候補を挙げる。既存領域の深掘り候補は各ディレクトリの
README のロードマップに置き、ここには重複させない。

- 半導体製造 (semiconductor manufacturing) — リソグラフィ (EUV)、プロセスノード、
  チップレットと先端パッケージング、歩留まり。すべての章の物理的前提
- 無線通信 (wireless & RF) — 変調・OFDM、5G NR、GNSS など広域側。近距離 (Bluetooth)
  は `peripherals-connectivity/` でカバー済み。`aerospace/` の航法と
  `software/networking/` の物理層側の土台
- ロボティクス (robotics) — アクチュエータ、制御ループ、キネマティクス。
  `automotive/`・`aerospace/` の制御工学の一般化
- NIC とスイッチ (NIC & switch hardware) — DMA リングとディスクリプタ、
  チェックサム/TSO などのオフロード、スイッチ ASIC とバッファリング。
  `software/networking/`（特に kernel-network-stack）の物理側の対
- データセンターインフラ (datacenter infrastructure) — 給電 (UPS/PDU)、冷却
  (空冷 → 液冷)、Clos トポロジ。`power-generation/` と計算機システムの合流点
- FPGA・再構成可能計算 (FPGA & reconfigurable computing) — LUT とルーティング、
  HLS、SmartNIC/DPU。`gpu-accelerators/` が描く専用化スペクトラムの続き
- 量子コンピューティング (quantum computing) — 量子ビットの物理、誤り訂正、
  極低温制御エレクトロニクス
- 鉄道信号と制御 (railway signaling) — 閉塞、ATC/ETCS/CBTC、フェールセーフ設計。
  `automotive/` の機能安全と対比する第三の輸送ドメイン
