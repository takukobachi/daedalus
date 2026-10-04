# ABI・呼び出し規約・システムコール (ABI, Calling Conventions & System Calls)

## 概要 (Overview)

ISA が「CPU が理解する命令」を定めるのに対し、**ABI (Application Binary Interface)** は「コンパイル済みコード同士が実行時にどう協調するか」を定める。関数の引数はどのレジスタに乗るか、戻り値はどこに返るか、スタックはどう使うか、構造体はメモリにどう並ぶか。ソースコードからは見えないこれらの取り決めが、別々にコンパイルされた関数やライブラリ、そして OS との連携を成立させる。

本ドキュメントは、呼び出し規約 (calling convention)・スタックフレーム・そしてユーザ空間からカーネルへ越境する**システムコール (system call)** の機構を整理し、[`isa-riscv-intro.md`](../cpu/isa-riscv-intro.md)（ISA）と [`virtual-memory-mmu.md`](../cpu/virtual-memory-mmu.md)（特権分離）の上に位置づける。実行ファイル形式・リンク・ロードのソフトウェア側の詳細は [`linkers-loaders-elf.md`](../../software/compilers-runtimes/linkers-loaders-elf.md) が扱う。

## 方法論 (Methodology)

- ABI を「呼び出し規約・データレイアウト・実行ファイル形式・syscall 規約」の層に分解する。
- 1回の関数呼び出しを、引数渡し → スタックフレーム → 戻り、という具体的な流れで追う。
- ユーザ→カーネルの越境を、特権レベル遷移 (privilege transition) と引数渡しの観点で示す。
- 例は RISC-V と x86-64 の規約を対比して具体化する。

## 発見事項 (Findings)

1. **ABI は「見えない契約」。** ソースに書かれないが、これが食い違うとリンクは通っても実行時に壊れる（引数の取り違え、スタック破壊）。
2. **呼び出し規約はレジスタの役割分担。** 引数用・戻り値用・呼び出し側保存 (caller-saved)・呼び出され側保存 (callee-saved) の区別が、性能と正しさを両立させる。
3. **システムコールは「関数呼び出しに似て非なるもの」。** 単なるジャンプではなく**特権レベルの遷移**を伴う、ハードウェアが介在する制御移行である。
4. **越境コストは高い。** モード遷移・レジスタ退避・(KPTI下では)ページテーブル切替が伴い、関数呼び出しの何百倍にもなりうる。さらにキャッシュ・TLB・分岐予測器の汚染という間接コストが、復帰後のユーザコードの性能まで落とす[^flexsc]。これが「syscall を減らす」最適化の動機。
5. **ABI 安定性が互換性の基盤。** OS の syscall ABI が安定だからこそ、古いバイナリが新しいカーネルで動く[^linux-regress]。

## アーキテクチャ (Architecture)

### ABI が定めるもの

| 層 | 内容 |
|---|---|
| 呼び出し規約 (calling convention) | 引数/戻り値のレジスタ、保存責任、スタック整列 |
| データレイアウト (data layout) | 型のサイズ・アラインメント、構造体のパディング、エンディアン |
| 実行ファイル形式 (binary format) | ELF（Linux）等、シンボル・再配置・動的リンク（[`linkers-loaders-elf.md`](../../software/compilers-runtimes/linkers-loaders-elf.md)） |
| syscall 規約 | syscall 番号、引数レジスタ、トラップ命令 |

### スタックフレーム (Stack Frame)

関数呼び出しごとに、スタック上に**フレーム (frame)** が積まれる[^csapp]。典型的な内容を示す。

```
   高アドレス
        │  ...呼び出し元のフレーム...
        ├─────────────────────────┤
        │  引数（レジスタに乗らない分） │
        │  戻りアドレス (return addr)  │ ← ra/リンクレジスタ or push
        │  保存した callee-saved レジスタ│
        │  ローカル変数               │
   sp ─▶│  （次の呼び出し用の領域）     │
   低アドレス
```

- **スタックは低アドレス方向に伸びる**（一般的な実装）。
- `sp`（スタックポインタ）がフレームの先端、`fp`（フレームポインタ, 任意）がフレーム基点。
- 関数の入口 (prologue) でフレームを確保し、出口 (epilogue) で巻き戻す。

### 呼び出し規約の要点（RISC-V / x86-64 対比）

各欄は x86-64 psABI[^x86-psabi] と RISC-V ELF psABI[^riscv-psabi] の整数呼び出し規約に基づく。

