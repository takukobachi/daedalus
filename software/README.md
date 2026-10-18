# Software

ソフトウェアの体系的な技術分析。OS・並行性・分散システム・データベースといった
システムソフトウェアから、ネットワーク、セキュリティ、実システムの仕様分析まで、
ソフトウェア全般を対象に詳しく分析する。低レイヤのテーマは `hardware/` が積み上げた層
（ISA → キャッシュ → 仮想記憶 → HW/SW 境界）と相互参照で繋ぎ、ハードウェアから
アプリケーションまでを一本のスタックとして見通せるようにする。

## 構成

### 実行基盤 (Execution Foundations)

ハードウェア資源をプロセス・実行環境へと変換する層。
`hardware/hw-sw-interface/` の真上から順に積み上げる。

- `virtualization/` — 仮想化とコンテナ（VT-x/EPT, KVM/QEMU, virtio, namespaces/cgroups,
  gVisor/Firecracker）。`hardware/hw-sw-interface/` と `operating-systems/` の間に挟まる
  もう一つの特権層
- `operating-systems/` — OS 内部（スケジューリング, 仮想記憶, syscall, FS, I/O, eBPF）。
  `hardware/hw-sw-interface/` の真上の層。カーネルを再コンパイルせずに拡張する eBPF もここで扱う
- `embedded/` — 組み込み・リアルタイム OS（RM/EDF と応答時間解析, WCET, 優先度逆転,
  RTOS カーネル設計, PREEMPT_RT, AUTOSAR）。「平均を速くする」`operating-systems/` に
  対して「最悪を保証する」側の実行基盤。`engineering/automotive/`（ECU）・
  `engineering/robotics/`（制御計算機）と software 側を束ねる橋
- `compilers-runtimes/` — コンパイラとランタイム（コンパイラ中盤・バックエンド (SSA, 最適化パイプライン,
  命令選択, レジスタ割り当て), ELF・リンカ・ローダ, 階層型 JIT, 投機と deopt,
  GC, 言語処理系の設計対比, WebAssembly）。コンパイル後の成果物がプロセスになるまで
  （`hardware/hw-sw-interface/abi-syscalls.md` の ABI の実施者側）と、「実行しながら翻訳する」
  側の両方を扱い、`android/`・`ios/` のランタイム分析の一般論になる

### 並行・分散・データ (Concurrency, Distribution & Data)

1 台のマシン内の並行性から、ネットワークをまたぐ分散、永続データの管理へと
スケールを広げる層。

- `concurrency/` — 並行・並列（言語メモリモデル, 同期, lock-free, 非同期）。
  `memory-ordering-atomics.md` の上位層
- `networking/` — ネットワーキング（LAN・スイッチング, IP・NAT・IPv6, メール配送と送信者認証, TCP・輻輳制御, HTTP/2/3・QUIC, TLS, DNS, カーネル
  ネットワークスタック, BGP・経路制御, VPC 型オーバーレイ, ロードバランシング）。
  `distributed-systems/` の故障モデルの発生源であり、
  NIC (`hardware/hw-sw-interface/`) の上位層
- `distributed-systems/` — 分散システム（一貫性モデル, 合意, 複製）。
  `cache-coherence.md` の coherence/consistency 議論のスケールアップ
- `databases/` — データベースとストレージエンジン（B+tree, LSM-tree, WAL,
  トランザクション, 分離レベル, クエリ実行・最適化, 分散データベース）。
  `hardware/hw-sw-interface/storage-nvm-interface.md` の上位層
- `caching/` — 分散キャッシュ（memcached, Redis, キャッシュ・アサイドと無効化の競合,
  スタンピードとホットキー, 置換方式, キャッシュ喪失によるメタステーブル障害）。
  `databases/` の前に置く値の写しの層で、`distributed-systems/` の一貫性・故障モデルの応用先
- `messaging/` — メッセージングとストリーム処理（分散ログ, 複製と耐久性,
  配信セマンティクス/exactly-once, ウィンドウとチェックポイント, バックプレッシャ）。
  `distributed-systems/`・`databases/` の機構をシステム間のデータフローへ応用する層
- `object-storage/` — オブジェクトストレージ（S3 型設計, メタデータ層とデータ層の分離,
  イレイジャーコーディング, CRUSH 型配置, 耐久性のモデル）。`distributed-systems/`・
  `databases/`・`hardware/storage/` の合流点
- `search/` — 検索と情報検索（転置インデックス, BM25, 動的枝刈り, 不変セグメント,
  HNSW/IVF-PQ による近似最近傍探索, ハイブリッド検索）。厳密一致ではなく関連度で
  top-k を返す第三のデータシステムで、`databases/`（LSM 同型のセグメント設計）と
  `ai/`（埋め込みの供給側）の合流点
