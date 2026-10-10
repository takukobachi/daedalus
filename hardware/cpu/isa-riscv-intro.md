# 命令セットアーキテクチャ入門 (ISA Primer with RISC-V)

## 概要 (Overview)

**命令セットアーキテクチャ (ISA: Instruction Set Architecture)** は、ハードウェアとソフトウェアの間に引かれた契約である。CPU が理解する命令・レジスタ・メモリモデル・特権レベルを定義し、その下の実装（マイクロアーキテクチャ）と、その上のソフトウェア（コンパイラ・OS）を分離する。プログラマが書いた高級言語は、最終的に機械語に変換される。その機械語の語彙を定めるのが ISA である。

本ドキュメントは、オープンで仕様が読める **RISC-V** を題材に ISA の基本構造を整理し、x86 / ARM との設計思想の違い、そしてコンパイラ出力（アセンブリ）の読み方を、ソフトエンジニアの視点で示す。

## 方法論 (Methodology)

- ISA を「レジスタ・命令形式・アドレッシング・呼び出し規約・特権」の構成要素に分解する。
- RISC（縮小命令セット）と CISC（複雑命令セット）の対比を軸に、各設計判断の根拠を示す。
- RISC-V のモジュラ構成（基本 + 拡張）を通じ、ISA が「固定」ではなく「設計可能な対象」であることを示す。
- 簡単な C 関数のコンパイル結果を例に、アセンブリの読み解き方を具体化する。

## 発見事項 (Findings)

1. **ISA は抽象境界である。** 同じ ISA でも実装は無数にありうる（インオーダ／アウトオブオーダ、低消費電力／高性能）。ソフトは ISA だけを見ればよく、実装の差は ISA が吸収する。
2. **RISC-V は「小さな基本＋拡張」のモジュラ設計。** 必要な機能だけを組み合わせる。組込みからスパコンまで同じ枠組みで覆える。
3. **RISC は「単純な命令を多数」、CISC は「複雑な命令を少数」。** 現代の x86 は内部で RISC 的なマイクロオペに分解しており、境界は曖昧になっている。
4. **レジスタ数と呼び出し規約がコードの形を決める。** どの引数がどのレジスタに乗るかは ISA/ABI が規定し、関数呼び出しの実体を形作る。
5. **アセンブリは読めると強力な診断手段。** コンパイラ最適化の結果、性能特性、未定義動作の影響が、生成コードに現れる。

## アーキテクチャ (Architecture)

### ISA の構成要素

| 要素 | 内容 | RISC-V (RV64I) の例 |
|---|---|---|
| レジスタ (registers) | 汎用レジスタの本数と幅 | 32本 × 64bit (`x0`–`x31`) |
| 命令形式 (instruction format) | 命令のビット配置 | 固定32bit（R/I/S/B/U/J 型） |
| アドレッシング (addressing) | メモリ参照方法 | base + 12bit 即値オフセットのみ |
| メモリモデル (memory model) | 順序保証 | RVWMO（弱順序、[`memory-ordering-atomics.md`](memory-ordering-atomics.md) 参照） |
| 特権レベル (privilege) | 実行モード | M / S / U（マシン/スーパバイザ/ユーザ） |

### RISC vs CISC

| 観点 | RISC (RISC-V, ARM, MIPS) | CISC (x86) |
|---|---|---|
| 命令数・複雑さ | 少・単純 | 多・複雑（1命令で複合動作） |
| 命令長 | 固定（または整然）→ デコード容易 | 可変長 → デコード複雑 |
| メモリアクセス | **ロード/ストアのみ** (load-store architecture) | 多くの命令が直接メモリ参照可 |
| レジスタ数 | 多い (32) | 少ない (x86-64で16) |
| 思想 | 命令を単純化しコンパイラ/パイプラインで稼ぐ | 1命令の表現力で稼ぐ |

現代の x86 はフロントエンドで命令を**マイクロオペ (µops)** に分解し、内部はRISC的に実行する。よって「RISC対CISC」は今や思想の差であり、実装上の優劣を直接意味しない。

### RISC-V のモジュラ構成

