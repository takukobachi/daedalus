# Operating Systems

オペレーティングシステム (operating system) 内部の分析。`hardware/hw-sw-interface/`
（ABI・システムコール・MMIO・ストレージ）の真上の層にあたり、ハードウェア資源を
プロセスへ多重化する機構を扱う。

## 構成

- `process-thread-scheduling.md` — プロセス・スレッドと CPU スケジューリング
  (context switch, preemption, CFS/EEVDF, リアルタイム, マルチコア負荷分散)

## 今後のテーマ候補

- 仮想記憶とページング (virtual memory & paging) — page cache, demand paging, swap
  （`hardware/cpu-architecture/virtual-memory-mmu.md` の上位層）
- システムコール境界 (kernel/userspace boundary) — `hardware/hw-sw-interface/abi-syscalls.md` の続き
- ファイルシステム (filesystems) — VFS, journaling, ext4/xfs
- I/O スタック (I/O stack) — block 層, io_uring, ページキャッシュ writeback
