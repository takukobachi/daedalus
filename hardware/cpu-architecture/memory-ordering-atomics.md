# メモリ順序とアトミック (Memory Ordering & Atomics)

## 概要 (Overview)

シングルスレッドのプログラムは「書いた順に実行される」かのように振る舞う。しかしマルチコアでは、コンパイラとCPUが性能のために**メモリ操作を並べ替える (reorder)**。その並べ替えが許される範囲を規定するのが**メモリモデル (memory model)** であり、複数スレッドが共有データを正しくやり取りするための土台が**アトミック操作 (atomic operations)** と**メモリ順序 (memory ordering)** である。

本ドキュメントは、並べ替えがなぜ起きるか、ハードウェアのメモリモデルがどう異なるか、そして C++ の `memory_order` をどう使い分けるかを整理し、ロックフリー (lock-free) や同期処理の正しさを支える原理を示す。

## 方法論 (Methodology)

- 「逐次一貫性 (sequential consistency)」を理想モデルとして基準に置き、実機が何を緩める (relax) かを差分で説明する。
- ハードウェアモデル（x86 の TSO と ARM/POWER の弱順序 (weak ordering)）を比較する。
- C++11 メモリモデルの `memory_order` 列挙を、保証の強さ順に整理し、典型パターン（スピンロック、フラグ同期、二重チェックロック）へ対応づける。
- 正しさは「happens-before 関係が成立するか」で判定する。直感ではなくモデルに基づいて推論する。

## 発見事項 (Findings)

1. **並べ替えは2段で起きる。** コンパイラ最適化（命令の入れ替え・レジスタ保持・除去）と、CPU のアウトオブオーダ実行・ストアバッファ (store buffer) の両方。どちらもシングルスレッドの意味は保つが、他スレッドからの観測順は崩れる。
2. **ハードウェアのメモリモデルは一様でない。** x86 は比較的強い (TSO)、ARM/POWER は弱い。x86 で動いたコードが ARM で壊れる、という移植バグの主因。
3. **アトミック性 (atomicity) と順序 (ordering) は別概念。** 「破れない単一操作」であることと、「周囲の操作との順序保証」は独立に指定する必要がある。
4. **`memory_order` は性能と正しさのダイヤル。** `relaxed` は最速だが順序保証なし、`seq_cst` は最強だが最も高コスト。多くのケースは `acquire`/`release` で十分。
5. **データ競合 (data race) は未定義動作 (undefined behavior)。** 非アトミックな共有変数への並行アクセス（少なくとも一方が書き込み）は、C/C++ では UB であり「動いているように見える」だけのことがある。

## アーキテクチャ (Architecture)

### 逐次一貫性 (Sequential Consistency) — 理想

Lamport の定義: 全プロセッサの操作が、各プロセッサ内の順序を保ったまま、何らかの単一の全順序にインターリーブされたかのように見える。直感に最も近いが、実装すると毎回の操作で全コアの可視性を待つ必要があり遅い。実機はこれを緩める。

### ハードウェアのメモリモデル比較

| モデル | 代表 | 許す並べ替え | 備考 |
|---|---|---|---|
| Sequential Consistency | （理想） | なし | 実装は非現実的に遅い |
| TSO (Total Store Order) | x86-64, SPARC | Store→Load のみ | ストアバッファ由来。比較的強い |
| Weak / Relaxed | ARMv8, POWER | ほぼ全種 | 明示的バリアが必須 |

**x86 (TSO) の要点**: ストアバッファにより「自分のストアが他コアに見える前に、自分の後続ロードが先に進む」(Store→Load の並べ替え) のみ許す。Store→Store, Load→Load, Load→Store は保たれる。これが「x86 は寛容」と言われる理由。

**ARM/POWER の要点**: 上記を含むほぼ全ての並べ替えを許す。正しさは明示的なバリア命令 (`dmb` など) かアトミックの順序指定で担保する。

