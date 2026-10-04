# コンパイラとランタイム (Compilers & Runtimes)

言語処理系の実行時側の分析。ソースコードが「速く走る」までの機構として、JIT コンパイル・
ガベージコレクション・ランタイムの投機を扱う。[`../toolchain/`](../toolchain/)
(AOT コンパイル後の成果物がプロセスになるまで) の対になる「実行しながら翻訳する」側で、
[`../android/art-runtime-internals.md`](../android/art-runtime-internals.md) (ART) や
ブラウザの JS エンジンはこの層の実プロダクトである。

`language-implementations-cpython-go-rust.md` を起点に、標準ライブラリ内部・
非同期実行器へ掘り下げる CPython / Go / Rust 比較の三部作を含む。

## 構成

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

- コンパイラ中盤 (compiler middle-end) — IR・SSA 形式、主要最適化 (インライン化,
  ループ変換, ベクトル化)、LLVM のパス構造。`../../hardware/cpu-architecture/simd-vector.md`
  の自動ベクトル化の供給側
