# 計算量理論 (Computational Complexity)

## 概要 (Overview)

離散数学・論理シリーズの第4回。第3回 ([`automata-formal-languages.md`](automata-formal-languages.md) §5) は「計算できるか」を問い、停止性問題のように原理的に解けない問題があることを示した。本回は、計算できる問題について「**どれだけの時間と記憶で計算できるか**」を問う。解けることが分かっていても、入力が少し大きくなるだけで宇宙の年齢を超える時間がかかるなら、実務では解けないのと同じである。

Daedalus の分析文書には「この問題は NP 困難なので、実装はヒューリスティクスで近似する」という記述が繰り返し現れる。レジスタ割り当て、命令スケジューリング、FPGA の配置配線、クエリの結合順序、発電機の起動停止計画、コンテナの配置がその例である。本回はこの決まり文句の中身を明らかにする。(1) 計算量の測り方と多項式・指数の差、(2) P と NP、(3) 帰着と NP 完全性、(4) NP 困難な問題への実務の対処、(5) NP の外側にある計算量クラス、(6) 最悪計算量と実際の性能のずれを順に扱う。SAT と CDCL ソルバは第1回 ([`propositional-boolean-logic.md`](propositional-boolean-logic.md) §5) で扱ったので、本回はその背後にある理論の枠組みに焦点を当てる。

## 前提知識 (Prerequisites)

- [`propositional-boolean-logic.md`](propositional-boolean-logic.md) — SAT と NP 完全性の入口、CDCL ソルバ (シリーズ第1回)
- [`graphs-trees-algorithms.md`](graphs-trees-algorithms.md) — グラフ、最短経路などの多項式時間アルゴリズム (シリーズ第2回)
- [`automata-formal-languages.md`](automata-formal-languages.md) — チューリング機械、停止性問題 (シリーズ第3回)

## 本論 (Main Content)

### 1. 計算量の測り方 (measuring complexity)

計算量は、入力の大きさ n の関数として、必要な基本操作の回数 (時間計算量) や記憶量 (空間計算量) を数える。定数倍と低次の項を無視して増え方だけを見るのが O 記法である。最悪の入力で測るのが**最悪計算量 (worst-case complexity)** であり、計算量理論の標準はこれである。

増え方の違いは、n が大きくなると決定的になる。1 秒に 10⁹ 回の操作ができる計算機で n = 100 の入力を考える。

```text
増え方    n = 100 での操作回数     所要時間の目安
n log n   約 7 × 10²              1 μs 未満
n²        10⁴                      10 μs
n³        10⁶                      1 ms
2ⁿ        約 1.3 × 10³⁰           約 4 × 10¹³ 年
n!        約 9.3 × 10¹⁵⁷          —
```

多項式 (n^k) と指数 (2ⁿ) の差は、計算機が速くなっても埋まらない。計算機が 1000 倍速くなると、n³ のアルゴリズムで同じ時間に扱える n は 10 倍に増えるが、2ⁿ のアルゴリズムでは約 10 しか増えない。この差から、「多項式時間で解ける問題を効率よく解ける問題とみなす」という慣習 (Cobham–Edmonds のテーゼ) が生まれた。ただし n¹⁰⁰ のアルゴリズムは実用にならないし、1.001ⁿ のアルゴリズムは実用的な n で十分速い。この慣習は理論を組み立てるための目安であって、実務での速さを直接保証するものではない。

### 2. P と NP (P and NP)

計算量理論は主に**判定問題** (答えが「はい」か「いいえ」の問題) を扱う。最適化問題は「コスト k 以下の解はあるか」という判定問題に直せる。

```text
P:  多項式時間で答えを求められる判定問題の集まり
    例: 最短経路の長さが k 以下か、2-SAT、線形計画 (LP)

NP: 答えが「はい」のとき、それを裏付ける証拠 (証明書) を
    多項式時間で検証できる判定問題の集まり
    例: SAT (証拠 = 充足する変数割り当て)
        グラフの k 彩色 (証拠 = 各頂点の色)
        巡回セールスマン「長さ k 以下の巡回路はあるか」(証拠 = 巡回路)
        部分和「和がちょうど T になる部分集合はあるか」(証拠 = 部分集合)
```