### リトマステスト (Litmus Tests) — モデル差を最小例で見る

メモリモデルの議論は、2スレッド・2変数の最小例（リトマステスト）で具体化できる。

**SB (Store Buffering)** — 相互排他 (Dekker) の骨格:

```
初期値: x = y = 0
  Thread 1:          Thread 2:
  x = 1;             y = 1;
  r1 = y;            r2 = x;
```

**MP (Message Passing)** — フラグによる受け渡しの骨格:

```
初期値: data = ready = 0
  Thread 1:          Thread 2:
  data = 42;         r1 = ready;
  ready = 1;         r2 = data;
```

| 結果 | SC | TSO (x86) | Weak (ARM/POWER) |
|---|---|---|---|
| SB: `r1 == 0 && r2 == 0` | 不可 | **可** | 可 |
| MP: `r1 == 1 && r2 == 0` | 不可 | 不可 | **可** |

- SB の「両方 0」は、両スレッドのストアがまだストアバッファ内にあるうちにロードが先行すると起きる。**x86 ですら起きる**ため、Dekker/Peterson 型のアルゴリズムはフルバリアなしでは成立しない。
- MP の「フラグは立っているのにデータが見えない」は TSO では起きないが、弱順序では起きる。**「x86 では動いたのに ARM で壊れる」移植バグの最小形**がこれであり、対策が本文の release/acquire ペアである。

このように「どの結果が観測されうるか」の集合こそがメモリモデルの定義であり、直感や実機テストの通過は保証にならない（herd7 などのモデル検査ツールで機械的に検証できる）。

### 並べ替えを止める道具: メモリバリア (Memory Barrier / Fence)

- **取得バリア (acquire)**: それ以降のメモリ操作が、バリアより前に追い越されない（後ろから前への移動を禁止）。
- **解放バリア (release)**: それ以前のメモリ操作が、バリアより後に追い越されない（前から後ろへの移動を禁止）。
- **フルバリア (full / seq_cst)**: 双方向に並べ替えを禁止。最も強く最もコスト高。

`release` で書いた内容は、対応する `acquire` で読んだスレッドから「それ以前の全書き込み込みで」可視になる。これが acquire/release ペアによる同期の核心。

## 技術詳細 (Technical Details)

### 1. アトミック操作の種類

- **ロード / ストア (load / store)**: 単一の読み／書きを破れない単位で行う。
- **Read-Modify-Write (RMW)**: 読み・更新・書きを不可分に行う。代表例:
  - `fetch_add` / `fetch_sub` / `fetch_or` …
  - **CAS (Compare-And-Swap)** / `compare_exchange`: 「期待値と一致すれば新値に差し替える」。ロックフリー構造の基本部品。
  - x86 では `lock` プレフィックス (`lock xadd`, `lock cmpxchg`) で実現。ARM では LL/SC (`ldxr`/`stxr`) ループ。

### 2. C++ の抽象と ISA 命令の対応

`memory_order` はコンパイル時に各 ISA の命令へ落ちる。対応を見ると「どのモデルで何がタダか」が分かる:

| 概念 | x86-64 (TSO) | ARMv8 | RISC-V (RVWMO) |
|---|---|---|---|
| acquire ロード | 通常の `mov`（TSO で暗黙に成立） | `ldar` | `lr.d.aq` / `fence r,rw` |
| release ストア | 通常の `mov`（同上） | `stlr` | `sc.d.rl` / `fence rw,w` |
| seq_cst ストア | `mov` + `mfence`（または `xchg`） | `stlr`(+`dmb`) | `fence` 併用 |
| フルバリア | `mfence` / `lock` 付き RMW | `dmb ish` | `fence rw,rw` |
| アトミック RMW | `lock xadd` / `lock cmpxchg` | `ldxr`/`stxr` ループ（v8.1 LSE: `ldadd` 等の単命令） | `amoadd.d` / `lr`+`sc`（`.aq`/`.rl` ビット付き） |

