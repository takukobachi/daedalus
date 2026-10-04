# キャッシュとメモリ階層 (Cache & Memory Hierarchy)

## 概要 (Overview)

現代のCPUにおいて、演算性能とメモリアクセス速度の差（**メモリウォール / memory wall**）は数百倍に達する。この差を埋めるのがキャッシュ (cache) を中心としたメモリ階層 (memory hierarchy) であり、ソフトウェアの実効性能を決定づける最大の要因の一つである。

本ドキュメントは、メモリ階層の構造・動作原理を整理し、それがソフトウェア設計（データ構造・アルゴリズム・並行処理）にどう跳ね返るかを明らかにする。

## 方法論 (Methodology)

- 階層の各段（レジスタ → L1/L2/L3 キャッシュ → 主記憶 → ストレージ）を、容量・遅延・帯域・管理単位の4軸で比較する。
- キャッシュの内部構造（ライン、連想度、置換、書き込み方式）を分解し、各設計判断がヒット率に与える影響を示す。
- 観測可能な指標（遅延サイクル数、ミス率）に基づき、ソフトウェア最適化の指針を導く。
- 数値は一般的な x86-64 サーバ／デスクトップ世代の代表値であり、世代・実装で変動する。絶対値ではなく**桁感 (order of magnitude)** を理解の基準とする。

## 発見事項 (Findings)

1. **遅延は階層ごとに桁で増える。** L1 ヒットは約 4 サイクルだが、主記憶アクセスは 200〜300 サイクル超に達する。1回のキャッシュミスは数十〜数百命令分の機会損失に相当する。
2. **転送の最小単位はキャッシュライン (cache line) であり、通常 64 バイト。** 1バイト読んでも64バイトが運ばれる。したがって性能は「何バイト触るか」ではなく「どのラインに触るか」で決まる。
3. **データ局所性 (locality) が支配的。** 時間局所性 (temporal) と空間局所性 (spatial) を満たすアクセスパターンは、同じ計算量でも桁違いに速い。
4. **並行処理では false sharing が隠れたボトルネックになる。** 別スレッドが同一ラインの別変数を書くだけで、コヒーレンシ (coherency) のために性能が崩壊する。
5. **キャッシュは性能機構であると同時にサイドチャネル (side channel) でもある。** アクセスの有無がタイミングに漏れ、Spectre / Meltdown 系攻撃の土台となる。

## アーキテクチャ (Architecture)

### メモリ階層の全体像

```
        ┌──────────────┐  容量小・高速・高価
        │  Registers   │  ~数百B,  <1 cycle
        ├──────────────┤
        │  L1 I/D Cache│  ~32–64KB/core, ~4 cycles
        ├──────────────┤
        │   L2 Cache   │  ~256KB–1MB/core, ~12 cycles
        ├──────────────┤
        │ L3 Cache(LLC)│  ~数十MB/共有, ~40 cycles
        ├──────────────┤
        │  Main Memory │  ~数十GB(DRAM), ~200–300 cycles
        ├──────────────┤
        │   Storage    │  ~TB(SSD/NVMe), ~10^4–10^6 cycles
        └──────────────┘  容量大・低速・安価
```

代表的な遅延の桁感（参考値）を示す。

| 階層 | 遅延 (cycles) | 遅延 (実時間, 3GHz換算) |
|---|---|---|
| L1 cache | ~4 | ~1.3 ns |
| L2 cache | ~12 | ~4 ns |
| L3 cache | ~40 | ~13 ns |
| 主記憶 (DRAM) | ~200–300 | ~70–100 ns |
| NVMe SSD | ~10^5 | ~数十 µs |

### キャッシュの内部構造

- **ライン (line)**: 転送・管理の最小単位。一般に 64 バイト。アドレスは `[tag | index | offset]` に分解される。
  - `offset` (下位6ビット, 64B時): ライン内の位置
  - `index`: どのセット (set) に入るか
  - `tag`: 同一セット内でラインを識別