RISC-V は単一の巨大 ISA ではなく、**基本整数命令セット + 任意の拡張**で構成される。

- **基本 (base)**: `RV32I` / `RV64I`（32/64bit 整数）、組込み向け `RV32E`。
- **標準拡張 (standard extensions)**:
  - `M` — 乗除算 (multiply/divide)
  - `A` — アトミック (atomic, [`memory-ordering-atomics.md`](memory-ordering-atomics.md) の RMW)
  - `F` / `D` — 単精度/倍精度浮動小数点
  - `C` — 圧縮命令 (16bit, コード密度向上)
  - `V` — ベクトル (vector / SIMD)
- 慣用名 **`RV64GC`** = `RV64IMAFDC`（汎用の標準的な組み合わせ）。

この「足し算」式の設計により、マイコンは `RV32IMC`、サーバは `RV64GC` + 仮想化拡張、と同じ枠組みで最適点を選べる。これが x86/ARM との最大の思想的差。

## 技術詳細 (Technical Details)

### RISC-V レジスタと呼び出し規約 (ABI)

32本の汎用レジスタには ABI 上の役割名がある（詳細は [`../hw-sw-interface/abi-syscalls.md`](../hw-sw-interface/abi-syscalls.md)）。

| レジスタ | ABI名 | 役割 |
|---|---|---|
| `x0` | `zero` | 常に 0（書いても無視） |
| `x1` | `ra` | 戻りアドレス (return address) |
| `x2` | `sp` | スタックポインタ (stack pointer) |
| `x10`–`x17` | `a0`–`a7` | 引数 / 戻り値 |
| `x5`–`x7`,`x28`–`x31` | `t0`–`t6` | 一時 (caller-saved) |
| `x8`–`x9`,`x18`–`x27` | `s0`–`s11` | 保存 (callee-saved) |

### 命令形式（固定32bit）

固定長ゆえデコードが単純で、パイプライン化（`pipeline & branch prediction`）に適する。

- **R 型**: レジスタ間演算 `add rd, rs1, rs2`
- **I 型**: 即値演算/ロード `addi rd, rs1, imm` / `ld rd, off(rs1)`
- **S 型**: ストア `sd rs2, off(rs1)`
- **B 型**: 条件分岐 `beq rs1, rs2, label`
- **U/J 型**: 上位即値/ジャンプ `lui`, `jal`

ビットレベルで見ると（R 型）次のようになる。

```
 31        25 24   20 19   15 14    12 11    7 6       0
┌────────────┬───────┬───────┬────────┬───────┬─────────┐
│   funct7   │  rs2  │  rs1  │ funct3 │  rd   │ opcode  │
└────────────┴───────┴───────┴────────┴───────┴─────────┘
add x1, x2, x3  =  0000000 00011 00010 000 00001 0110011  =  0x003100B3
```

設計上の要点は、**全形式でレジスタフィールド (rs1/rs2/rd) が同じビット位置にある**こと。デコーダは命令種別の判定を待たずにレジスタファイルの読み出しを始められる。固定長・固定フィールドは「デコードの単純さ＝パイプラインの速さ」への投資である（x86 の可変長デコードが要する複雑さとの対比）。

### 擬似命令 (Pseudo-instructions)

アセンブリに現れる「命令」の一部は、実命令への糖衣である。逆アセンブル読解時の対応表を示す。

| 擬似命令 | 実体 | 意味 |
|---|---|---|
| `nop` | `addi x0, x0, 0` | 何もしない（x0 への書込は捨てられる） |
| `mv rd, rs` | `addi rd, rs, 0` | レジスタコピー |
| `li rd, imm` | `lui` + `addi` 等に展開 | 即値ロード |
| `j label` | `jal x0, label` | 無条件ジャンプ（戻り先を捨てる） |
| `ret` | `jalr x0, 0(ra)` | 関数からの復帰 |
| `call label` | `auipc ra` + `jalr ra` | 遠距離の関数呼び出し |
| `beqz rs, label` | `beq rs, x0, label` | ゼロ比較分岐 |

`x0`（常に0）の存在が大きな役割を果たしている。コピーも nop もゼロ比較も、**専用命令を追加せず既存命令+x0 で表現**できる。ISA を小さく保つ設計判断の好例。

