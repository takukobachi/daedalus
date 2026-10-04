# CPUアーキテクチャ (CPU Architecture)

ソフトウェアの視点から見たCPU内部の分析。アーキテクチャ上の選択が性能や並行性にどう影響するかを扱う。

## ドキュメント

- [`cache-memory-hierarchy.md`](cache-memory-hierarchy.md) — キャッシュとメモリ階層 (cache & memory hierarchy)
- [`pipeline-superscalar-ooo.md`](pipeline-superscalar-ooo.md) — パイプライン・スーパースカラ・アウトオブオーダ実行 (pipeline, superscalar & out-of-order execution)
- [`memory-ordering-atomics.md`](memory-ordering-atomics.md) — メモリ順序とアトミック (memory ordering & atomics)
- [`cache-coherence.md`](cache-coherence.md) — マルチコア・キャッシュコヒーレンス (multicore cache coherence)
- [`virtual-memory-mmu.md`](virtual-memory-mmu.md) — 仮想記憶・MMU・TLB (virtual memory)
- [`interrupts-exceptions-traps.md`](interrupts-exceptions-traps.md) — 割り込み・例外・トラップ (interrupts, exceptions & traps)
- [`isa-riscv-intro.md`](isa-riscv-intro.md) — 命令セットアーキテクチャ入門 (ISA primer, RISC-V)
- [`branch-prediction-speculation.md`](branch-prediction-speculation.md) — 分岐予測と投機実行 (branch prediction & speculative execution)
- [`simd-vector.md`](simd-vector.md) — SIMD・ベクトル演算 (SIMD / vector processing)
- [`dram-memory-controllers.md`](dram-memory-controllers.md) — DRAM とメモリコントローラ (DRAM & memory controllers)

## 今後のテーマ候補

- ヘテロジニアスマルチコア (heterogeneous multicore) — big.LITTLE / P・E コア、
  非対称コア間の共有キャッシュとマイグレーション、OS スケジューラとの接点
- 性能計測と PMU (performance monitoring) — ハードウェア性能カウンタ、`perf` が読む
  マイクロアーキテクチャ事象、トップダウン解析 (top-down microarchitecture analysis)
- NUMA とマルチソケット (NUMA & multi-socket) — UPI / Infinity Fabric、メモリ
  アフィニティ、[`cache-coherence.md`](cache-coherence.md)・
  [`dram-memory-controllers.md`](dram-memory-controllers.md) のスケールアップ側の続き