解ける問題は検証もできるので P ⊆ NP である。逆に、検証が速い問題はすべて解くのも速いのか (P = NP か) は未解決であり、Clay 数学研究所のミレニアム懸賞問題の一つになっている。大多数の研究者は P ≠ NP と予想している。NP の「N」は「非決定性 (nondeterministic)」の意味で、「指数時間」の意味ではない。正しい証拠を推測で言い当てられる機械なら多項式時間で解ける、という定義に由来する。

検証の向きを逆にしたクラスが **co-NP** である。答えが「いいえ」のときに短い証拠がある問題の集まりで、「この論理式はどんな割り当てでも真か (恒真性)」が代表例になる。偽にする割り当てが1つあれば「いいえ」を示せるが、「はい」を示す短い証拠は知られていない。「このプログラムにはバグがない」を示す方が、バグを1つ見つけるより難しいと考えられているのは、この非対称性による。

### 3. 帰着と NP 完全性 (reductions and NP-completeness)

問題 A を問題 B に**多項式時間帰着 (polynomial-time reduction)** するとは、A の任意の入力を、多項式時間で B の入力に変換し、答えが一致するようにすることである。B を解ければ A も解けるので、「B は A と同じかそれ以上に難しい」と言える。

```text
NP 困難 (NP-hard):   NP のすべての問題がこの問題に帰着できる
NP 完全 (NP-complete): NP 困難であり、かつそれ自身が NP に属する

Cook (1971):  SAT は NP 完全 (最初の NP 完全問題)
Karp (1972):  SAT からの帰着で、21 の組合せ問題 (彩色、ハミルトン閉路、
              ナップサック、集合被覆など) が NP 完全であることを示した
```

1つの NP 完全問題が多項式時間で解ければ、帰着を通じて NP のすべての問題が多項式時間で解け、P = NP になる。逆に P ≠ NP なら、NP 完全問題はどれも多項式時間では解けない。数千の NP 完全問題が知られているが、どれにも多項式時間のアルゴリズムが見つかっていないことが、P ≠ NP の予想の状況証拠になっている (Garey & Johnson 1979)。

帰着は両方向に使える。新しい問題が NP 困難であることを示すには、既知の NP 完全問題をその問題へ帰着する。逆に、新しい問題を SAT へ帰着 (符号化) すれば、SAT ソルバで解ける。グラフの 3 彩色なら、「頂点 v が色 c を持つ」を表す変数を頂点ごとに 3 つ用意し、「各頂点はいずれかの色を持つ」「隣り合う頂点は同じ色を持たない」を節として書けばよい。ハードウェア検証やパッケージの依存解決が SAT ソルバを使えるのは、この方向の帰着による。

最適化問題では「NP 完全」ではなく「NP 困難」と言うのが正確である。最適解を求める問題は判定問題ではないので NP に属さないが、その判定版が NP 完全なら最適化版は NP 困難になる。

### 4. NP 困難な問題への対処 (coping with NP-hardness)

NP 困難であることは、その問題が実務で解けないことを意味しない。意味するのは「**すべての入力で、最適解を、多項式時間で**」の3つを同時には満たせない (P ≠ NP なら) ということである。実務の対処法は、このうちどれを緩めるかで分類できる (Garey & Johnson 1979; Arora & Barak 2009)。

