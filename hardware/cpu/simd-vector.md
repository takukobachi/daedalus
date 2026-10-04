# SIMD・ベクトル演算 (SIMD / Vector Processing)

## 概要 (Overview)

スカラ演算 (scalar) は1命令が1組のデータを処理する。これに対し **SIMD (Single Instruction, Multiple Data)** は、1命令で複数のデータ要素 (element) を同時に処理する**データ並列 (data parallelism)** の方式である[^flynn][^hp]。配列・行列・信号・画像のように「同じ演算を多数の要素へ一様に適用する」処理で、命令フェッチ／デコードのオーバヘッドを償却し、スループットを数倍に引き上げる。

SIMD の利得は理屈の上では「レーン数 (lane) 倍」だが、実効性能はメモリ帯域・アライメント・制御フロー・コンパイラの自動ベクトル化能力に強く律速される。本ドキュメントは、固定幅 SIMD (x86 SSE/AVX/AVX-512) とベクタ長非依存 (scalable: ARM SVE, RISC-V RVV) の設計差を軸に、レジスタ／レーン構成、マスク実行、ベクトル化の限界、メモリアクセス様式、そして AMX/GPU との対比までを整理し、コンパイラとプログラマへの実務的示唆を導く。

## 方法論 (Methodology)

- スカラ実行を基準に置き、SIMD が何を**並列化**し何を**しない**かを差分で説明する。
- 命令セットを「固定幅 (fixed-width)」と「ベクタ長非依存 (vector-length agnostic, VLA)」の2系統に分け、レジスタ幅・レーン・マスクの扱いを比較する。
- 性能を決める要因を、計算 (compute)・メモリ (memory)・制御 (control) の3軸に分解し、自動ベクトル化が破綻する条件を列挙する。
- 数値は代表的な世代・実装の値であり、世代・マイクロアーキテクチャ・実装で変動する。絶対値ではなく**桁感 (order of magnitude)** を理解の基準とする。
- 抽象的な「N倍速い」ではなく、帯域律速かどうか・どのレーンが無駄になるか、という観測可能な性質で評価する。

## 発見事項 (Findings)

1. **SIMD はデータ並列であって、命令並列ではない。** 1命令が複数レーンに同じ演算を適用する。条件分岐のように「要素ごとに違うことをしたい」処理とは本質的に相性が悪く、そこをマスク (mask) で埋めるのが SIMD プログラミングの中心課題になる。
2. **固定幅とベクタ長非依存は思想が異なる。** x86 (SSE 128b → AVX 256b → AVX-512 512b) はレジスタ幅を世代ごとに拡張し、コードはその幅に固定される。ARM SVE / RISC-V RVV は幅を実行時に問い合わせるループとして書き、**同じバイナリが幅の異なる実装で動く**。前者は再コンパイル前提、後者は前方互換 (forward-compatible) を狙う[^intel-sdm][^sve][^riscv-v]。
3. **実効性能はしばしばメモリ帯域で頭打ちになる。** レーンを増やしても、データを供給するキャッシュ／DRAM 帯域が足りなければ理論ピークに届かない。SIMD は計算律速 (compute-bound) のカーネルでこそ効果を発揮し[^roofline]、メモリ律速 (memory-bound) では局所性の改善が先決になる。
4. **自動ベクトル化 (auto-vectorization) は脆い。** ループ間依存 (loop-carried dependence)・ポインタのエイリアシング (aliasing)・非定型な制御フロー・未知のアライメントのいずれか1つで、コンパイラはベクトル化を諦めるかスカラへ退化する。「速いはず」のコードが気づかないうちに遅くなっていることの主因。
5. **広い SIMD は周波数を下げることがある。** 一部の世代の AVX-512 は、重い 512 ビット演算で消費電力・発熱を抑えるためコア周波数を落とす (downclocking)。ベクトル化された区間は速くても、混在する非ベクトルコードまで遅くなり、ワークロード全体では損をしうる[^intel-opt][^downs]。

## アーキテクチャ (Architecture)

### スカラ vs SIMD のレーン構成

