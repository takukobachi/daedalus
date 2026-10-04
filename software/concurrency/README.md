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

- 非同期ランタイムの実装 (async runtime internals) — event loop, work-stealing
- 構造化並行性 (structured concurrency) とキャンセル伝播
- アクターモデル (actor model) とメッセージパッシング