- **すべての入力を諦める (特殊な構造を使う)**: 一般のグラフ彩色は NP 完全だが、SSA 形式のプログラムから作る干渉グラフは弦グラフ (chordal graph) になり、多項式時間で最適に彩色できる (Hack et al. 2006)。クエリの結合順序の最適化も、一般の結合グラフでは NP 困難だが、結合グラフが非巡回で特定の性質を持つコスト関数なら多項式時間で解ける (Ibaraki & Kameda 1984)。2-SAT は P に属し、3-SAT は NP 完全である。困難さは問題の名前ではなく、入力が持ちうる構造で決まる。
- **最適解を諦める (近似アルゴリズム)**: 頂点被覆は、極大マッチングの両端点を選ぶだけで最適の2倍以内の解が得られる。三角不等式を満たす巡回セールスマン問題は、Christofides 法で最適の 1.5 倍以内に収まる。ただし近似にも限界がある。MAX-3SAT では、ランダムな割り当てで節の 7/8 を期待値として満たせるが、それより少しでも良い比率を保証することは NP 困難である (Håstad 2001)。
- **最悪時の多項式時間を諦める (厳密解法と擬多項式時間)**: 分枝限定法、整数計画ソルバ、CDCL ソルバは最悪では指数時間だが、構造のある実例では十分速く最適解を出す。ナップサック問題は容量 W に対して O(nW) の動的計画法で解ける。入力の大きさは W の桁数 (log W) で測るので、これは多項式時間ではなく**擬多項式時間 (pseudo-polynomial time)** だが、W が現実的な大きさなら実用になる。
- **保証そのものを諦める (ヒューリスティクス)**: 貪欲法、局所探索、焼きなまし法は、解の質にも時間にも理論的な保証を持たないが、多くの実務の問題で十分な解を素早く出す。Kubernetes のスケジューラは、配置を最適化問題として解かず、絞り込みと得点付けで1つずつ決める。

### 5. NP の外側 (beyond NP)

NP は計算量の階層の一段にすぎない。主なクラスの包含関係は次のとおりである。

```text
P ⊆ NP ⊆ PSPACE ⊆ EXPTIME     (P ≠ EXPTIME は証明済み。途中のどこが真の包含かは未解決)

PSPACE: 多項式の記憶量で解ける問題。時間は指数でもよい
  例: 量化子付きブール式 (QBF) の真偽、一般化地理ゲーム (generalized geography) の必勝判定、
      LTL (線形時相論理) の充足可能性とモデル検査 (Sistla & Clarke 1985)
EXPTIME: 指数時間で解ける問題
  例: n×n に一般化したチェスの必勝判定 (Fraenkel & Lichtenstein 1981)、
      ML 系言語の型推論 (Hindley–Milner) の判定 (Mairson 1990)
判定不能: どれだけ時間をかけても一般には解けない (第3回 §5)
```

モデル検査が PSPACE 完全であることは、検証ツールが状態爆発と戦う理由を説明する。ただし、この困難さの大部分は式の大きさに対して指数的であり、検査対象のモデルの大きさに対しては線形である。式は通常短いので、実務の障害は式よりもモデルの状態数になる。ML の型推論も最悪では指数時間だが、その最悪ケースは型の入れ子を人工的に深くしたプログラムでしか現れない。

暗号の安全性は、NP 完全性とは別の種類の困難さに依存している。RSA が頼る素因数分解は NP と co-NP の両方に属するので、NP 完全である可能性は低い (もし NP 完全なら NP = co-NP になる)。暗号に必要なのは「最悪の入力で難しい」ことではなく、「ランダムに選んだ鍵のほとんどで難しい」こと、つまり平均的な困難さである。この2つは別の性質であり、P ≠ NP が証明されても暗号の安全性が保証されるわけではない。逆に、量子計算機上の Shor のアルゴリズムは素因数分解を多項式時間で解く (Shor 1997)。量子計算機が NP 完全問題を効率よく解けるとは考えられていないが、素因数分解はその範囲の外にある。耐量子暗号への移行が必要とされるのはこのためである。

### 6. 最悪計算量と実際の性能のずれ (worst case versus practice)

最悪計算量は保証としては強いが、実際の性能の予測としては外れることがある。