スカラ加算は1要素、SIMD 加算は1命令でレジスタ幅ぶんの要素を並列に処理する。32 ビット要素 (`f32`) の場合、レジスタ幅とレーン数の関係は次の通り。

```
スカラ (scalar): 1命令 = 1要素
    a0 ─(+)─ b0 ──> c0

SIMD 128b (例: SSE / NEON), f32 ×4 レーン:
   ┌──────┬──────┬──────┬──────┐
 A │  a3  │  a2  │  a1  │  a0  │
   ├──────┼──────┼──────┼──────┤
 B │  b3  │  b2  │  b1  │  b0  │
   └──┬───┴──┬───┴──┬───┴──┬───┘
     (+)    (+)    (+)    (+)      ← 1命令で4レーン同時
   ┌──┴───┬──┴───┬──┴───┬──┴───┐
 C │a3+b3 │a2+b2 │a1+b1 │a0+b0 │
   └──────┴──────┴──────┴──────┘

レジスタ幅とレーン数 (f32 = 32bit 要素の場合):
   128 bit → 4 レーン   (SSE / NEON)
   256 bit → 8 レーン   (AVX / AVX2)
   512 bit → 16 レーン  (AVX-512)
要素幅を変えればレーン数も変わる (512b なら f64×8, f32×16, i16×32, i8×64)。
```

レーン (lane) はレジスタを要素幅で区切った各スロットを指す。同じ 512 ビットでも、`f64` なら8レーン、`i8` なら64レーンになる。要素幅を狭くするほど並列度は上がるが、精度・値域は要素幅に制約される。

### 固定幅 (fixed-width) vs ベクタ長非依存 (vector-length agnostic)

```
固定幅 (x86): 幅がオペコードに焼き付く
   AVX2 のコードは「256bit / 8レーン」前提でループを書く。
   512bit 実装で動かしても、AVX2 命令は 256bit のまま。
   → 幅を活かすには AVX-512 で再コンパイルが必要。

ベクタ長非依存 (ARM SVE / RISC-V RVV): 幅を実行時に問い合わせる
   for (i = 0; i < N; i += VL) {        // VL = ハードウェアが返すベクタ長
       pred = whilelt(i, N);            // 末尾も述語で処理 (端数ループ不要)
       v = load(pred, &a[i]);
       ...
       store(pred, &c[i], v);
   }
   → 同一バイナリが 128b 実装でも 512b 実装でも正しく動く。
     ハードが広ければ反復回数が減るだけ。
```

固定幅は世代ごとに幅を倍々に増やしてきた (`128 → 256 → 512`)[^intel-sdm]。利点は単純で最適化しやすいこと、欠点は幅が変わるたびに ISA とコードが増殖し、再コンパイル／ディスパッチが要ること。ベクタ長非依存は「幅は実装が決め、コードは幅に依存しない」という反転した設計で (SVE は 128〜2048 ビットの 128 ビット刻みの実装幅を許し[^sve]、RVV は `vsetvli` が返す `vl` で反復幅を決める[^riscv-v])、前方互換と端数処理の簡潔さ (末尾要素を述語で自然に扱える) を得る代わりに、コンパイラ・人間とも書き方の発想転換を要する。

### マスク／述語実行 (Masking / Predication)

レーンごとに「実行する／しない」を1ビットで制御するのがマスク（x86 AVX-512 の `k` レジスタ）／述語 (predicate)（SVE の `p` レジスタ, RVV の `v0` マスク）である[^intel-sdm][^arm-arm][^riscv-v]。

```
条件 if (a[i] > 0) c[i] = a[i]; を SIMD 化:
  v   = [ -3 ,  5 , -1 ,  8 ]
  mask= [  0 ,  1 ,  0 ,  1 ]   ← (v > 0) をレーンごとに評価
  store(mask, &c, v)           ← mask=1 のレーンだけ書き込む
                                 mask=0 のレーンは元の c を保持
```

マスクにより、分岐を持つループを「全レーンを計算してから不要レーンを捨てる」形に変換できる。これが SIMD で制御フローを扱う基本手段であり、ベクタ長非依存 ISA では末尾の端数要素もマスクで処理するため、固定幅で必要だった「スカラの後始末ループ (epilogue)」が不要になる。