| 項目 | RISC-V (RV64) | x86-64 System V |
|---|---|---|
| 整数引数 | `a0`–`a7` (8本) | `rdi, rsi, rdx, rcx, r8, r9` (6本) |
| 戻り値 | `a0`(–`a1`) | `rax`(–`rdx`) |
| 戻りアドレス | `ra` レジスタ | スタックに `call` が push |
| callee-saved | `s0`–`s11`, `sp` | `rbx, rbp, r12`–`r15` |
| caller-saved | `t0`–`t6`, `a0`–`a7` | その他 |

**caller-saved / callee-saved の意味**:
- caller-saved（一時）: 呼び出し先が自由に壊してよい。値を保ちたければ**呼び出す側**が退避。
- callee-saved（保存）: 呼び出し先が使うなら**自分で**退避・復元する義務。呼び出し元は保たれると信じてよい。

この分担により、全レジスタを毎回退避せずに済み、性能が出る。

## 技術詳細 (Technical Details)

### 関数呼び出しの流れ（例示）

```c
// 例示コード (illustrative)。リポジトリ内の実コードではない。
long callee(long x, long y);
long caller(void) { return callee(3, 4) + 1; }
```

RV64 での概略を示す。

```asm
caller:
    addi sp, sp, -16     # フレーム確保
    sd   ra, 8(sp)       # 戻りアドレスを退避 (callee が ra を壊すため)
    li   a0, 3           # 第1引数
    li   a1, 4           # 第2引数
    call callee          # jal ra, callee（ra に戻り先を設定）
    addi a0, a0, 1       # 戻り値(a0) + 1
    ld   ra, 8(sp)       # ra 復元
    addi sp, sp, 16      # フレーム解放
    ret
```

[`isa-riscv-intro.md`](../cpu/isa-riscv-intro.md) のアセンブリ読解とつながる。`ra` を退避しているのは、`call` が `ra` を上書きするためである。これが「なぜ再帰やネスト呼び出しでスタックが要るか」の答えになる。

### データレイアウトとパディング

```c
// 例示コード (illustrative)
struct S { char a; int b; char c; };  // sizeof は 6 ではなく 12 になりうる
```

アラインメント規則により `int b` は4バイト境界に置かれ、`a` の後にパディングが入る。この並びは ABI が決め[^x86-psabi][^riscv-psabi]、[`cache-memory-hierarchy.md`](../cpu/cache-memory-hierarchy.md) のライン効率にも影響する（メンバ順を整えると縮む）。エンディアン・サイズも ABI 依存で、異機種間のバイナリ互換やシリアライズで問題になる。

### システムコール (System Calls)

ユーザプログラムがファイル I/O やメモリ確保など特権操作を行うには、カーネルへ依頼する必要がある。これが syscall。

**通常の関数呼び出しとの違い**: 単なる `call` ではモードが変わらない。syscall は**専用命令でハードウェアにトラップ**させ、CPU を U モード → S モードへ遷移させる[^riscv-priv]。トラップ全般の機構は [`interrupts-exceptions-traps.md`](../cpu/interrupts-exceptions-traps.md) を参照。

- **トラップ命令**: RISC-V `ecall`[^riscv-priv]、x86-64 `syscall`[^intel-sdm]、ARM `svc`。
- **規約（Linux 例）**[^syscall-man]:
  - syscall 番号をレジスタに（x86-64: `rax`、RISC-V: `a7`）。
  - 引数を規定レジスタに（x86-64: `rdi, rsi, rdx, r10, r8, r9`、RISC-V: `a0`–`a5`）。x86-64 では第4引数が関数呼び出し規約の `rcx` ではなく `r10` になる。`syscall` 命令が戻りアドレスを `rcx` に、RFLAGS を `r11` に退避して上書きするためである[^intel-sdm]。
  - トラップ → カーネルのハンドラ (trap handler) が番号で分岐し処理 → 戻り値をレジスタに置いてユーザへ復帰。

```
  [User U-mode]                         [Kernel S-mode]
   a7 = __NR_write                       trap handler:
   a0 = fd; a1 = buf; a2 = len             番号(a7)で sys_write へ分岐
   ecall  ───────(特権遷移)──────────▶     引数を検証して実行
   ◀──────────(sret で復帰)──────────     戻り値を a0 に
   a0 = 結果
```

### 越境コストと最適化

syscall は関数呼び出しの何百倍も重い。理由は次のとおりである。
- モード遷移とレジスタ退避/復元（直接コスト）。
- カーネル実行によるキャッシュ・TLB・分岐予測器の汚染（間接コスト）。Soares と Stumm は、この間接コストが直接コストを上回りうることを測定で示した[^flexsc]。
- KPTI（[`virtual-memory-mmu.md`](../cpu/virtual-memory-mmu.md)）下ではページテーブル切替で TLB が冷える。KPTI は KAISER[^kaiser] を起源とする、ユーザ/カーネルのページテーブル分離である。

