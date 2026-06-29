# Concurrency

並行・並列プログラミング (concurrency & parallelism) の分析。
`hardware/cpu-architecture/memory-ordering-atomics.md` が扱うハードウェアの
メモリ順序を、言語・ランタイムレベルの抽象へと積み上げる層。

## 構成

- `memory-models-and-synchronization.md` — 言語メモリモデル (C/C++11, Java JMM)、
  happens-before、アトミック/CAS、ロックと futex、lock-free 入門、非同期ランタイム

## 今後のテーマ候補

- lock-free / wait-free データ構造 — ABA, hazard pointer, RCU の詳細
- 非同期ランタイムの実装 (async runtime internals) — event loop, work-stealing
- 構造化並行性 (structured concurrency) とキャンセル伝播
- アクターモデル (actor model) とメッセージパッシング