- `compression/` — データ圧縮とコーデック（LZ77, Huffman/算術符号/ANS,
  DEFLATE/zstd, 予測→変換→量子化の映像・音声符号化, H.264/AV1/Opus）。
  `databases/`（ページ圧縮）・`observability/`（時系列符号化）・
  `networking/`（転送圧縮, 配信）が暗黙に依存する「縮める層」で、
  理論限界は `foundations/information-theory/` が与える

### プラットフォーム分析 (Platform Analysis)

上記の一般機構が、実プロダクトの制約（バッテリー・メモリ・相互不信のアプリ群）の下で
どう再設計されるかを読む応用層。各プラットフォームは同じ問題系への対照的な解として
相互対比する。

- `android/` — Android プラットフォーム内部（Zygote, Binder, ART, lmkd, Treble,
  グラフィックス, セキュリティモデル, メッセージループ, 入力, View システム, パッケージ管理,
  ブートと init, アプリ起動の全行程）
- `ios/` — iOS プラットフォーム内部（XNU, launchd, Mach IPC/XPC, ARC, Jetsam,
  コード署名, Secure Enclave）
- `browsers/` — ブラウザ内部（マルチプロセスとサイト分離, サンドボックス, Mojo,
  レンダリングパイプライン, V8）。署名もインストールもない任意コード実行という
  最も敵対的な制約下の第三のプラットフォーム
- `cloud/` — クラウド基盤（設計原則, コントロールプレーン, サーバーレス, コンテナ
  オーケストレーション, IAM・KMS・機密コンピューティング, ブロックストレージ・データベース・
  レイクハウス, ネットワークトポロジ・マルチリージョン, イベント駆動, IaC, 可観測性,
  スポット容量の経済, Kubernetes のネットワーク・ストレージ, デプロイの安全性,
  マルチテナント SaaS, オートスケーリング, コスト構造と FinOps）。仮想化・分散システム・ネットワーキングの一般機構が
  「マルチテナント＋従量課金」という制約下でどう製品化されるかを読む第四の
  プラットフォーム。AWS を主要な事例に、論文・標準・オープン実装と相互検証する

### 応用システム・仕様 (Applied Systems & Specifications)

- `ai/` — AI / LLM と ML システム。モデルを動かす下の層（GPU 実行モデル, KV cache,
  連続バッチング, 量子化, 学習基盤）は `hardware/accelerators/` の真上の層で、同ディレクトリと
  対で読む。モデルを部品として組み込む上の層（エージェントアーキテクチャ, エージェントループの工学,
  ツール使用, コンテキストエンジニアリング, MCP）も扱う。
  埋め込み検索の機構は `search/`、セキュリティ考察は `security/` と相互参照
- `apis/` — 実システムの API・プロトコルの仕様分析（決済, 認証 …）。一般原理では
  なく、特定のプロダクト・標準仕様を一次資料から再構成する層

### 横断領域 (Cross-cutting)

- `security/` — ソフトウェアセキュリティ（メモリ安全性, exploit 緩和の系譜, サンドボックス,
  応用暗号, LLM アプリケーションとプロンプトインジェクション, 境界防御とゼロトラスト …）。`hardware/security/` のソフトウェア側の対であり、
  各領域の「セキュリティ考察」セクションを束ねる横串
- `observability/` — 可観測性と性能工学（PMU/トップダウン解析, カーネルトレーシング,
  eBPF 集計, 分散トレーシング, メトリクス, SLO, 測定方法論）。`security/` に続く
  第二の横串で、各領域の「観測の入口」表を束ねて一般化する
- `formal-methods/` — 形式手法（時相論理, TLA+/SPIN によるモデル検査, Hoare 論理と
  演繹的検証 (seL4/CompCert), SMT・抽象解釈, eBPF 検証器・Rust 型システムという
  軽量形式手法）。`distributed-systems/`・`concurrency/`・`operating-systems/`（eBPF）に散在していた
  検証への言及を束ねる第三の横串で、「誤りを設計から排除する」側の品質保証

## 今後のテーマ候補

新しいトップレベル領域を生みうる候補を挙げる。既存領域の深掘り候補は各ディレクトリの
README のロードマップに置き、ここには重複させない。優先度は「既存ドキュメントとの
相互参照が何本張れるか（スタックの穴を埋めるか）」を基準に付けている。

### 優先度 低 (小さく閉じた題材)

- Git 内部 (Git internals) — content-addressed storage, Merkle DAG, packfile と delta 圧縮。
  小さく閉じた題材で、分散システムの複製・整合性の具体例にもなる