- **連想度 (associativity)**: 1つのセットに何本のラインを置けるか。
  - ダイレクトマップ (direct-mapped): 1-way。衝突しやすいが高速・単純。
  - フルアソシアティブ (fully associative): どこにでも置ける。衝突最小だが高コスト。
  - セットアソシアティブ (N-way set associative): 実装の主流（例: L1 8-way, L3 16-way）。
- **置換方式 (replacement policy)**: セットが満杯のとき何を捨てるか。LRU 近似 (pseudo-LRU) が一般的。
- **書き込み方式 (write policy)**:
  - ライトバック (write-back): キャッシュにのみ書き、追い出し時に主記憶へ。dirty ビットで管理。帯域効率が高く主流。
  - ライトスルー (write-through): 即座に主記憶へも書く。単純だが帯域を消費。
  - 書き込み割当 (write-allocate / no-write-allocate): 書き込みミス時にラインを読み込むか否か。

### キャッシュコヒーレンシ (Cache Coherency)

マルチコアでは各コアが同じラインのコピーを持ちうる。一貫性を保つプロトコルが **MESI** とその拡張 (MOESI, MESIF)。各ラインは状態を持つ。

- **M (Modified)**: 自コアが変更済み、他コアは無効。主記憶と不一致。
- **E (Exclusive)**: 自コアのみ保持、未変更。主記憶と一致。
- **S (Shared)**: 複数コアが保持しうる、未変更。
- **I (Invalid)**: 無効。

あるコアが書き込むと、他コアの該当ラインは Invalidate される。この無効化トラフィックが、後述の false sharing の正体である。

## 技術詳細 (Technical Details)

### 1. データ配置: AoS vs SoA

**Array of Structures (AoS)** と **Structure of Arrays (SoA)** はキャッシュ効率を大きく変える。

```c
// 説明用の例示コード (illustrative)。リポジトリ内の実コードではない。
// AoS: 1要素に全フィールドが隣接
struct ParticleAoS { float x, y, z; float vx, vy, vz; int id; };
struct ParticleAoS particles[N];

// SoA: フィールドごとに連続配列
struct ParticlesSoA { float x[N], y[N], z[N]; float vx[N], vy[N], vz[N]; int id[N]; };
```

`x` 座標のみを全要素走査する場合を考える。
- AoS では1要素ごとに `y,z,vx,...` も同じラインに載り、不要データでラインが埋まる → 空間局所性が悪化。
- SoA では `x[]` が連続し、1ライン(64B)に16個の `float` が載る → ミスが 1/16 近くに削減。

**指針**: 「一緒にアクセスするデータを物理的に隣接させる」。走査軸に合わせて AoS / SoA を選ぶ。

### 2. アクセスパターンと局所性

```c
// 例示コード (illustrative)
// 良い例: 行優先 (row-major) 配列を行方向に走査 → 空間局所性◎
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        sum += a[i][j];

// 悪い例: 列方向に走査 → 各アクセスが別ライン → ミス多発
for (int j = 0; j < N; j++)
    for (int i = 0; i < N; i++)
        sum += a[i][j];
```

C/C++ は行優先 (row-major) のため前者が正しい。後者は N が大きいとミス率が跳ね上がり、計算量 O(N²) は同じでも実行時間は数倍〜十数倍になりうる。**計算量複雑度 (time complexity) が同じでも定数項はメモリ階層が決める。**

### 3. False Sharing（偽共有）

```c
// 例示コード (illustrative)
// 悪い例: 2つのカウンタが同一キャッシュライン(64B)に同居
struct Counters { long a; long b; };  // a,b は隣接 → 同一ライン
// スレッド1が a++ 、スレッド2が b++ するだけで、
// 互いのラインを Invalidate し合い、コヒーレンシ・トラフィックが爆発する。

// 改善: ラインをまたぐようパディング (padding) で分離
struct CountersPadded {
    alignas(64) long a;   // 各カウンタを別ラインへ
    alignas(64) long b;
};
```

論理的には独立した変数でも、物理的に同一ラインだと「共有」とみなされ無効化が連鎖する。これが **false sharing**。対策はキャッシュライン境界へのアラインメント／パディング。

### 4. プリフェッチ (Prefetching)