読み取れること:
- **x86 では acquire/release が実質無料**（通常のロード/ストアが既にその強さを持つ）。コストが乗るのは seq_cst ストアと RMW だけ。
- **ARM/RISC-V は順序が欲しい箇所にだけ明示的に払う**。acquire/release 専用命令（`ldar`/`stlr`、`.aq`/`.rl`）は、フルバリアより弱く安い——`memory_order` を正しく最小に指定する意味は、弱順序 ISA でこそ大きい。
- ARMv8 が acquire/release を**命令として**持つのは、C++11 メモリモデルの普及後に設計されたため。言語モデルが ISA 設計へ逆流した例である。

### 3. C++ memory_order の使い分け

保証の強い順に整理する。

```cpp
// 例示コード (illustrative)。リポジトリ内の実コードではない。
std::atomic<int> x{0};

// memory_order_relaxed: アトミック性のみ。順序保証なし。
//   用途: 統計カウンタなど「合計が合えばよい」もの。
x.fetch_add(1, std::memory_order_relaxed);

// release / acquire: 生産者-消費者の受け渡し。
//   release 側の書き込みが acquire 側に「まとめて」見える。
data = 42;                                   // (1) 通常の書き込み
ready.store(true, std::memory_order_release); // (2) これ以前(1)を後ろに動かさない
//   --- 別スレッド ---
if (ready.load(std::memory_order_acquire))    // (3) これ以降を前に動かさない
    assert(data == 42);                       // (4) (1)は必ず見える

// seq_cst (既定): 全 seq_cst 操作に単一の全順序。最強・最コスト。
x.store(1);            // 既定で memory_order_seq_cst
```

| memory_order | 保証 | 主な用途 |
|---|---|---|
| `relaxed` | アトミック性のみ | カウンタ、フラグの集計 |
| `consume` | データ依存の順序（実質非推奨） | ほぼ acquire で代替 |
| `acquire` | 後続操作の先行禁止 | ロック取得、フラグ読み |
| `release` | 先行操作の後行禁止 | ロック解放、フラグ書き |
| `acq_rel` | RMW で両方 | CAS ループ |
| `seq_cst` | 全順序 | 迷ったら（既定） |

**指針**: まず正しさ優先で `seq_cst`（既定）を使い、プロファイルで律速と分かった箇所だけ `acquire`/`release` や `relaxed` に落とす。最適化を先回りしない。

### 4. 典型パターン

#### スピンロック (spinlock)

```cpp
// 例示コード (illustrative)
std::atomic_flag lock = ATOMIC_FLAG_INIT;
void acquire() { while (lock.test_and_set(std::memory_order_acquire)) { /* spin */ } }
void release() { lock.clear(std::memory_order_release); }
```
取得は `acquire`、解放は `release`。これでクリティカルセクション内の操作が外へ漏れない。

#### 二重チェックロック (Double-Checked Locking)

かつて多くの言語で壊れていた古典的バグ。問題は「オブジェクト生成」と「ポインタ公開」の並べ替え。他スレッドが**未初期化のオブジェクトを初期化済みポインタ越しに観測**しうる。正しくは公開を `release`、読みを `acquire`（または `std::call_once`）で行う。

#### ロックフリー・スタック と ABA 問題

CAS ベースのロックフリー構造では **ABA 問題**が生じる。値が A→B→A と戻ると、CAS は「変化なし」と誤認する。対策はタグ付きポインタ (tagged pointer / version counter) や、ハザードポインタ (hazard pointer)・エポックベース回収などのメモリ再利用管理。

### 5. コヒーレンシとの関係

メモリ順序は、`cache-memory-hierarchy.md` のキャッシュコヒーレンシ (MESI) の**上**に構築される。コヒーレンシは「単一アドレスについて全コアが同じ値に収束する」ことを保証するが、**複数アドレス間の順序**は保証しない。その複数アドレス間の可視順を縛るのがメモリモデルとバリアである。