## 技術詳細 (Technical Details)

### 1. 自動ベクトル化 (auto-vectorization) とその限界

コンパイラ (GCC/Clang の `-O3 -march=...`, ICC/ICX 等) はループを解析し、安全と判断できれば SIMD 命令を生成する。だが次のいずれかで容易に失敗する。

```c
// 説明用の例示コード (illustrative)。リポジトリ内の実コードではない。

// (a) ループ間依存 (loop-carried dependence): 逐次性が並列化を阻む
for (int i = 1; i < N; i++) a[i] = a[i-1] + b[i];   // a[i] は a[i-1] に依存 → 不可

// (b) エイリアシング (aliasing): a と b が重なるか不明 → 安全側で諦める
void add(float *a, float *b, int n) {
    for (int i = 0; i < n; i++) a[i] += b[i];        // restrict が無いと保守的
}

// (c) 制御フロー: 反復ごとに早期脱出・複雑な分岐があると困難
for (int i = 0; i < N; i++) { if (stop(x[i])) break; ... }   // データ依存の break

// (d) アライメント不明: 先頭が境界に揃っていないと前置スカラ処理が要る
```

- **依存 (dependence)**: 反復間にデータ依存があるとレーン並列にできない。リダクション (総和など) は専用の縮約命令／`#pragma omp simd reduction` で扱う。
- **エイリアシング**: ポインタ同士が重ならない保証 (`restrict` / `__restrict`) を与えると、保守的なベクトル化抑止が外れる。
- **制御フロー**: データ依存の `break` や複雑な分岐はマスク化しづらい。マスクで均せる形に書き換えるのが定石。
- **アライメント**: 後述。先頭の未整列要素を剥がす「ピール (peeling)」が入ると、短いループでは利得が消える。

**指針**: 自動ベクトル化を当てにするなら、まずコンパイラの**ベクトル化レポート** (`-fopt-info-vec` / `-Rpass=loop-vectorize`) で「されたか／なぜされなかったか」を確認する。推測しない。

### 2. intrinsics と手動ベクトル化

自動ベクトル化が届かない／不十分な場合、**intrinsics**（組み込み関数）でレーン演算を直接書く[^intrinsics]。

```c
// 例示コード (illustrative)。x86 AVX intrinsics の概形。
#include <immintrin.h>
void add8(float *a, float *b, float *c) {       // 8要素 (256bit) を1反復で
    __m256 va = _mm256_loadu_ps(a);             // unaligned load
    __m256 vb = _mm256_loadu_ps(b);
    __m256 vc = _mm256_add_ps(va, vb);          // 8レーン同時加算
    _mm256_storeu_ps(c, vc);
}
```

intrinsics は制御を握れる反面、**ISA ごとに書き直し**（SSE/AVX/AVX-512/NEON/SVE/RVV）が必要で移植性に乏しい。緩和策として、ラッパーライブラリ（例: Highway, std::experimental::simd, xsimd, Eigen 等）で「ベクタ長非依存に書いて各 ISA へ落とす」抽象化を使う方法がある。手書きアセンブリは最後の手段。

### 3. メモリアライメント と gather/scatter

- **アライメント (alignment)**: ベクタロード／ストアは、ベクタ幅境界 (16/32/64B) に揃ったアドレスで最速になる。古い世代では未整列アクセスにペナルティがあり、整列版 (`load`)／未整列版 (`loadu`) が別命令だった。近年は未整列でもキャッシュライン ([`cache-memory-hierarchy.md`](cache-memory-hierarchy.md)) をまたがない限り差は小さいが[^intel-opt]、**ラインまたぎ (line split)** は依然コスト。データを 64B 境界に確保する (`alignas(64)`) と安定する。
- **gather / scatter**: 連続しないアドレスから要素を集める (gather) / 散らす (scatter) 命令。`c[i] = a[idx[i]]` のような間接アクセスをベクトル化できるが、各レーンが別ラインを引くため**実効帯域が低く**、しばしば擬似スカラ並みに遅い。可能なら SoA 化 ([`cache-memory-hierarchy.md`](cache-memory-hierarchy.md) の AoS/SoA 参照) や事前並べ替えで連続アクセスに変換するのが王道。

