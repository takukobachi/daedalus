# Operating Systems

オペレーティングシステム (operating system) 内部の分析。`hardware/hw-sw-interface/`
（ABI・システムコール・MMIO・ストレージ）の真上の層にあたり、ハードウェア資源を
プロセスへ多重化する機構を扱う。カーネルを再コンパイルせずに安全に拡張する
eBPF (extended Berkeley Packet Filter) も、カーネル機構の一部としてここで扱う。

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
- `ebpf-kernel-programmability.md` — eBPF：カーネル内プログラマビリティの基盤。
  カーネルモジュール vs ユーザ空間の永遠のトレードオフと第三の道、classic BPF
  (1993) からの系譜、検証器 (verifier) の静的解析と「検証器と戦う」開発体験、
  JIT と実行時ハードニング、プログラムタイプとフック分類 (XDP/tc/kprobe/LSM/
  sched_ext)、マップとデータプレーン、BTF/CO-RE による可搬性、エコシステム比較

## 今後のテーマ候補

- システムコール境界のカーネル側 (kernel/userspace boundary) — エントリ処理、
  `copy_from_user` とユーザポインタ検証、vDSO、seccomp による syscall フィルタ。
  [`../../hardware/hw-sw-interface/abi-syscalls.md`](../../hardware/hw-sw-interface/abi-syscalls.md)
  が扱う呼び出し規約・特権遷移（ハードウェア側）の続き
- XDP と高速データプレーン深掘り — Katran / Cilium の実装分析、DPDK・カーネル
  バイパスとの対比、`../networking/` のトランスポート層との合流点
- sched_ext によるスケジューラの eBPF 化 — `process-thread-scheduling.md`
  の CFS/EEVDF を「差し替え可能なポリシー」として読み直す
- Windows eBPF と移植性 — eBPF ISA の標準化 (IETF)、PREVAIL 検証器、
  「Linux 固有機構」から「汎用ランタイム仕様」への転換の行方
