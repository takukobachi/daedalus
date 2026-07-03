# Operating Systems

オペレーティングシステム (operating system) 内部の分析。`hardware/hw-sw-interface/`
（ABI・システムコール・MMIO・ストレージ）の真上の層にあたり、ハードウェア資源を
プロセスへ多重化する機構を扱う。

## 構成

- `process-thread-scheduling.md` — プロセス・スレッドと CPU スケジューリング
  (context switch, preemption, CFS/EEVDF, リアルタイム, マルチコア負荷分散)
- `virtual-memory-paging.md` — OS の仮想記憶とページング：VMA と demand paging、COW、
  ページキャッシュと writeback、LRU 近似 (active/inactive, MGLRU)、swap、overcommit と OOM killer、THP
- `power-management.md` — 電力管理：cpuidle と眠りの深さの賭け、schedutil/EAS、
  サスペンドと wakeup source、無線 RRC テールとプッシュの経済、Android (Doze/バケット/
  WorkManager) と iOS (凍結/DAS) の統治層比較。`../android/`・`../ios/` を束ねる横串

## 今後のテーマ候補

- システムコール境界 (kernel/userspace boundary) — `hardware/hw-sw-interface/abi-syscalls.md` の続き
- ファイルシステム (filesystems) — VFS, journaling, ext4/xfs
- I/O スタック (I/O stack) — block 層, io_uring, ページキャッシュ writeback