### 4. AVX-512 のダウンクロック問題 (downclocking)

一部の x86 世代では、幅の広い AVX2/AVX-512 命令、とりわけ重い浮動小数点・整数乗算を実行すると、電力・熱制約のためコア周波数を一時的に下げる (license-based frequency reduction)[^intel-opt]。独立した計測では、512 ビット命令の開始時に電圧調整のための実行スロットリング (約 8〜20 µs) と周波数切替に伴う停止 (約 10 µs)、最後の重い命令から約 680 µs 後の復帰が観測されている (Skylake-SP 系での測定値)[^downs]。

- ベクトル区間自体は速くても、**周波数低下は同コアの非ベクトルコードにも及ぶ**ため、ベクトル比率の低いワークロードでは全体として遅くなりうる。
- 遷移には遅延があり、短い AVX-512 区間を散発的に使うと「下がっては戻る」を繰り返して損をする。
- 実務的緩和: (1) ベクトル化を計算密度の高いカーネルに集中させ散発使用を避ける、(2) ケースによっては 256 ビット幅 (AVX2 / AVX-512 の 256b 運用) に留める、(3) 必ず実測で比較する。世代依存が大きいため「AVX-512 は常に速い／常に損」のどちらも誤り。

### 5. キャッシュ帯域とロード／ストア、メモリ順序

- **帯域律速 (bandwidth-bound)**: SIMD は1命令あたりの要求バイト数を増やすため、計算が軽いカーネル（要素あたり1〜2演算）はすぐにキャッシュ／DRAM 帯域に律速される。**演算強度 (arithmetic intensity, FLOP/byte)** が低いほどレーン拡張の効果は薄い。Roofline モデル[^roofline]でいう「メモリ天井」に当たっている場合、まず局所性とデータ配置 ([`cache-memory-hierarchy.md`](cache-memory-hierarchy.md)) を直すべきで、より広い SIMD では改善しない。
- **ロード／ストアとメモリ順序**: ベクタロード／ストアも、スカラと同じハードウェアのメモリモデル（x86 は TSO、ARM は弱順序）に従う。SIMD だからといって追加の順序保証は付かない。とくに **non-temporal store**（`_mm256_stream_ps` 等、キャッシュを汚さずに書く）は通常ストアより**弱い順序**を持ち、可視化前に `sfence` 等のバリアを要する[^intel-sdm]。並行コードでベクタストアと同期フラグを併用する場合の順序規律は [`memory-ordering-atomics.md`](memory-ordering-atomics.md) に従う (言語側の扱いは [`../../software/concurrency/memory-models-synchronization.md`](../../software/concurrency/memory-models-synchronization.md))。また 1本のベクタストアが**要素単位でアトミックとは限らない**（引き裂き／tearing の可能性）点も、共有データでは注意を要する。

### 6. AMX／行列拡張・GPU との対比

- **行列拡張 (matrix extensions)**: Intel **AMX (Advanced Matrix Extensions)** は「タイル (tile)」と呼ぶ2次元レジスタ上で行列積 (matmul) を1命令クラスで実行する。ARM の SME (Scalable Matrix Extension) も同系統[^arm-arm]。行列積そのものの数学は [`../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md`](../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md) を参照。SIMD が1次元ベクトルの要素並列なのに対し、これらは行列積に特化し、AI/深層学習の GEMM を高密度に処理する。汎用ベクトル演算ではなく「行列カーネル専用器」と捉えるのが正しい。
- **GPU との対比**: GPU は SIMD ではなく **SIMT (Single Instruction, Multiple Threads)** で、多数のスレッドが同じ命令を実行する。CPU SIMD が「数〜数十レーン × 低遅延 × 既存スカラコードとの密結合」なのに対し、GPU は「数千スレッド × 高スループット × 高遅延 × 別アドレス空間・転送コスト」。粒度の大きいデータ並列で転送を償却できるなら GPU、レイテンシ重視・小〜中規模・CPU と密に絡む処理なら SIMD、と棲み分ける。GPU 側の実行モデルは [`../accelerators/gpu-simt-architecture.md`](../accelerators/gpu-simt-architecture.md)、行列専用器の極限は [`../accelerators/tpu-systolic-arrays.md`](../accelerators/tpu-systolic-arrays.md) で扱う。

