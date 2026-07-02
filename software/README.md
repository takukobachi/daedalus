# Software

ソフトウェアの体系的な技術分析。OS・並行性・分散システム・データベースといった
システムソフトウェアから、ネットワーク、セキュリティ、実システムの仕様分析まで、
ソフトウェア全般を対象に深掘りする。低レイヤのテーマは `hardware/` が積み上げた層
（ISA → キャッシュ → 仮想記憶 → HW/SW 境界）と相互参照で繋ぎ、ハードウェアから
アプリケーションまでを一本のスタックとして見通せるようにする。

## 構成

- `operating-systems/` — OS 内部（スケジューリング, 仮想記憶, syscall, FS, I/O）。
  `hardware/hw-sw-interface/` の真上の層。
- `distributed-systems/` — 分散システム（一貫性モデル, 合意, 複製）。
  `cache-coherence.md` の coherence/consistency 議論のスケールアップ。
- `concurrency/` — 並行・並列（言語メモリモデル, 同期, lock-free, 非同期）。
  `memory-ordering-atomics.md` の上位層。
- `databases/` — データベースとストレージエンジン（B+tree, LSM-tree, WAL,
  トランザクション, 分離レベル）。`hardware/hw-sw-interface/storage-nvm-interface.md` の上位層。
- `android/` — Android プラットフォーム内部（Zygote, Binder, ART, lmkd, Treble,
  Verified Boot）。`operating-systems/` の機構を実プロダクトの制約下で読む応用層。
- `networking/` — ネットワーキング（TCP・輻輳制御, HTTP/2/3・QUIC）。
  `distributed-systems/` の故障モデルの発生源であり、NIC (`hardware/hw-sw-interface/`) の上位層。
- `specifications/` — 実システム・プロトコルの仕様分析（決済, 認証 …）。

## 今後のテーマ候補

- セキュリティ (security) — 適用暗号, メモリ安全性, サンドボックス（namespace, seccomp, WASM）
- コンパイラとランタイム (compilers & runtimes) — IR, 最適化, GC, JIT