```text
問題/アルゴリズム       最悪計算量        実際
単体法 (LP)             指数時間          ほぼ常に速い
                        (Klee & Minty 1972 の反例)
ML の型推論             指数時間          実用のプログラムでは問題にならない
CDCL ソルバ (SAT)       指数時間          構造のある実例で数百万変数を解く
バックトラック型正規表現  指数時間          通常は速いが、悪意ある入力で ReDoS
```

単体法のずれを説明するのが、Spielman & Teng (2004) の**平滑化解析 (smoothed analysis)** である。最悪の入力にわずかなランダムな揺らぎを加えると、単体法の期待実行時間は多項式になる。最悪ケースは入力の空間の中で極端に狭い場所にあり、少し揺らすだけで外れる。現実のデータは測定誤差などの揺らぎを含むので、最悪ケースに当たりにくい。

最後の行は逆向きの教訓である。最悪ケースが現実の入力で起きにくくても、攻撃者は最悪ケースを意図的に作れる。ハッシュ表の衝突攻撃や ReDoS は、平均では速いアルゴリズムの最悪ケースを外部の入力から引き起こす攻撃である。外部から入力を受け取るシステムでは、平均性能ではなく最悪計算量が安全性の問題になる。

## 工学への接続 (Engineering Connections)

- **コンパイラのバックエンド**: レジスタ割り当て (グラフ彩色)、命令スケジューリング、命令選択は一般に NP 困難である。実装は、SSA の弦グラフ性のような構造の利用 (§4) と、リストスケジューリングのようなヒューリスティクスを組み合わせる。
- **数理最適化と市場**: 線形計画は多項式時間で解けるが、整数変数を含むと NP 困難になる。電力市場の起動停止計画は混合整数計画として、分枝限定法で「最適性ギャップ数 % 以内」の解を時間内に求めている (§4 の最悪時の保証を緩める対処)。
- **形式検証**: SAT は NP 完全、LTL モデル検査は PSPACE 完全、プログラムの一般的な性質は判定不能 (第3回)。検証ツールの設計は、どの困難さの段に問題を収めるかの選択である。並行システムの線形化可能性の検査も一般には NP 完全であり、検査ツールは探索の枝刈りで実用化している。
- **依存関係の解決**: パッケージの依存と競合の制約から、インストール可能な組み合わせを求める問題は NP 完全である (Mancinelli et al. 2006)。パッケージマネージャが SAT ソルバや貪欲な近似を使い分けるのはこのためである。
- **アルゴリズム的な DoS**: ハッシュ衝突や ReDoS のように、平均で速いアルゴリズムの最悪ケースを外部入力で引き起こす攻撃がある (§6)。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`software/compilers-runtimes/compiler-backend-code-generation.md`](../../../software/compilers-runtimes/compiler-backend-code-generation.md) | 命令選択・スケジューリング・レジスタ割り当ての NP 困難性と、ヒューリスティクスによる分割統治 (§4) |
| [`software/databases/query-execution-optimization.md`](../../../software/databases/query-execution-optimization.md) | 結合順序の探索 (System R 型の DP、PostgreSQL の遺伝的最適化) の背景にある困難性 (§4) |
| [`hardware/fpga/fpga-architecture-flow.md`](../../../hardware/fpga/fpga-architecture-flow.md) | NP 困難な配置配線と、反復によるタイミングクロージャ (§4) |
| [`engineering/energy/electricity-markets-dispatch.md`](../../../engineering/energy/electricity-markets-dispatch.md) | 起動停止計画の混合整数計画と最適性ギャップ (§4) |
| [`software/cloud/container-orchestration-kubernetes.md`](../../../software/cloud/container-orchestration-kubernetes.md) | 厳密な最適化を放棄したスケジューラの設計 (§4) |
| [`software/distributed-systems/consistency-models.md`](../../../software/distributed-systems/consistency-models.md) | 線形化可能性の検査が一般に NP 完全であること |
| [`software/formal-methods/formal-verification-landscape.md`](../../../software/formal-methods/formal-verification-landscape.md) | SAT・SMT・モデル検査の困難さの段と、検証ツールの設計 (§5) |
| [`../number-theory-algebra/public-key-cryptography.md`](../number-theory-algebra/public-key-cryptography.md) | 素因数分解の困難さが平均的な困難さであること、Shor のアルゴリズム (§5) |
| [`../optimization/convex-optimization-duality.md`](../optimization/convex-optimization-duality.md) | 凸最適化・LP は多項式時間、整数制約で NP 困難になる境界 (§4, §6) |