### アセンブリを読む — C からの対応

```c
// 例示コード (illustrative)。リポジトリ内の実コードではない。
long add(long a, long b) { return a + b; }
```

RV64I での典型的な生成（最適化時）を示す。

```asm
add:
    add  a0, a0, a1   # a0(=a) + a1(=b) → a0（戻り値レジスタ）
    ret               # ra へ戻る (jalr x0, 0(ra))
```

読み解きの要点は次のとおりである。
- 引数は `a0, a1`、戻り値は `a0`（ABI 規約）。
- ロード/ストアが現れない＝メモリに触れていない（全てレジスタ内）。
- `ret` は擬似命令で実体は `jalr`。

一段複雑な例として、配列の合計を示す。

```c
// 例示コード (illustrative)
long sum(const long *a, long n) {
    long s = 0;
    for (long i = 0; i < n; i++) s += a[i];
    return s;
}
```

```asm
sum:
    li    a5, 0          # s = 0
    beqz  a1, .done      # n == 0 なら早期リターン
.loop:
    ld    a4, 0(a0)      # a4 = *a      ← メモリアクセスはこの1命令だけ
    addi  a0, a0, 8      # a++ （long = 8バイトなので +8）
    addi  a1, a1, -1     # n--
    add   a5, a5, a4     # s += a4
    bnez  a1, .loop      # ループ継続
.done:
    mv    a0, a5         # 戻り値を a0 へ
    ret
```

この5命令のループから次のことが読める。
- 要素あたりロードは1回・逐次アクセス → プリフェッチが有効な形（[`cache-memory-hierarchy.md`](cache-memory-hierarchy.md)）。
- 律速はロードのレイテンシと分岐（`bnez` はほぼ常に taken → 予測しやすい。[`branch-prediction-speculation.md`](branch-prediction-speculation.md)）。
- ポインタインクリメント (`addi a0, a0, 8`) に「型のサイズ」が埋め込まれている。これが C のポインタ演算の実体である。

ループや配列アクセスを見ると、`ld`/`sd` の回数・アドレス計算がそのまま [`cache-memory-hierarchy.md`](cache-memory-hierarchy.md) のメモリ負荷に対応する。**アセンブリは性能の一次資料**であり、`godbolt`（Compiler Explorer）で C/C++/Rust の出力を即確認できる。

### 特権レベル (Privilege Levels)

RISC-V は3モードを定義する。
- **M (Machine)**: 最高特権。ファームウェア/ブート。
- **S (Supervisor)**: OS カーネル。仮想記憶 ([`virtual-memory-mmu.md`](virtual-memory-mmu.md)) を司る。
- **U (User)**: アプリケーション。

モード遷移（システムコール・例外・割り込み）が [`../hw-sw-interface/abi-syscalls.md`](../hw-sw-interface/abi-syscalls.md) のユーザ/カーネル境界を形作る。

### CSR — 制御・状態レジスタ (Control and Status Registers)

特権アーキテクチャの実体は、汎用レジスタとは別の **CSR** 群にある。`csrrw`/`csrrs`/`csrrc`（Zicsr 拡張）で読み書きし、モードごとにアクセス可否が決まる。

| CSR | 役割 |
|---|---|
| `mstatus` | グローバル状態（割り込み許可、前モードの記録） |
| `mtvec` / `stvec` | トラップハンドラの入口アドレス（[`interrupts-exceptions-traps.md`](interrupts-exceptions-traps.md) のベクタ） |
| `mepc` / `mcause` | トラップ発生時の PC と原因コード |
| `satp` | ページテーブルのルート（S モード。[`virtual-memory-mmu.md`](virtual-memory-mmu.md) のアドレス変換の起点） |

「OS がハードウェアを制御する」と言うとき、その具体的な操作対象はほぼ CSR である。ISA 仕様の Volume II（特権アーキテクチャ）が丸ごとこの層に充てられている。

## セキュリティ考察 (Security Considerations)

