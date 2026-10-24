# ネットワークハードウェア (Network Hardware)

NIC (network interface card) とスイッチ ASIC の内部分析。`../../software/networking/`（特に kernel-network-stack）の**物理側の対**であり、`../hw-sw-interface/`（MMIO・DMA・割り込み・PCIe）の部品がディスクリプタリングという一つの協定に組み上がる層。パケット単価（100GbE で 6.7 ns/パケット）という算術が、オフロード・マルチキュー・固定パイプライン・RDMA・DPU の設計すべてを規定する構造を読む。

## ドキュメント

- [`nic-switch-architecture.md`](nic-switch-architecture.md) — NIC とスイッチ (NIC & switch hardware)：パケット単価の算術、ディスクリプタリングと doorbell（lock-free リング協定）、オフロードの系譜（checksum/TSO/RSS と TOE の教訓）、RDMA/RoCEv2 とロスレスイーサネットの代償（PFC・DCQCN）、スイッチ ASIC のパイプライン（match-action・TCAM・共有バッファ・incast）、P4/RMT、SR-IOV/eSwitch と SmartNIC/DPU（AccelNet・Nitro）
- [`optical-interconnect.md`](optical-interconnect.md) — 光インターコネクト (optical interconnect)：銅の到達距離と光電変換の位置、トランシーバの部品と QSFP-DD/OSFP、NRZ→PAM4 とコヒーレント、SR/DR/FR/LR と IEEE 802.3df/dj、WDM、電力の内訳と pJ/bit、LPO と CPO (交換単位の取引)、シリコンフォトニクス、OCS (Apollo/Jupiter)、故障とリンクフラップの運用

## 今後のテーマ候補

- InfiniBand ファブリックの各論 (InfiniBand internals) — サブネットマネージャ、
  アダプティブルーティング・輻輳制御の詳細。SHARP と AI クラスタの全体像は
  [`../accelerators/interconnects-scaleout-training.md`](../accelerators/interconnects-scaleout-training.md)
  で扱ったため、残るのはファブリック運用の深掘り
- ストレージネットワーク (NVMe-oF) — NVMe over Fabrics (RDMA/TCP)、分離型ストレージ。
  `../hw-sw-interface/storage-nvm-interface.md` と RDMA の合流点
- ルーティングハードウェア (router hardware) — コアルータの深いバッファ、
  階層 QoS、FIB 圧縮。[`../../software/networking/ip-routing-bgp.md`](../../software/networking/ip-routing-bgp.md) の物理側