**最適化の動機**: システムコールを減らす／まとめる。具体例を以下に挙げる。
- バッファリング（`write` をまとめる）、`readv`/`writev`（ベクタI/O）、`io_uring`（投入/完了をキュー化して syscall 回数を激減）[^io-uring]、`vDSO`（`gettimeofday` 等をトラップなしでユーザ空間実行）[^vdso]。

### ABI 安定性と互換性

Linux カーネルは「ユーザ空間を壊さない」("We don't cause regressions") を原則とし、syscall ABI を安定に保つ[^linux-regress]。だから何年も前のバイナリが新カーネルで動く。逆にライブラリ ABI（例: glibc のシンボルバージョン）の破壊は、再コンパイルなしの動作を壊す。ABI は互換性契約そのもの。

## セキュリティ考察 (Security Considerations)

- **境界での引数検証**: カーネルはユーザから渡されたポインタ・長さを**信頼せず検証**する。`copy_from_user`/`copy_to_user` でアクセス可能性と権限を確認しないと、カーネルメモリの読み書き（権限昇格）につながる。
- **スタック保護**: 戻りアドレスがスタックにあるため、バッファオーバーフローで書き換えると制御を奪われる（ROP 等）。対策: スタックカナリア (stack canary)[^stackguard]、[`virtual-memory-mmu.md`](../cpu/virtual-memory-mmu.md) の NX/W^X、ASLR、シャドースタック。ROP は既存コード断片 (gadget) を戻りアドレスの連鎖でつなぎ、コード注入なしに任意計算を組み立てる[^rop]。ソフトウェア側の緩和策の全体像は [`memory-safety-exploit-mitigations.md`](../../software/security/memory-safety-exploit-mitigations.md) を参照。
- **syscall フィルタリング**: `seccomp-bpf` で許可する syscall を制限し、攻撃面を縮小する（コンテナ/サンドボックスの基盤）[^seccomp]。サンドボックス設計での使われ方は [`sandbox-architectures.md`](../../software/security/sandbox-architectures.md) を参照。
- **投機実行との交差**: syscall 境界は Spectre 類の対策点でもあり、境界での投機バリアやポインタのサニタイズが入る。
- **ABI 混同**: 32bit/64bit や呼び出し規約の取り違えはスタック破壊・情報漏洩を生む。FFI（言語間連携）では特に厳密な ABI 整合が要る。

## 結論 (Conclusions)

- ABI は「コンパイル済みコード同士・OS との実行時契約」であり、ソースに現れないがリンク後の正しさを支配する。
- 呼び出し規約の caller/callee-saved 分担とスタックフレームが、関数呼び出し・再帰・性能の実体を形作る。
- システムコールは関数呼び出しに似るが、特権レベル遷移を伴うハードウェア介在の越境で、コストが高い。これが syscall 削減最適化（バッファリング、io_uring、vDSO）の動機。
- 境界はセキュリティの最前線でもあり、引数検証・スタック保護・seccomp が要となる。

### 次の接続先 (Related Topics)

- [`../cpu/isa-riscv-intro.md`](../cpu/isa-riscv-intro.md) — レジスタと命令、特権レベルの基礎。
- [`../cpu/virtual-memory-mmu.md`](../cpu/virtual-memory-mmu.md) — KPTI と越境コスト、メモリ保護。
- [`../cpu/interrupts-exceptions-traps.md`](../cpu/interrupts-exceptions-traps.md) — `ecall`/`syscall` を含むトラップ機構の全体。
- [`mmio-dma-interrupts.md`](mmio-dma-interrupts.md) — もう一つの境界（CPU とデバイス）との対比。
- [`../../software/compilers-runtimes/linkers-loaders-elf.md`](../../software/compilers-runtimes/linkers-loaders-elf.md) — ELF・再配置・動的リンクという ABI の実行ファイル形式層。
- [`../../software/operating-systems/process-thread-scheduling.md`](../../software/operating-systems/process-thread-scheduling.md) — syscall の受け手であるカーネル側のプロセス管理。

## 参考文献 (References)

**一次資料 (Primary — 仕様・公式ドキュメント)**

