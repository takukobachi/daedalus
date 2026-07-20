# 命題論理とブール代数 (Propositional Logic and Boolean Algebra)

## 概要 (Overview)

離散数学・論理シリーズの第1回。これまでの数学シリーズが扱ってきた実数の世界（解析・線形代数）と違い、計算機の内部は**有限個の離散的な値**——突き詰めれば真/偽の2値——でできている。その2値の世界の代数が**ブール代数 (Boolean algebra)** であり、その形式言語が**命題論理 (propositional logic)** である。

このテーマが土台であるのは、Shannon の 1938 年の洞察による——リレー（スイッチ）回路の直列・並列は、ブール代数の AND・OR とちょうど同じ法則に従う。つまり**論理式の変形は回路の変形であり、式の簡単化は回路の削減である**。この同一視の上に、論理ゲート・ALU・FPGA の LUT からコンパイラの条件式最適化、そして「式を充足する割り当てはあるか」(SAT) を中心とする検証技術までが建っている。本回は命題論理の意味論から出発し、標準形・関数的完全性・簡単化を経て、SAT と NP 完全性——「難しさ」の理論の入口——まで進む。

## 前提知識 (Prerequisites)

- シリーズの起点であり、数学的な前提はほぼ無い。集合の記法と、
  [`../linear-algebra/vectors-matrices-linear-maps.md`](../linear-algebra/vectors-matrices-linear-maps.md) 程度の「写像」の語彙のみ仮定する

## 本論 (Main Content)

### 1. 命題論理 — 構文と意味論 (syntax and semantics)

命題変数 p, q, … と結合子 ¬(否定)・∧(かつ)・∨(または)・→(ならば) から式を組み立てる。意味論は**真理値表 (truth table)**——各変数への真偽の割り当て（付値）ごとに式の値を定める——で与えられる (Huth & Ryan 2004, Ch. 1)。

```text
p q │ p∧q  p∨q  p→q  ¬p
0 0 │  0    0    1    1     p→q は「p が偽なら無条件に真」
0 1 │  0    1    1    1     (空虚な真, vacuous truth)。
1 0 │  0    1    0    0     「前提が満たされない場合」を
1 1 │  1    1    1    0     既定で真とする契約と読む
```

3つの中心概念を区別する。式が**恒真 (tautology)** とはすべての付値で真（例: p∨¬p）、**充足可能 (satisfiable)** とは真にする付値が少なくとも1つある、**同値 (equivalent)** とはすべての付値で値が一致する、である。恒真性と充足可能性は表裏——φ が恒真 ⇔ ¬φ が充足不能——であり、後述の SAT がこの三概念すべての判定を代表する。

### 2. ブール代数の法則 (laws of Boolean algebra)

∧・∨・¬ は以下の法則に従う（真理値表で全付値を確かめれば証明になる——有限だから全数検査が証明になる、というのが離散数学の作法である）:

```text
交換・結合・分配:  p∧(q∨r) = (p∧q)∨(p∧r)   および ∧↔∨ を入れ替えた双対
吸収:             p∨(p∧q) = p
ド・モルガン:      ¬(p∧q) = ¬p∨¬q,   ¬(p∨q) = ¬p∧¬q
```

すべての法則が **∧↔∨、0↔1 の入れ替えで対になる**（双対性）。n 変数のブール関数 f: {0,1}ⁿ → {0,1} は全部で 2^(2ⁿ) 個あり、n=2 で 16 個、n=4 で 65536 個——**有限だが爆発的**というのも離散数学の典型的な風景である。

### 3. 標準形と関数的完全性 (normal forms and functional completeness)

任意のブール関数は真理値表から機械的に式へ変換できる。真になる行ごとに変数の積（最小項）を作って ∨ で束ねれば**加法標準形 (DNF)**、偽になる行から双対に作れば**乗法標準形 (CNF)** になる。すなわち {∧, ∨, ¬} は**関数的に完全 (functionally complete)**——あらゆるブール関数を表せる。さらに NAND 1 種だけでも完全である（¬p = p NAND p から再構成する）——CMOS 実装で NAND/NOR が基本ゲートである事実と噛み合う。

もう一つの機械的分解が **Shannon 展開**:

```text
f(x, y, …) = ( x ∧ f(1, y, …) ) ∨ ( ¬x ∧ f(0, y, …) )
```

「変数 x で場合分けし、残りは小さい関数に任せる」再帰であり、回路としてはマルチプレクサ (MUX) による選択そのものである。これを全変数について展開し切った姿が真理値表＝**k 入力の任意関数をメモリに焼いた LUT (lookup table)** であり、FPGA の基本セルの数学的正体である。

### 4. 簡単化 — 式の変形は回路の削減 (minimization)

同じ関数を表す式は無数にあり、項数・リテラル数が少ない式は小さく速い回路に対応する。隣接する最小項（1 変数だけ異なる積項）は `xy ∨ x¬y = x` で併合でき、これを視覚化したのがカルノー図 (Karnaugh 1953)、系統化したのが Quine–McCluskey 法である。**ドントケア**（起こり得ない入力）を「都合よく 0/1 どちらにも読む」ことで簡単化の自由度が増える——仕様の未定義領域が最適化の資源になる、という構図はコンパイラの未定義動作の利用と同型である。

