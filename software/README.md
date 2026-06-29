# Software

ソフトウェアの体系的な技術分析。`hardware/` が積み上げた層（ISA → キャッシュ →
仮想記憶 → HW/SW 境界）の真上から、システムソフトウェア、分散システム、
アプリケーション仕様までを一本のスタックとして繋いでいく。

> 注: 実装コード（`prototypes/` 実験コード、`tools/` 開発ツール、`libraries/`
> 再利用ライブラリ）は本ディレクトリの分析ドキュメントとは別管理であり、ここでは
> 扱わない。

## 構成

- `operating-systems/` — OS 内部（スケジューリング, 仮想記憶, syscall, FS, I/O）。
  `hardware/hw-sw-interface/` の真上の層。
- `distributed-systems/` — 分散システム（一貫性モデル, 合意, 複製）。
  `cache-coherence.md` の coherence/consistency 議論のスケールアップ。
- `concurrency/` — 並行・並列（言語メモリモデル, 同期, lock-free, 非同期）。
  `memory-ordering-atomics.md` の上位層。
- `specifications/` — 実システム・プロトコルの仕様分析（決済, 認証 …）。

## 今後のテーマ候補

- データベースとストレージエンジン (databases & storage engines) — B+tree vs
  LSM-tree, WAL, MVCC, 分離レベル（`hardware/hw-sw-interface/storage-nvm-interface.md` の上位層）
- ネットワーキングとプロトコル (networking) — TCP/IP, TLS, HTTP/2/3・QUIC
- セキュリティ (security) — 適用暗号, メモリ安全性, サンドボックス（namespace, seccomp, WASM）
- コンパイラとランタイム (compilers & runtimes) — IR, 最適化, GC, JIT
