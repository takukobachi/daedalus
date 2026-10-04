# ストレージデバイス物理 (Storage Device Physics)

ストレージデバイスの物理と内部制御。[`../hw-sw-interface/storage-nvm-interface.md`](../hw-sw-interface/storage-nvm-interface.md)
(NVMe・インタフェース層) の**下**の層にあたり、媒体の物理特性がインタフェース以上の
全階層 (block 層・ファイルシステム・データベース) の設計にどのような制約を課すかを扱う。

## ドキュメント

- [`nand-flash-ftl.md`](nand-flash-ftl.md) — NAND フラッシュと FTL：セルの物理と摩耗、
  読み・書き・消去の非対称、FTL (マッピング・GC・ウェアレベリング)、書込増幅と
  オーバープロビジョニング、SLC→QLC と 3D 積層、ZNS/FDP

## 今後のテーマ候補

- HDD の物理 (hard disk drives) — 磁気記録 (PMR/SMR/HAMR)、サーボ制御、ヘリウム充填。
  SMR の「追記しかできない」制約は NAND と同型の問題を別物理で再現する
- Emerging memory — PCM・MRAM・ReRAM、ストレージクラスメモリの興亡 (Optane の教訓)。
  [`../cpu/dram-memory-controllers.md`](../cpu/dram-memory-controllers.md)
  (揮発側) と `storage-nvm-interface.md` (永続側) の間を埋める層