- **ISA がメモリ安全機構を提供する**: NX 相当のページ実行権限（S モードの仮想記憶経由）、特権分離（U が直接ハードを触れない）が基盤。
- **オープン ISA の利点と注意**: 仕様公開により独立した検証・形式手法が可能（[`formal-verification-landscape.md`](../../software/formal-methods/formal-verification-landscape.md)）。一方、拡張の組み合わせ自由度ゆえ、実装ごとの脆弱性（投機実行など。[`branch-prediction-speculation.md`](branch-prediction-speculation.md)）は依然マイクロアーキテクチャ依存で、ISA が公開でも安全とは限らない。
- **カスタム命令のリスク**: RISC-V はカスタム拡張を許すため、独自命令が特権チェックを誤ると保護を破りうる。検証が必須。
- **サプライチェーン**: オープンであることはコア IP の監査可能性を高めるが、製造・実装段階の信頼は別問題（ハードウェアトロイの懸念は ISA の外）。

## 結論 (Conclusions)

- ISA はハード/ソフトの契約であり、実装の多様性を吸収する抽象境界である。
- RISC-V の「小さな基本＋拡張」モジュラ設計は、組込みからサーバまで単一枠組みで覆い、ISA を「設計可能な対象」に変えた点が革新的。
- RISC/CISC の差は今や思想の差であり、現代 x86 も内部はµops で RISC 的。
- アセンブリを読む力は、性能・最適化・未定義動作を理解する一次手段であり、`cache`・`memory-ordering`・`virtual-memory` の各テーマと直結する。

### 次の接続先 (Related Topics)

- [`isa-x86-arm-comparison.md`](isa-x86-arm-comparison.md) — 同じ枠組みで現役の2大 ISA (x86-64 と AArch64) を比較する続編。
- [`../hw-sw-interface/abi-syscalls.md`](../hw-sw-interface/abi-syscalls.md) — レジスタ規約・呼び出し規約・特権境界の詳細。
- [`memory-ordering-atomics.md`](memory-ordering-atomics.md) — RISC-V の弱順序メモリモデル (RVWMO) と `A` 拡張。
- [`virtual-memory-mmu.md`](virtual-memory-mmu.md) — S モードが司るアドレス変換。
- [`interrupts-exceptions-traps.md`](interrupts-exceptions-traps.md) — `mtvec`/`mcause` を使うトラップ処理の流れ。
- [`../../software/compilers-runtimes/linkers-loaders-elf.md`](../../software/compilers-runtimes/linkers-loaders-elf.md) — 機械語がオブジェクトファイル・ELF・リロケーションとしてどう配置されるか（ソフトウェア側）。
- [`../../foundations/mathematics/discrete-logic/propositional-boolean-logic.md`](../../foundations/mathematics/discrete-logic/propositional-boolean-logic.md) — 命令が最終的に実行されるブール論理回路の基礎。

## 参考文献 (References)

**一次資料 (Primary — 仕様・教科書)**

- *The RISC-V Instruction Set Manual, Volume I: Unprivileged ISA / Volume II: Privileged Architecture*. RISC-V International. — [riscv.org/technical/specifications](https://riscv.org/technical/specifications/) — Accessed: 2026-07-02。命令形式・拡張・CSR・特権レベルの規範。
- *RISC-V ELF psABI Specification*. — [riscv-non-isa/riscv-elf-psabi-doc](https://github.com/riscv-non-isa/riscv-elf-psabi-doc) — Accessed: 2026-07-02。レジスタの ABI 名と呼び出し規約の規範。
- Hennessy, J. L., & Patterson, D. A. (2019). *Computer Architecture: A Quantitative Approach* (6th ed.). Morgan Kaufmann. — RISC 設計思想と ISA/マイクロアーキテクチャ分離の標準教科書（付録 A: ISA 原理）。

**二次資料 (Secondary)**

- Patterson, D., & Waterman, A. (2017). *The RISC-V Reader: An Open Architecture Atlas*. — [riscvbook.com](http://riscvbook.com/) — Accessed: 2026-07-02。設計判断の背景を仕様策定者自身が解説。
- Compiler Explorer. — [godbolt.org](https://godbolt.org/) — Accessed: 2026-07-02。本文のコンパイル出力の検証手段。
