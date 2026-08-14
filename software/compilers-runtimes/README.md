# コンパイラとランタイム (Compilers & Runtimes)

言語処理系の実行時側の分析。ソースコードが「速く走る」までの機構——JIT コンパイル・
ガベージコレクション・ランタイムの投機——を扱う。[`../toolchain/`](../toolchain/)
(AOT コンパイル後の成果物がプロセスになるまで) の対になる「実行しながら翻訳する」側で、
[`../android/art-runtime-internals.md`](../android/art-runtime-internals.md) (ART) や
ブラウザの JS エンジンはこの層の実プロダクトである。

## 構成

- `managed-runtimes-gc-jit.md` — マネージドランタイム：階層型 JIT (インタプリタ →
  最適化コンパイラ)、hidden class とインラインキャッシュ、投機と deoptimization、
  GC (世代仮説・並行化・バリア)、参照カウントとの対比、JIT/GC のセキュリティ
- `language-implementations-cpython-go-rust.md` — 言語処理系の設計対比:「意味論の
  保証をいつ払うか」の軸で CPython (バイトコード・GIL・特殊化・free-threading)、
  Go (GMP スケジューラ・伸長スタック・並行 GC)、Rust (所有権・単相化・unsafe 境界)
  を 7 軸で比較。並行安全性の 3 戦略 (ロック・検出器・型)、async の三答
  (「関数の色」問題・stackful/stackless)、エラー処理の課金方式の対比

## 今後のテーマ候補

- コンパイラ中盤 (compiler middle-end) — IR・SSA 形式、主要最適化 (インライン化,
  ループ変換, ベクトル化)、LLVM のパス構造。`../../hardware/cpu-architecture/simd-vector.md`
  の自動ベクトル化の供給側
- WebAssembly — 検証可能な移植可能バイナリ、サンドボックス、コンポーネントモデル。
  `../ebpf/` (カーネル内の検証付き実行) とのサンドボックス設計対比
