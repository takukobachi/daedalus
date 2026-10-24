# ハードウェア・ソフトウェア境界 (Hardware/Software Interface)

ソフトウェアとハードウェアが接する境界。CPUが周辺機器を操作する仕組みと、ユーザコードがマシンに到達する経路を扱う。

## ドキュメント

- [`boot-initialization.md`](boot-initialization.md) — ブート・初期化シーケンス (boot & initialization)
- [`mmio-dma-interrupts.md`](mmio-dma-interrupts.md) — デバイスとの対話: MMIO・DMA・割り込み (device I/O)
- [`bus-interconnect.md`](bus-interconnect.md) — バスと相互接続: PCIe・AMBA/AXI・CXL (bus & interconnect)
- [`abi-syscalls.md`](abi-syscalls.md) — ABI・呼び出し規約・システムコール (ABI & syscalls)
- [`clock-power-reset.md`](clock-power-reset.md) — クロック・電源・リセット管理 (clock, power & reset management)
- [`storage-nvm-interface.md`](storage-nvm-interface.md) — ストレージ・不揮発メモリインタフェース (storage & non-volatile memory interface)
- [`iommu-dma-isolation.md`](iommu-dma-isolation.md) — IOMMU と DMA 隔離: VT-d・AMD-Vi・SMMUv3 から機密 I/O まで (IOMMU & DMA isolation)

## 今後のテーマ候補

- CXL とメモリ拡張 (CXL & memory pooling) — [`bus-interconnect.md`](bus-interconnect.md)
  の CXL 概説の先。メモリ拡張・プーリング・ティアリングとキャッシュコヒーレントな接続
