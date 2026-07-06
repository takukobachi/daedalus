# Software

ソフトウェアの体系的な技術分析。OS・並行性・分散システム・データベースといった
システムソフトウェアから、ネットワーク、セキュリティ、実システムの仕様分析まで、
ソフトウェア全般を対象に深掘りする。低レイヤのテーマは `hardware/` が積み上げた層
（ISA → キャッシュ → 仮想記憶 → HW/SW 境界）と相互参照で繋ぎ、ハードウェアから
アプリケーションまでを一本のスタックとして見通せるようにする。

## 構成

### 実行基盤 (Execution Foundations)

ハードウェア資源をプロセス・実行環境へと変換する層。
`hardware/hw-sw-interface/` の真上から順に積み上げる。

- `virtualization/` — 仮想化とコンテナ（VT-x/EPT, KVM/QEMU, virtio, namespaces/cgroups,
  gVisor/Firecracker）。`hardware/hw-sw-interface/` と `operating-systems/` の間に挟まる
  もう一つの特権層
- `operating-systems/` — OS 内部（スケジューリング, 仮想記憶, syscall, FS, I/O）。
  `hardware/hw-sw-interface/` の真上の層
- `ebpf/` — eBPF（検証器, JIT, マップ, XDP/tracing/LSM フック）。カーネルを
  再コンパイルせずに拡張する「カーネル内の安全な実行基盤」。`operating-systems/` と
  `networking/`・セキュリティ応用をつなぐ
- `toolchain/` — ツールチェーン（ELF, リンカ, 動的リンク, ローダ, ASLR）。
  コンパイル後の成果物がプロセスになるまでを扱う層で、
  `hardware/hw-sw-interface/abi-syscalls.md` の ABI の実施者側
- `compilers-runtimes/` — コンパイラとランタイム（階層型 JIT, 投機と deopt, GC）。
  `toolchain/` (AOT 側) の対になる「実行しながら翻訳する」側で、
  `android/`・`ios/` のランタイム分析の一般論

### 並行・分散・データ (Concurrency, Distribution & Data)

1 台のマシン内の並行性から、ネットワークをまたぐ分散、永続データの管理へと
スケールを広げる層。

- `concurrency/` — 並行・並列（言語メモリモデル, 同期, lock-free, 非同期）。
  `memory-ordering-atomics.md` の上位層
- `networking/` — ネットワーキング（TCP・輻輳制御, HTTP/2/3・QUIC, TLS, DNS, カーネル
  ネットワークスタック）。`distributed-systems/` の故障モデルの発生源であり、
  NIC (`hardware/hw-sw-interface/`) の上位層
- `distributed-systems/` — 分散システム（一貫性モデル, 合意, 複製）。
  `cache-coherence.md` の coherence/consistency 議論のスケールアップ
- `databases/` — データベースとストレージエンジン（B+tree, LSM-tree, WAL,
  トランザクション, 分離レベル）。`hardware/hw-sw-interface/storage-nvm-interface.md` の上位層

### プラットフォーム分析 (Platform Analysis)

上記の一般機構が、実プロダクトの制約（バッテリー・メモリ・相互不信のアプリ群）の下で
どう再設計されるかを読む応用層。両者は同じ制約への対照的な解として相互対比する。

- `android/` — Android プラットフォーム内部（Zygote, Binder, ART, lmkd, Treble,
  グラフィックス, セキュリティモデル）
- `ios/` — iOS プラットフォーム内部（XNU, launchd, Mach IPC/XPC, ARC, Jetsam,
  コード署名, Secure Enclave）

### 応用システム・仕様 (Applied Systems & Specifications)

- `ml-systems/` — ML システム（GPU 実行モデル, KV cache, 連続バッチング, 量子化,
  サービング）。`hardware/gpu-accelerators/` の真上の層で、同ディレクトリと対で読む
- `specifications/` — 実システム・プロトコルの仕様分析（決済, 認証 …）。一般原理では
  なく、特定のプロダクト・標準仕様を一次資料から再構成する層

### 横断領域 (Cross-cutting)

- `security/` — ソフトウェアセキュリティ（メモリ安全性, exploit 緩和の系譜, サンドボックス,
  応用暗号 …）。`hardware/hardware-security/` のソフトウェア側の対であり、
  各領域の「セキュリティ考察」セクションを束ねる横串

## 今後のテーマ候補

新しいトップレベル領域を生みうる候補を挙げる。既存領域の深掘り候補は各ディレクトリの
README のロードマップに置き、ここには重複させない。

- ブラウザ内部 (browser internals) — プロセスモデルとサンドボックス, レンダリング
  パイプライン, V8/JavaScriptCore。`android/`・`ios/` に続く第三のプラットフォーム分析
- 組み込み・リアルタイム OS (embedded & RTOS) — 決定論的スケジューリング,
  WCET, AUTOSAR。`hardware/automotive/` の ECU 側との合流点
- メッセージングとストリーム処理 (messaging & stream processing) — 分散ログ,
  exactly-once セマンティクス, バックプレッシャ。`distributed-systems/`・`databases/` の応用層
- 検索と情報検索 (search & IR) — 転置インデックス, ランキング (BM25),
  近似最近傍探索 (ANN)。`databases/` と `ml-systems/`（ベクトル検索）の合流点
- Git 内部 (Git internals) — content-addressed storage, Merkle DAG, packfile と delta 圧縮。
  小さく閉じた題材で、分散システムの複製・整合性の具体例にもなる
