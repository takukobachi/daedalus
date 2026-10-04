# コンパイラとランタイム (Compilers & Runtimes)

言語処理系の分析。コンパイル後の成果物（オブジェクトファイル）が実行中のプロセスになるまでの
**リンク (linking) とロード (loading)** と、ソースコードが「速く走る」までの実行時側の機構
（JIT コンパイル・ガベージコレクション・ランタイムの投機）の両方を扱う。前者は
`../../hardware/hw-sw-interface/abi-syscalls.md` が定めた ABI という「見えない契約」を
**誰が守らせているか**への答えであり、後者は「実行しながら翻訳する」側である。
[`../android/art-runtime-internals.md`](../android/art-runtime-internals.md) (ART) や
ブラウザの JS エンジンは後者の実プロダクトである。

AOT コンパイラ本体 (IR 上の中盤最適化と、機械語を作るバックエンド) は
`compiler-middle-end-ir-optimization.md` → `compiler-backend-code-generation.md` →
`linkers-loaders-elf.md` の順に、ソースからオブジェクトファイル・プロセスまでを一続きで追える。

`language-implementations-cpython-go-rust.md` を起点に、標準ライブラリ内部・
非同期実行器へ掘り下げる CPython / Go / Rust 比較の三部作を含む。

## 構成

- `compiler-middle-end-ir-optimization.md` — コンパイラ中盤:「事実を証明し、
  許された書き換えだけを行う」作業としての最適化。LLVM IR の設計と MLIR、
  SSA 構築 (Cytron の支配辺境法 / Braun の on-the-fly 法)、SCCP・GVN・LICM・
  インライン化のコストモデル、Loop/SLP 自動ベクトル化、別名解析、新パスマネージャと
  `-O2`/`-O3` パイプライン、UB と poison/undef/freeze、PGO・LTO・ThinLTO
- `compiler-backend-code-generation.md` — コンパイラバックエンド: 理想機械を有限の
  実機へ写す NP 困難な写像の連鎖。命令選択 (木被覆・BURS、SelectionDAG vs
  GlobalISel)、レジスタ割り当て (Chaitin/Briggs の彩色、線形走査、LLVM greedy の
  追い出しと分割)、リストスケジューリングとマシンモデル、ソフトウェアパイプライニング、
  呼び出し規約の低水準化、MC 層とフィックスアップ・relaxation
- `linkers-loaders-elf.md` — リンカとローダ：シンボル解決と再配置という2つの
  基本操作、ELF の二重ビュー（セクション vs セグメント）、静的リンクの規則と
  失敗モード、PLT/GOT と遅延束縛、シンボル介入と LD_PRELOAD、ld.so の
  ブートストラップ、IFUNC と xz 事件、ASLR/PIE/RELRO、LTO とリンカの進化
  (BFD ld → gold → lld → mold)、dyld / Android linker / PE との対比
- `managed-runtimes-gc-jit.md` — マネージドランタイム：階層型 JIT (インタプリタ →
  最適化コンパイラ)、hidden class とインラインキャッシュ、投機と deoptimization、
  GC (世代仮説・並行化・バリア)、参照カウントとの対比、JIT/GC のセキュリティ
- `language-implementations-cpython-go-rust.md` — 言語処理系の設計対比:「意味論の
  保証をいつ払うか」の軸で CPython (バイトコード・GIL・特殊化・free-threading)、
  Go (GMP スケジューラ・伸長スタック・並行 GC)、Rust (所有権・単相化・unsafe 境界)
  を 8 軸で比較。並行安全性の 3 戦略 (ロック・検出器・型)、async の三答
  (「関数の色」問題・stackful/stackless)、エラー処理の課金方式、
  起動とデプロイの形の対比
- `stdlib-internals-cpython-go-rust.md` — 標準ライブラリのデータ構造内部:
  dict / map / HashMap の実装対比 (compact dict, Go の漸進疎開と Swiss map,
  hashbrown/SwissTable の SIMD 照合)、反復順序という意味論の分岐、
  SipHash と HashDoS、文字列表現 (「i 番目の文字」への三つの答え)
- `async-executors-asyncio-tokio.md` — 非同期実行器の内部:asyncio の
  イベントループ (コールバック機械 + Task の二層) と Tokio のワークスティーリング
  実行器 (poll/Waker 契約, LIFO スロット, 協調的バジェット)、キャンセルの三哲学
  (注入・破棄・合意)、構造化並行性への収斂、協調世界の可用性攻撃面
- `webassembly-portable-binary-sandbox.md` — WebAssembly:「検証をロード時に、
  最終コンパイルをホストに」という第四の実行モデル。サンドボックスから逆算された
  ISA (構造化制御フロー・線形メモリ・guard page)、ホスト保護とモジュール内部の
  非対称、WasmGC とコンポーネントモデル、eBPF との検証設計対比

## 今後のテーマ候補

- ビルドシステムと再現可能ビルド (build systems & reproducible builds) —
  依存グラフとパッケージ依存解決 (go modules の MVS, cargo の semver, pip)、
  キャッシュと分散ビルド、ビット単位で同一な成果物が
  サプライチェーン検証に持つ意味
- デバッグ情報とアンワインド (debug info & unwinding) — DWARF、
  スタックアンワインド (CFI)、シンボリケーション。プロファイラ・例外処理・
  クラッシュレポートの共通基盤
