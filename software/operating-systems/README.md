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
- `filesystems.md` — ファイルシステム：VFS と名前解決 (dcache/RCU-walk)、オンディスク構造
  (inode/extent)、クラッシュ一貫性の系譜 (fsck → ジャーナリング → CoW)、ext4/jbd2 の
  モード、btrfs/ZFS、fsync の契約と fsyncgate
- `block-io-stack.md` — ブロック I/O スタック：bio と blk-mq (単一キューの破綻と再設計)、
  I/O スケジューラの現在地、io_uring (リング・SQPOLL・完了型 I/O)、O_DIRECT、
  レイテンシの観測 (biolatency, PSI)

## 今後のテーマ候補

- システムコール境界 (kernel/userspace boundary) — `hardware/hw-sw-interface/abi-syscalls.md` の続き
