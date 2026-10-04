# ハードウェア (Hardware)

テーマ別に整理した計算機ハードウェアの分析。CPU の内部から SoC の外界までを、ソフトウェアに接続する層を中心に扱う。航空・自動車・電力などの物理的な工学システムは `../engineering/` で扱う。

## 構成

ソフトウェアに接続する層の順に積み上げる。

- `semiconductor-manufacturing/` — 半導体製造（リソグラフィ/EUV, プロセスノード,
  チップレット・先端パッケージング, 歩留まり）。すべての計算機システム分析の
  物理的前提となる最下層
- `cpu/` — CPU内部（ISA, キャッシュ, メモリ順序, 仮想記憶 …）
- `accelerators/` — GPU・アクセラレータ（SIMT, メモリ階層/HBM, Tensor Core …）。
  `cpu/` の対照編で、`software/ai/` と対で読む
- `fpga/` — FPGA・再構成可能計算（LUT と配線ファブリック, 設計フロー・
  タイミングクロージャ, HLS, 部分再構成, SmartNIC/DPU …）。`accelerators/` が描く
  専用化スペクトラム（CPU → GPU → ASIC）の間を埋める「書き換え可能なハードウェア」の層
- `signal-integrity/` — 高速シリアルリンクと SI（SerDes, PAM4, イコライゼーション,
  ジッタ, S パラメータ, COM …）。PCIe (`hw-sw-interface/`)・USB/ディスプレイ
  (`peripherals/`)・Ethernet (`networking/`) が論理層で共有する
  共通の物理層。理論は `foundations/physics/electromagnetism/` の伝送線路編が供給
- `hw-sw-interface/` — HW・SW境界（MMIO, DMA, 割り込み, ABI …）
- `peripherals/` — 周辺デバイスと接続（入力デバイス/HID, オーディオ
  サブシステム, ディスプレイ, カメラ, USB, Bluetooth, Wi-Fi …）。`hw-sw-interface/` の
  一段上でSoCと外界をつなぐ層。`software/android/`・`software/ios/` の入力・オーディオ・
  表示スタックと対で読む
- `networking/` — ネットワークハードウェア（NIC, ディスクリプタリング,
  オフロード, RDMA/RoCE, スイッチ ASIC, SmartNIC/DPU …）。
  `software/networking/`（特に kernel-network-stack）の物理側の対
- `wireless/` — 無線通信・RF（リンクバジェット, シャノン容量, 変調/QAM, OFDM,
  MIMO, RF チェーン …）。広域・高速側の無線 PHY の共通理論で、Wi-Fi/Bluetooth
  (`peripherals/`) の下の層。有線の `signal-integrity/` と
  「同じ敵 (ISI・ノイズ) と戦う無線側」として対で読む。理論は
  `foundations/physics/electromagnetism/` の電磁波編が供給
- `storage/` — ストレージデバイス物理（NAND フラッシュ, FTL, 書込増幅 …）。
  `hw-sw-interface/storage-nvm-interface.md` の下の層で、`cpu/` の
  DRAM 分析と対になる永続側の媒体物理
- `security/` — ハードウェアセキュリティ（サイドチャネル, 一時的実行攻撃,
  Rowhammer, TEE, secure boot …）。`cpu/` の投機実行・キャッシュ機構と
  ソフトウェア側セキュリティの橋渡し
- `timekeeping/` — 時刻同期とクロック（発振器, 原子時計, GNSS 時刻, NTP/PTP …）。
  `software/distributed-systems/` の論理時計に対する「物理時計側」の基礎

## 今後のテーマ候補

新しいトップレベル領域を生みうる候補を挙げる。既存領域の深掘り候補は各ディレクトリの
README のロードマップに置き、ここには重複させない。優先度は「既存ドキュメントとの
相互参照が何本張れるか（スタックの穴を埋めるか）」を基準に付けている。

### 優先度 低 (将来の拡張)

- 量子コンピューティング (quantum computing) — 量子ビットの物理、誤り訂正、
  極低温制御エレクトロニクス
