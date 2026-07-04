# Virtualization & Containers

仮想化 (virtualization) とコンテナ (container) の分析。1台の物理計算機を、
相互に信頼しない複数の OS／ワークロードで安全に多重化する機構を扱う。
`hardware/hw-sw-interface/` が定めた HW/SW 境界と `operating-systems/` の間に
挟まる「もう一つの特権層」であり、CPU (VT-x/EPT)・メモリ・I/O (IOMMU, virtio) の
各資源に「トラップと変換」の間接層をもう一段積む層である。

## 構成

- `hardware-virtualization-containers.md` — 仮想化とコンテナ：Popek-Goldberg 基準と
  x86 の罪、VT-x/AMD-V (VMCS, VM exit)、シャドウページテーブル vs EPT/NPT、
  virtio と SR-IOV/IOMMU、KVM/QEMU 分業と Firecracker、namespaces/cgroups v2、
  gVisor/Kata と隔離–密度–互換性のトライアングル

## 今後のテーマ候補

- ライブマイグレーション (live migration) — pre-copy / post-copy、dirty page tracking、
  パススルーデバイスとの両立問題
- vDPA / SR-IOV 深掘り — データプレーンのハードウェア化と、仮想化の可搬性を
  どう両立させるか
- Kata / microVM エコシステムと WASM サンドボックス — プロセスより軽く VM より
  堅い隔離を狙う第三世代サンドボックスの比較
