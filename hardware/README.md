# ハードウェア (Hardware)

テーマ別に整理したハードウェア分析。ソフトウェアに繋がる層を中心としつつ、物理的な工学システム（航空・自動車・電力など）まで対象を広げていく。

## 構成

### 計算機システム (Computing Systems)

CPU の内部から SoC の外界までを、ソフトウェアに接続する層の順に積み上げる。

- `semiconductor-manufacturing/` — 半導体製造（リソグラフィ/EUV, プロセスノード,
  チップレット・先端パッケージング, 歩留まり）。すべての計算機システム分析の
  物理的前提となる最下層
- `cpu-architecture/` — CPU内部（ISA, キャッシュ, メモリ順序, 仮想記憶 …）
- `gpu-accelerators/` — GPU・アクセラレータ（SIMT, メモリ階層/HBM, Tensor Core …）。
  `cpu-architecture/` の対照編で、`software/ml-systems/` と対で読む
- `fpga-reconfigurable/` — FPGA・再構成可能計算（LUT と配線ファブリック, 設計フロー・
  タイミングクロージャ, HLS, 部分再構成, SmartNIC/DPU …）。`gpu-accelerators/` が描く
  専用化スペクトラム（CPU → GPU → ASIC）の間を埋める「書き換え可能なハードウェア」の層
- `serdes-signal-integrity/` — 高速シリアルリンクと SI（SerDes, PAM4, イコライゼーション,
  ジッタ, S パラメータ, COM …）。PCIe (`hw-sw-interface/`)・USB/ディスプレイ
  (`peripherals-connectivity/`)・Ethernet (`network-hardware/`) が論理層で共有する
  共通の物理層。理論は `foundations/physics/electromagnetism/` の伝送線路編が供給
- `hw-sw-interface/` — HW・SW境界（MMIO, DMA, 割り込み, ABI …）
- `peripherals-connectivity/` — 周辺デバイスと接続（入力デバイス/HID, オーディオ
  サブシステム, ディスプレイ, カメラ, USB, Bluetooth, Wi-Fi …）。`hw-sw-interface/` の
  一段上でSoCと外界をつなぐ層。`software/android/`・`software/ios/` の入力・オーディオ・
  表示スタックと対で読む
- `network-hardware/` — ネットワークハードウェア（NIC, ディスクリプタリング,
  オフロード, RDMA/RoCE, スイッチ ASIC, SmartNIC/DPU …）。
  `software/networking/`（特に kernel-network-stack）の物理側の対
- `wireless-rf/` — 無線通信・RF（リンクバジェット, シャノン容量, 変調/QAM, OFDM,
  MIMO, RF チェーン …）。広域・高速側の無線 PHY の共通理論で、Wi-Fi/Bluetooth
  (`peripherals-connectivity/`) の下の層。有線の `serdes-signal-integrity/` と
  「同じ敵 (ISI・ノイズ) と戦う無線側」として対で読む。理論は
  `foundations/physics/electromagnetism/` の電磁波編が供給
- `storage-devices/` — ストレージデバイス物理（NAND フラッシュ, FTL, 書込増幅 …）。
  `hw-sw-interface/storage-nvm-interface.md` の下の層で、`cpu-architecture/` の
  DRAM 分析と対になる永続側の媒体物理
- `hardware-security/` — ハードウェアセキュリティ（サイドチャネル, 一時的実行攻撃,
  Rowhammer, TEE, secure boot …）。`cpu-architecture/` の投機実行・キャッシュ機構と
  ソフトウェア側セキュリティの橋渡し
- `timekeeping/` — 時刻同期とクロック（発振器, 原子時計, GNSS 時刻, NTP/PTP …）。
  `software/distributed-systems/` の論理時計に対する「物理時計側」の基礎

### 物理工学ドメイン (Physical Engineering Domains)

計算機の外に広がる工学システム。物理プラントと制御ソフトウェアの接点を軸に読む。

- `automotive/` — 自動車（車載ネットワーク, ECU, E/E アーキテクチャ, 機能安全, EV …）
- `power-generation/` — 電力・発電（発電方式, 電力系統, 系統制御 …）
- `datacenter-infrastructure/` — データセンターインフラ（給電チェーン, UPS, 冷却・液冷,
  Clos トポロジ, PUE/WUE …）。`power-generation/` と計算機システムの合流点で、
  AI クラスタ (`gpu-accelerators/`) の電力・冷却制約の物理的実体
- `aerospace/` — 航空・宇宙（航空機の基礎工学, ロケット, SpaceX …）
- `robotics/` — ロボティクス（アクチュエータ, キネマティクス・動力学, 制御, 状態推定・SLAM,
  移動と操作 …）。
  `automotive/`・`aerospace/` に散在する制御工学の一般化であり、`foundations/` の
  線形代数・微分方程式の主要な応用先

## 今後のテーマ候補

新しいトップレベル領域を生みうる候補を挙げる。既存領域の深掘り候補は各ディレクトリの
README のロードマップに置き、ここには重複させない。優先度は「既存ドキュメントとの
相互参照が何本張れるか（スタックの穴を埋めるか）」を基準に付けている。

### 優先度 低 (将来の拡張)

- 量子コンピューティング (quantum computing) — 量子ビットの物理、誤り訂正、
  極低温制御エレクトロニクス
- 鉄道信号と制御 (railway signaling) — 閉塞、ATC/ETCS/CBTC、フェールセーフ設計。
  `automotive/` の機能安全と対比する第三の輸送ドメイン
- 医療機器 (medical devices) — IEC 62304/60601、規制下のソフトウェアライフサイクル、
  ペースメーカ・輸液ポンプの安全設計。automotive (ISO 26262)・railway に続く
  安全規格ドメインの第三例