## セキュリティ考察 (Security Considerations)

- **定数時間実装 (constant-time) との関係**: 暗号処理では「実行時間・メモリアクセスが秘密値に依存しない」ことが要件になる。SIMD はデータ依存分岐をマスク化して**全レーンを計算してから捨てる**ため、分岐を消して定数時間性を助ける方向に使える。実際、定数時間暗号の高速実装は SIMD のマスク／ビットスライス (bitslicing)[^biham] を多用する。
- **gather/scatter のタイミング漏洩**: 一方で、秘密値で決まるインデックスを使う **gather/scatter は秘密依存のメモリアクセス**になり、キャッシュサイドチャネル ([`cache-memory-hierarchy.md`](cache-memory-hierarchy.md)、[`../security/side-channels-tee-secure-boot.md`](../security/side-channels-tee-secure-boot.md)) を開く。定数時間が要る箇所では、テーブル参照をベクタ選択 (blend/select) で全要素走査に置き換えるなど、秘密でアドレスを動かさない実装にする。
- **マスクされたレーンの扱い**: マスクで「書かない」と指定したレーンでも、実装によっては内部で読み・計算が走り、フォールトやタイミングに痕跡を残しうる。秘密データのマスク処理は ISA の保証 (suppress-fault 等) を確認する。gather 命令自体も投機実行の漏洩源になった例があり、Downfall (Gather Data Sampling) は gather が内部ベクタレジスタバッファの古いデータを一時的に転送することを悪用して、他のプロセスやコンテキストのデータを読み出した[^downfall]。
- **ダウンクロックによる間接漏洩**: 重い SIMD 実行に伴う周波数変動は、隣接スレッド／コアの実行時間に観測可能な差を生み、周波数・電力ベースのサイドチャネルの一因になりうる。

教訓: SIMD は「秘密依存の分岐を消す」点では定数時間化に味方するが、「秘密依存のアドレス（gather/scatter）」を導入すると逆にサイドチャネルを開く。性能のための機構が機密性とどちらに働くかは、秘密がデータ経路に乗るかアドレス経路に乗るかで決まる。

## 結論 (Conclusions)

- SIMD はデータ並列の機構であり、「同じ演算を多数の要素へ一様に適用する」処理で命令オーバヘッドを償却してスループットを上げる。条件分岐の多い不均一な処理は本質的に不得手で、そこをマスク／述語で均すのが要諦。
- 固定幅 (x86) とベクタ長非依存 (SVE/RVV) は設計思想が異なる。前者は再コンパイル前提で単純、後者は同一バイナリの前方互換と端数処理の簡潔さを狙う。移植性を重視するならベクタ長非依存の抽象化で書く。
- 実効性能は計算・メモリ・制御の3軸で決まる。レーンを増やす前に、帯域律速か (Roofline)・自動ベクトル化が成立しているか（レポートで確認）・アライメントと gather/scatter が足を引っ張っていないかを計測で押さえる。AVX-512 のダウンクロックのように、広ければ速いとは限らない。
- セキュリティ面では、SIMD は定数時間化を助ける一方、秘密依存の gather/scatter は新たなサイドチャネルになる。秘密を扱うコードでは、性能最適化がアドレス経路に秘密を乗せていないかを常に点検する。

### 次の接続先 (Related Topics)

- [`cache-memory-hierarchy.md`](cache-memory-hierarchy.md) — SIMD の実効性能を律速する帯域・局所性・データ配置 (AoS/SoA)・アライメント。
- [`memory-ordering-atomics.md`](memory-ordering-atomics.md) — ベクタロード／ストアと non-temporal store のメモリ順序、引き裂きと並行性。
- [`../accelerators/gpu-simt-architecture.md`](../accelerators/gpu-simt-architecture.md) — SIMT によるデータ並列とダイバージェンス。
- [`../../software/ai/ml-inference-systems.md`](../../software/ai/ml-inference-systems.md) — SIMD/行列拡張を使う推論カーネルのシステム側。