ハードウェアプリフェッチャはストライド (stride) 一定の規則的アクセスを検知し、先読みする。リンクリスト走査のような**ポインタ追跡 (pointer chasing)** は予測できず、毎回ミスしやすい。連続配列がリンク構造より速い主因の一つ。必要に応じソフトウェアプリフェッチ命令 (`prefetcht0` 等) を用いるが、誤用はキャッシュ汚染を招く。

### 5. 測定方法 (Measurement)

- **`perf stat`** (Linux): `cache-misses`, `LLC-load-misses`, `L1-dcache-load-misses` 等のハードウェアカウンタを取得。
- **Cachegrind** (Valgrind): キャッシュミスをシミュレートし、行単位で内訳を提示。
- 指標は絶対数より**ミス率 (misses / accesses)** とミスの**局在 (どのループか)** を見る。推測ではなく計測で律速箇所を特定する。

## セキュリティ考察 (Security Considerations)

キャッシュは「アクセスしたか否か」がタイミング差として観測できるため、強力なサイドチャネルになる。

- **Flush+Reload**: 攻撃者が共有ラインを `clflush` で追い出し、被害者の実行後に再アクセス時間を測る。速ければ被害者が触った（=ヒット）と判定し、秘密依存のアクセスを推定する。
- **Prime+Probe**: 攻撃者がセットを自分のデータで埋め (prime)、被害者実行後に各ラインの遅延を測る (probe)。追い出されていれば被害者がそのセットを使ったと分かる。
- **Spectre / Meltdown**: 投機実行 (speculative execution) が秘密値をキャッシュ状態に残し、上記の手法で読み出す。アーキテクチャ上は「なかったこと」になる投機結果が、マイクロアーキテクチャ状態（キャッシュ）に痕跡を残す点が本質。
- **緩和策 (mitigations)**: 定数時間実装 (constant-time)、秘密依存のメモリアクセス・分岐の排除、`lfence` による投機抑止、ページテーブル分離 (KPTI)、ハードウェア側の投機制御。いずれも性能と引き換えになる。

教訓: キャッシュ最適化（データ依存のアクセス）と機密性（アクセスを秘密に依存させない）は時に対立する。暗号処理など秘密を扱うコードでは、性能より定数時間性を優先する。

## 結論 (Conclusions)

- メモリ階層は「容量と速度のトレードオフ」を多段で解決する仕組みであり、実効性能の支配要因である。
- ソフトウェア側の鉄則は **局所性を高め、ラインを意識し、false sharing を避ける** の3点に集約される。
- 計算量複雑度が同じでも、データ配置とアクセス順序で実行時間は桁で変わる。最適化は計測 (`perf`, Cachegrind) で律速箇所を特定してから行う。
- キャッシュは性能機構であると同時に情報漏洩経路でもある。秘密を扱うコードでは、性能最適化がそのままサイドチャネルになりうることを前提に設計する。

### 次の接続先 (Related Topics)

- `memory-ordering-atomics.md` — コヒーレンシの上に成り立つメモリ順序と並行制御。
- `../hw-sw-interface/` — DMA とキャッシュ整合 (cache coherence) の境界問題。

## 参考文献 (References)

**一次資料 (Primary — 教科書・論文)**

- Hennessy, J. L., & Patterson, D. A. (2019). *Computer Architecture: A Quantitative Approach* (6th ed.). Morgan Kaufmann. — メモリ階層・キャッシュ構成 (Ch. 2, Appendix B) の標準教科書。
- Yarom, Y., & Falkner, K. (2014). FLUSH+RELOAD: A High Resolution, Low Noise, L3 Cache Side-Channel Attack. *USENIX Security Symposium*. — [PDF](https://eprint.iacr.org/2013/448.pdf) — Accessed: 2026-07-02。キャッシュサイドチャネルの代表的一次論文。

**二次資料 (Secondary)**

- Drepper, U. (2007). *What Every Programmer Should Know About Memory*. — [PDF](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) — Accessed: 2026-07-02。階層・局所性・測定のソフトウェア視点の定番解説。
- Intel. *Intel 64 and IA-32 Architectures Optimization Reference Manual*. — データ配置・プリフェッチの実装別チューニング指針（intel.com で配布）。
