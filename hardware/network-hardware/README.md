# ネットワークハードウェア (Network Hardware)

NIC (network interface card) とスイッチ ASIC の内部分析。`../../software/networking/`（特に kernel-network-stack）の**物理側の対**であり、`../hw-sw-interface/`（MMIO・DMA・割り込み・PCIe）の部品がディスクリプタリングという一つの協定に組み上がる層。パケット単価（100GbE で 6.7 ns/パケット）という算術がすべての設計——オフロード・マルチキュー・固定パイプライン・RDMA・DPU——を強制する構造を読む。

## ドキュメント

- [`nic-switch-architecture.md`](nic-switch-architecture.md) — NIC とスイッチ (NIC & switch hardware)：パケット単価の算術、ディスクリプタリングと doorbell（lock-free リング協定）、オフロードの系譜（checksum/TSO/RSS と TOE の教訓）、RDMA/RoCEv2 とロスレスイーサネットの代償（PFC・DCQCN）、スイッチ ASIC のパイプライン（match-action・TCAM・共有バッファ・incast）、P4/RMT、SR-IOV/eSwitch と SmartNIC/DPU（AccelNet・Nitro）

## 今後のテーマ候補

- 光インターコネクト (optical interconnect) — 光トランシーバ (SFP/OSFP)、DWDM、
  co-packaged optics、リニアドライブ光学。SerDes の先にある「銅の限界」の突破口で、
  AI クラスタの帯域スケーリングの物理的本体
- InfiniBand ファブリックの各論 (InfiniBand internals) — サブネットマネージャ、
  アダプティブルーティング・輻輳制御の詳細。SHARP と AI クラスタの全体像は
  [`../gpu-accelerators/interconnects-scaleout-training.md`](../gpu-accelerators/interconnects-scaleout-training.md)
  で扱ったため、残るのはファブリック運用の深掘り
- ストレージネットワーク (NVMe-oF) — NVMe over Fabrics (RDMA/TCP)、分離型ストレージ。
  `../hw-sw-interface/storage-nvm-interface.md` と RDMA の合流点
- ルーティングハードウェア (router hardware) — コアルータの深いバッファ、
  階層 QoS、FIB 圧縮。`../../software/networking/` の BGP 候補の物理側