2 段（積和）回路は速いが項数が爆発しうるため、実際の論理合成は多段化・共通部分式の括り出しで面積と**段数（＝組合せ遅延）**を交換する。式の深さがクリティカルパスとしてクロック周波数を制約する——代数的な構造がタイミングという物理量に直結する。

### 5. 充足可能性 (SAT) と NP 完全性

**SAT**: 与えられた CNF 式を真にする付値は存在するか。n 変数の全数検査は 2ⁿ 通りで、これを本質的に回避できるかは未解決（P vs NP）である。Cook (1971) は SAT が **NP 完全**——「解の検証が速い問題」のすべてが SAT に変換できる——であることを示した。回路の等価性検証・プランニング・パッケージ依存解決・スケジューリングが、どれも SAT の衣を着られるということである。

決定的に重要なのは、**NP 完全は「実務で解けない」を意味しない**ことである。現代の CDCL (conflict-driven clause learning) ソルバは、単位伝播（含意の連鎖）・矛盾からの節学習・非時系列バックトラックにより、産業規模（数百万変数）の実例を日常的に解く。最悪ケースの理論的困難と、構造を持つ実例の実際的可解性は別物であり——ハードウェアの等価性検証・モデル検査・シンボリック実行（SMT ソルバは SAT に算術等の理論を載せた拡張）という検証産業がこの上に成立している。

## 工学への接続 (Engineering Connections)

- **論理ゲートは物理化されたブール代数である**: CMOS の NAND/NOR ゲートは §3 の関数的完全性の実装であり、ド・モルガン則はプルアップ網とプルダウン網の双対構造として物理に現れる。
- **FPGA の LUT は真理値表のメモリ化である** (§3): k-LUT は k 変数の任意関数を実装でき、論理合成は §4 の簡単化＋Shannon 展開による k 入力への分解を機械化したものである。
- **コンパイラと分岐**: 条件式の簡単化・短絡評価・分岐の除去 (if-conversion) は §2 の法則の適用である。条件の ∧/∨ の並べ替えは評価コストと副作用の順序に制約され、「代数的に同値」と「プログラムとして同値」の差が現れる。
- **検証の共通基盤としての SAT/SMT** (§5): 等価性検証（最適化前後の回路の XOR を SAT に問い「充足不能＝等価」）、モデル検査の有界展開 (BMC)、シンボリック実行のパス条件判定は、すべて SAT/SMT ソルバへの帰着である。
- **クエリとフィルタの述語処理**: WHERE 句・フィルタの述語正規化（CNF 化してインデックスで引ける項を抽出する）は §3 の標準形変換の応用である。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`foundations/physics/quantum-solid-state/mosfet-cmos.md`](../../physics/quantum-solid-state/mosfet-cmos.md) | CMOS NAND/NOR＝関数的完全性の物理実装、プルアップ/プルダウンの双対性 (§2–3) |
| [`hardware/fpga-reconfigurable/fpga-architecture-flow.md`](../../../hardware/fpga-reconfigurable/fpga-architecture-flow.md) | LUT＝真理値表のメモリ化、論理合成＝簡単化と Shannon 展開の機械化 (§3–4) |
| [`hardware/cpu-architecture/pipeline-superscalar-ooo.md`](../../../hardware/cpu-architecture/pipeline-superscalar-ooo.md) | 組合せ論理の段数がクリティカルパス＝クロック周期を決める (§4) |
| [`software/security/fuzzing-program-analysis.md`](../../../software/security/fuzzing-program-analysis.md) | シンボリック実行のパス条件＝SAT/SMT 判定、検証技術の土台 (§5) |
| [`hardware/hardware-security/side-channels-tee-secure-boot.md`](../../../hardware/hardware-security/side-channels-tee-secure-boot.md) | 「代数的に同値な式」が消費電力・時間で区別可能になる——同値性の限界 (§4) |

## 参考文献 (References)

- Huth, M., & Ryan, M. (2004). *Logic in Computer Science: Modelling and Reasoning about Systems* (2nd ed.). Cambridge University Press. DOI: 10.1017/CBO9780511810275 — 命題論理の構文・意味論の主参照
- Shannon, C. E. (1938). A Symbolic Analysis of Relay and Switching Circuits. *Transactions of the AIEE*, 57(12), 713–723. DOI: 10.1109/T-AIEE.1938.5057767 — 回路＝ブール代数の同一視の原典
- Karnaugh, M. (1953). The Map Method for Synthesis of Combinational Logic Circuits. *Transactions of the AIEE, Part I*, 72(9), 593–599. — 図式的簡単化の原典
- Cook, S. A. (1971). The Complexity of Theorem-Proving Procedures. *STOC '71*, 151–158. DOI: 10.1145/800157.805047 — SAT の NP 完全性の原典
- Biere, A., Heule, M., van Maaren, H., & Walsh, T. (Eds.). (2021). *Handbook of Satisfiability* (2nd ed.). IOS Press. — CDCL ソルバと応用の包括的参照
- Rosen, K. H. (2019). *Discrete Mathematics and Its Applications* (8th ed.). McGraw-Hill. — シリーズ全体の副読本