## セキュリティ考察 (Security Considerations)

- **データ競合 = 未定義動作**: 非アトミックな共有変数への並行アクセスは、コンパイラに「競合は起きない」と仮定した最適化を許す。結果として境界チェックの消去や値の引き裂き (torn read/write) が起き、メモリ安全性 (memory safety) を破壊しうる。セキュリティバグの温床。
- **引き裂き (tearing)**: アラインされていない／ワード幅を超える共有変数の非アトミック読み書きは、半分だけ更新された値を露出しうる。フラグやポインタの中途半端な状態が認可判定に使われると脆弱性になる。
- **TOCTOU との関係**: チェックと使用の間に別スレッドが状態を変える Time-of-check to time-of-use 競合は、アトミック化または適切なロックで window を閉じる必要がある。
- **投機実行との交差**: バリアは並べ替えを縛るが、投機実行のサイドチャネル（`cache-memory-hierarchy.md` 参照）を必ずしも塞がない。`lfence` 等は順序とともに投機抑止にも使われるが、目的を区別して用いる。

教訓: 「テストで動いた」は並行コードの正しさを保証しない。競合は確率的・プラットフォーム依存で顕在化する。正しさは ThreadSanitizer 等の動的検出と、モデルに基づく静的な推論の両輪で担保する。

## 結論 (Conclusions)

- メモリ順序は「コンパイラとCPUによる並べ替え」をどこまで許すかの契約であり、並行プログラムの正しさの根幹。
- ハードウェアモデルは一様でない（x86 は強く ARM は弱い）。移植性のためにモデルに依存せず、言語のメモリモデル (`memory_order`) で意図を明示する。
- アトミック性と順序は別物。`relaxed`〜`seq_cst` を、まず正しさ優先（既定 `seq_cst`）、計測後に必要箇所だけ緩める、という順で使う。
- データ競合は性能問題ではなく未定義動作＝セキュリティ問題である。共有可変状態は、アトミック化・ロック・不変化のいずれかで必ず律する。

### 次の接続先 (Related Topics)

- `cache-memory-hierarchy.md` — メモリ順序が立脚するキャッシュコヒーレンシ。
- `../hw-sw-interface/` — 割り込み・DMA と並行性、デバイスレジスタへの順序保証 (MMIO ordering)。

## 参考文献 (References)

**一次資料 (Primary — 論文・仕様)**

- Lamport, L. (1979). How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs. *IEEE Transactions on Computers*, C-28(9), 690–691. DOI: 10.1109/TC.1979.1675439 — 逐次一貫性の定義の原典。
- Sorin, D. J., Hill, M. D., & Wood, D. A. (2020). *A Primer on Memory Consistency and Cache Coherence* (2nd ed.). Morgan & Claypool. DOI: 10.2200/S00962ED2V01Y201910CAC049 — TSO・弱順序・コヒーレンシとの関係の標準的整理。
- *The RISC-V Instruction Set Manual, Volume I* — RVWMO（メモリモデル）章と A 拡張。 — [riscv.org/technical/specifications](https://riscv.org/technical/specifications/) — Accessed: 2026-07-02。
- Intel. *Intel 64 and IA-32 Architectures Software Developer's Manual*, Vol. 3A, Chapter 8 “Multiple-Processor Management”（Memory Ordering 節）. — [intel.com SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) — Accessed: 2026-07-02。x86 TSO の許容並べ替えの規範。
- ISO/IEC 14882 (C++), `std::memory_order`. — [cppreference: memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order) — Accessed: 2026-07-02。言語メモリモデルの各順序の定義。

**ツール**

- herd7 / litmus7 (diy toolsuite). — [diy.inria.fr](https://diy.inria.fr/) — Accessed: 2026-07-02。リトマステストをメモリモデル上で機械検証するツール。