## 参考文献 (References)

**一次資料 (Primary — 仕様・論文・教科書)**

[^flynn]: Flynn, M. J. (1966). Very High-Speed Computing Systems. *Proceedings of the IEEE*, 54(12), 1901–1909. DOI: 10.1109/PROC.1966.5273 — SIMD 分類 (Flynn's taxonomy) の原典。

[^intel-sdm]: Intel. *Intel 64 and IA-32 Architectures Software Developer's Manual*, Vol. 1（SSE/AVX/AVX-512 のレジスタ幅・opmask `k` レジスタ・non-temporal store の弱い順序と `sfence`）. — [intel.com SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) — Accessed: 2026-10-04。

[^intrinsics]: Intel. *Intel Intrinsics Guide*. — [intel.com intrinsics guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) — Accessed: 2026-07-02。SSE/AVX/AVX-512 intrinsics の一次リファレンス。

[^riscv-v]: RISC-V International. *RISC-V "V" Vector Extension Specification* (v1.0) — `vsetvli`/`vl` によるベクタ長非依存設計と `v0` マスクの規範。 — [riscv/riscv-v-spec](https://github.com/riscv/riscv-v-spec) — Accessed: 2026-07-02。

[^arm-arm]: Arm. *Arm Architecture Reference Manual for A-profile architecture* (DDI 0487) — NEON / SVE・SVE2 / SME の規範。 — [developer.arm.com](https://developer.arm.com/documentation/ddi0487/latest) — Accessed: 2026-07-02。

[^sve]: Stephens, N., Biles, S., Boettcher, M., Eapen, J., Eyole, M., Gabrielli, G., Horsnell, M., Magklis, G., Martinez, A., Premillieu, N., Reid, A., Rico, A., & Walker, P. (2017). The ARM Scalable Vector Extension. *IEEE Micro*, 37(2), 26–39. DOI: 10.1109/MM.2017.35 — VLA 設計の動機、128〜2048 ビットの実装幅、述語レジスタ。

[^hp]: Hennessy, J. L., & Patterson, D. A. (2019). *Computer Architecture: A Quantitative Approach* (6th ed.). Morgan Kaufmann. — Ch. 4 "Data-Level Parallelism in Vector, SIMD, and GPU Architectures"。

[^roofline]: Williams, S., Waterman, A., & Patterson, D. (2009). Roofline: An Insightful Visual Performance Model for Multicore Architectures. *Communications of the ACM*, 52(4), 65–76. DOI: 10.1145/1498765.1498785 — 演算強度とメモリ天井の原典。

[^biham]: Biham, E. (1997). A Fast New DES Implementation in Software. *Fast Software Encryption (FSE 1997)*, LNCS 1267, 260–272. DOI: 10.1007/BFb0052352 — ビットスライス実装の原典。

[^downfall]: Moghimi, D. (2023). Downfall: Exploiting Speculative Data Gathering. *32nd USENIX Security Symposium*, 7179–7193. — [usenix.org PDF](https://www.usenix.org/system/files/usenixsecurity23-moghimi.pdf) — Accessed: 2026-10-04。gather 命令の投機的データ転送による漏洩 (Gather Data Sampling)。

**二次資料 (Secondary)**

[^intel-opt]: Intel. *Intel 64 and IA-32 Architectures Optimization Reference Manual*. — [intel.com](https://www.intel.com/content/www/us/en/developer/articles/technical/intel64-and-ia32-architectures-optimization.html) — Accessed: 2026-10-04。ベクトル化・アライメント・AVX-512 の周波数挙動を含むチューニング指針（ベンダ資料）。

[^downs]: [Gathering Intel on Intel AVX-512 Transitions](https://travisdowns.github.io/blog/2020/01/17/avxfreq1.html) — Downs, T. (2020) — Accessed: 2026-10-04。AVX/AVX-512 の電圧・周波数遷移の独立計測。