[^x86-psabi]: *System V Application Binary Interface — AMD64 Architecture Processor Supplement* (x86-64 psABI). — [gitlab.com/x86-psABIs/x86-64-ABI](https://gitlab.com/x86-psABIs/x86-64-ABI) — Accessed: 2026-07-02。x86-64 の呼び出し規約・スタックフレーム・データレイアウトの規範。

[^riscv-psabi]: *RISC-V ELF psABI Specification*. — [riscv-non-isa/riscv-elf-psabi-doc](https://github.com/riscv-non-isa/riscv-elf-psabi-doc) — Accessed: 2026-07-02。RISC-V の呼び出し規約の規範。

[^riscv-priv]: RISC-V International. *The RISC-V Instruction Set Manual, Volume II: Privileged Architecture*. — [riscv.org/specifications/ratified](https://riscv.org/specifications/ratified/) — Accessed: 2026-10-04。`ecall` による環境呼び出し例外、`sret` による復帰、U/S モード遷移の規範。

[^intel-sdm]: Intel Corporation. *Intel 64 and IA-32 Architectures Software Developer's Manual*, Vol. 2 (SYSCALL/SYSRET 命令). — [intel.com/sdm](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) — Accessed: 2026-10-04。`syscall` が RIP を RCX に、RFLAGS を R11 に保存する動作の規範。

[^syscall-man]: Linux man-pages, *syscall(2)* — アーキテクチャ別のシステムコール呼び出し規約（レジスタ対応表）。 — [man7.org](https://man7.org/linux/man-pages/man2/syscall.2.html) — Accessed: 2026-07-02。

[^io-uring]: Linux man-pages, *io_uring(7)*. — [man7.org](https://man7.org/linux/man-pages/man7/io_uring.7.html) — Accessed: 2026-10-04。共有リング（SQ/CQ）と SQ ポーリングによる syscall 削減。

[^vdso]: Linux man-pages, *vdso(7)*. — [man7.org](https://man7.org/linux/man-pages/man7/vdso.7.html) — Accessed: 2026-10-04。頻出 syscall（`gettimeofday` 等）をユーザ空間の通常関数呼び出しに置き換える仕組み。

[^seccomp]: The Linux Kernel documentation. *Seccomp BPF (SECure COMPuting with filters)*. — [docs.kernel.org](https://docs.kernel.org/userspace-api/seccomp_filter.html) — Accessed: 2026-10-04。利用可能な syscall を絞りカーネル攻撃面を縮小する機構。

[^linux-regress]: The Linux Kernel documentation. *Reporting regressions* ("We don't cause regressions" 原則). — [docs.kernel.org](https://docs.kernel.org/admin-guide/reporting-regressions.html) — Accessed: 2026-10-04。ユーザ空間を壊さないという互換性方針の公式記述。

**一次資料 (Primary — 教科書・論文)**

[^csapp]: Bryant, R. E., & O'Hallaron, D. R. (2016). *Computer Systems: A Programmer's Perspective* (3rd ed.). Pearson. — Ch. 3（機械語レベルの手続き呼び出し・スタックフレーム）、Ch. 8（例外制御フローとシステムコール）。

[^flexsc]: Soares, L., & Stumm, M. (2010). FlexSC: Flexible System Call Scheduling with Exception-Less System Calls. *Proc. 9th USENIX OSDI*, 33–46. — [usenix.org](https://www.usenix.org/conference/osdi10/flexsc-flexible-system-call-scheduling-exception-less-system-calls) — Accessed: 2026-10-04。syscall の直接コストとキャッシュ・TLB 汚染による間接コストの測定。

[^kaiser]: Gruss, D., Lipp, M., Schwarz, M., Fellner, R., Maurice, C., & Mangard, S. (2017). KASLR is Dead: Long Live KASLR. *Engineering Secure Software and Systems (ESSoS 2017)*, LNCS 10379. DOI: 10.1007/978-3-319-62105-0_11 — KPTI の原型 KAISER（ユーザ/カーネルのアドレス空間分離）。

[^stackguard]: Cowan, C. et al. (1998). StackGuard: Automatic Adaptive Detection and Prevention of Buffer-Overflow Attacks. *Proc. 7th USENIX Security Symposium*. — [usenix.org](https://www.usenix.org/legacy/publications/library/proceedings/sec98/full_papers/cowan/cowan.pdf) — Accessed: 2026-10-04。スタックカナリアの原典。

[^rop]: Shacham, H. (2007). The Geometry of Innocent Flesh on the Bone: Return-into-libc without Function Calls (on the x86). *Proc. 14th ACM CCS*, 552–561. DOI: 10.1145/1315245.1315313 — ROP（return-oriented programming）の原典。
