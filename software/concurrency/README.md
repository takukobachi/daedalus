# Concurrency

並行・並列プログラミング (concurrency & parallelism) の分析。
`hardware/cpu/memory-ordering-atomics.md` が扱うハードウェアの
メモリ順序を、言語・ランタイムレベルの抽象へと積み上げる層。

## 構成

- `memory-models-synchronization.md` — 言語メモリモデル (C/C++11, Java JMM)、
  happens-before、アトミック/CAS、ロックと futex、lock-free 入門、非同期ランタイム
- `lock-free-data-structures.md` — lock-free データ構造とメモリ回収：Treiber stack、
  Michael–Scott queue、ABA 問題、hazard pointer / epoch / RCU、wait-free、競合とコヒーレンスの実コスト

## 今後のテーマ候補

- 非同期ランタイムの実装は [`../compilers-runtimes/async-executors-asyncio-tokio.md`](../compilers-runtimes/async-executors-asyncio-tokio.md)
  で扱い済み（event loop, work-stealing）。残る候補は以下
- 構造化並行性 (structured concurrency) とキャンセル伝播 — asyncio TaskGroup・Tokio JoinSet と
  キャンセルの基本は同文書 §4 で扱い済み。残るのは言語横断の比較 (Kotlin coroutines、
  Swift の TaskGroup、Java の StructuredTaskScope) と、キャンセルを安全に伝える設計論
- アクターモデル (actor model) とメッセージパッシング