## 参考文献 (References)

- Garey, M. R., & Johnson, D. S. (1979). *Computers and Intractability: A Guide to the Theory of NP-Completeness*. W. H. Freeman. — NP 完全性の理論と問題の一覧、対処法の古典
- Arora, S., & Barak, B. (2009). *Computational Complexity: A Modern Approach*. Cambridge University Press. — 計算量クラス、近似困難性、平均的な困難さと暗号
- Sipser, M. (2012). *Introduction to the Theory of Computation* (3rd ed.). Cengage Learning. — P・NP・PSPACE と帰着の標準的教科書
- Cook, S. A. (1971). The Complexity of Theorem-Proving Procedures. *Proc. STOC '71*, 151–158. DOI: 10.1145/800157.805047 — SAT の NP 完全性
- Karp, R. M. (1972). Reducibility among Combinatorial Problems. In R. E. Miller & J. W. Thatcher (Eds.), *Complexity of Computer Computations* (pp. 85–103). Plenum Press. DOI: 10.1007/978-1-4684-2001-2_9 — 21 の NP 完全問題
- Håstad, J. (2001). Some Optimal Inapproximability Results. *Journal of the ACM*, 48(4), 798–859. DOI: 10.1145/502090.502098 — MAX-3SAT の 7/8 近似の限界
- Hack, S., Grund, D., & Goos, G. (2006). Register Allocation for Programs in SSA-Form. *Compiler Construction (CC 2006)*, 247–262. Springer. — SSA の干渉グラフが弦グラフであること
- Ibaraki, T., & Kameda, T. (1984). On the Optimal Nesting Order for Computing N-Relational Joins. *ACM Transactions on Database Systems*, 9(3), 482–502. — 結合順序の困難性と多項式時間で解ける場合
- Mancinelli, F., Boender, J., Di Cosmo, R., Vouillon, J., Durak, B., Leroy, X., & Treinen, R. (2006). Managing the Complexity of Large Free and Open Source Package-Based Software Distributions. *Proc. ASE 2006*, 199–208. DOI: 10.1109/ASE.2006.49 — パッケージのインストール可能性の NP 完全性
- Sistla, A. P., & Clarke, E. M. (1985). The Complexity of Propositional Linear Temporal Logics. *Journal of the ACM*, 32(3), 733–749. — LTL の充足可能性・モデル検査の PSPACE 完全性
- Fraenkel, A. S., & Lichtenstein, D. (1981). Computing a Perfect Strategy for n×n Chess Requires Time Exponential in n. *Journal of Combinatorial Theory, Series A*, 31(2), 199–214. — 一般化チェスの EXPTIME 完全性
- Mairson, H. G. (1990). Deciding ML Typability is Complete for Deterministic Exponential Time. *Proc. POPL '90*, 382–401. — ML 型推論の EXPTIME 完全性
- Klee, V., & Minty, G. J. (1972). How Good is the Simplex Algorithm? In O. Shisha (Ed.), *Inequalities III* (pp. 159–175). Academic Press. — 単体法が指数時間かかる反例
- Spielman, D. A., & Teng, S.-H. (2004). Smoothed Analysis of Algorithms: Why the Simplex Algorithm Usually Takes Polynomial Time. *Journal of the ACM*, 51(3), 385–463. — 平滑化解析
- Shor, P. W. (1997). Polynomial-Time Algorithms for Prime Factorization and Discrete Logarithms on a Quantum Computer. *SIAM Journal on Computing*, 26(5), 1484–1509. DOI: 10.1137/S0097539795293172 — 量子計算機による素因数分解
