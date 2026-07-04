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
  グラフィックス, セキュリティモデル）。`operating-systems/` の機構を実プロダクトの制約下で読む応用層。
- `ios/` — iOS プラットフォーム内部（XNU, launchd, Mach IPC/XPC, ARC, Jetsam,
  コード署名, Secure Enclave）。`android/` と同じ制約への対照的な解として相互対比する。
- `networking/` — ネットワーキング（TCP・輻輳制御, HTTP/2/3・QUIC）。
  `distributed-systems/` の故障モデルの発生源であり、NIC (`hardware/hw-sw-interface/`) の上位層。
- `specifications/` — 実システム・プロトコルの仕様分析（決済, 認証 …）。
- `virtualization/` — 仮想化とコンテナ（VT-x/EPT, KVM/QEMU, virtio, namespaces/cgroups,
  gVisor/Firecracker）。`hardware/hw-sw-interface/` と `operating-systems/` の間に挟まる
  もう一つの特権層。
- `ml-systems/` — ML システム（GPU 実行モデル, KV cache, 連続バッチング, 量子化,
  サービング）。`hardware/gpu-accelerators/` の真上の層で、同ディレクトリと対で読む。

## 今後のテーマ候補

- セキュリティ (security) — 適用暗号, メモリ安全性, サンドボックス（namespace, seccomp, WASM）
- コンパイラとランタイム (compilers & runtimes) — IR, 最適化, GC, JIT
- ブラウザ内部 (browser internals) — プロセスモデルとサンドボックス, レンダリング
  パイプライン, V8/JavaScriptCore。`android/`・`ios/` に続く第三のプラットフォーム分析
- 組み込み・リアルタイム OS (embedded & RTOS) — 決定論的スケジューリング,
  WCET, AUTOSAR。`hardware/automotive/` の ECU 側との合流点
