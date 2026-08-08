# 凸最適化と双対性 (Convex Optimization and Duality)

## 概要 (Overview)

最適化シリーズの第1回。「制約の下で目的関数を最小化する」という問題形式——min f(x) subject to gᵢ(x) ≤ 0——は、工学の驚くほど広い範囲の共通言語である。ロケットの着陸誘導（燃料最小・着地点制約）、電力系統の経済負荷配分（コスト最小・需給一致制約）、機械学習の訓練（損失最小・正則化）、モデル予測制御（追従誤差最小・アクチュエータ制約）は、すべてこの1つの形の実例である。

本文書の主張は2つ。第一に、この問題クラスの「解ける/解けない」の分水嶺は問題の大きさではなく**凸性 (convexity)** である——凸問題では局所最適が大域最適であり、数百万変数でも確実に解けるが、非凸では小さな問題すら原理的に困難になる。ゆえに実務の核心は「解くこと」より「**どう凸に定式化するか**」にある。第二に、制約付き最適化が生む**双対変数 (dual variable)** は「制約を1単位緩めたときの目的関数の改善量」——**影の価格 (shadow price)**——という経済的実体を持ち、電力市場の価格やネットワークの公平なレート配分は、この数学的対象が現実の制度・プロトコルとして実装されたものである。理論が比喩でなく機構として市場と通信路を動かしている。

## 前提知識 (Prerequisites)

- [`../calculus-analysis/multivariable-calculus.md`](../calculus-analysis/multivariable-calculus.md) — 勾配、ヘッセ行列、テイラー展開、鞍点
- [`../linear-algebra/vectors-matrices-linear-maps.md`](../linear-algebra/vectors-matrices-linear-maps.md) — 内積、正定値性、二次形式
- [`../linear-algebra/matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) — §3 のニュートン法が解く線形系と条件数

## 本論 (Main Content)

### 1. 凸性 — 「解ける」ことの正体 (convexity)

集合 C が**凸 (convex)** とは、C 内の任意の2点を結ぶ線分が C に含まれること。関数 f が凸とは、グラフ上の任意の2点を結ぶ弦が常にグラフの上側にあること（f″ ≥ 0、多変数ではヘッセ行列の半正定値性）。**凸関数を凸集合上で最小化する**のが凸最適化であり、その決定的な性質が:

```text
凸問題では、局所最適解は大域最適解である
```

証明は一行——局所最適 x より良い点 y があれば、x と y を結ぶ線分上で f は弦の下側（凸性）にあり、x のすぐ隣に x より良い点が存在して局所最適性に反する。この定理が意味するのは、**「坂を下り続ければ着く場所が答えである」ことの保証**である。非凸問題ではこの保証が消え、勾配法は無数の局所解・鞍点のどれかに捕まる——大域最適の証明は一般に NP 困難になる。

凸問題の内側にはさらに階層がある。**LP**（線形計画: 目的も制約も線形）⊂ **QP**（二次計画: 目的が凸二次）⊂ **SOCP**（二次錐計画: ノルム制約 ‖Ax+b‖ ≤ cᵀx+d を許す）⊂ **SDP**（半正定値計画）。右へ行くほど表現力が増し、解く手間も増える。実務の定式化とは「自分の問題をこの階層のなるべく左に落とす」作業であり、モデリング言語 (CVX 系) はこの変換の自動化である。

### 2. 勾配法 — 一階の情報で降りる (gradient methods)

無制約の凸最小化の最も素朴な解法が**勾配降下 (gradient descent)**: xₖ₊₁ = xₖ − η∇f(xₖ)。収束の速さを支配するのが**条件数 (condition number)** κ——目的関数の等高線の「潰れ具合」（ヘッセ行列の最大・最小固有値比）である。κ が大きい細長い谷では、勾配は谷底ではなく対岸を向き、反復はジグザグして κ に比例する回数を要する。**前処理・正規化・モーメンタム**という実務の道具はすべて「κ を実効的に下げる」工夫として統一的に読める（[`../linear-algebra/matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) の反復法の条件数依存と同じ構図）。

二階の情報を使う**ニュートン法** xₖ₊₁ = xₖ − [∇²f]⁻¹∇f は、等高線を局所的に球に変換してから降りるため条件数の影響を受けず、最適解近傍で二次収束する。代償は毎反復の線形系求解 O(n³) であり、次元 n との取引になる——L-BFGS（曲率の低ランク近似）や内点法（§4）はこの取引の中間解である。

機械学習の訓練が使う**確率的勾配降下 (SGD)** は、全データの勾配ではなくミニバッチの**不偏だがノイズを含む勾配推定**で進む (Robbins & Monro 1951)。1 ステップが桁で安く、しかも損失関数が非凸（深層学習）でもノイズが鞍点からの脱出を助けるため、「凸理論の保証が消えた領域で、実験的に大域的に良い解へ着く」という現代最大の最適化実務が成立している——理論と実務のこのずれは §5 で戻る。

### 3. 制約と KKT 条件 — 最適性の代数 (Lagrange multipliers and KKT)

制約 gᵢ(x) ≤ 0 付きの最小化には**ラグランジュ関数** L(x, λ) = f(x) + Σλᵢgᵢ(x) を導入する。最適解での必要条件が **KKT 条件 (Karush–Kuhn–Tucker conditions)**:

```text
停留性:      ∇f(x*) + Σ λᵢ*∇gᵢ(x*) = 0     (目的の勾配は、効いている制約の
                                             勾配の重み付き和と釣り合う)
実行可能性:  gᵢ(x*) ≤ 0,  λᵢ* ≥ 0
相補性:      λᵢ* gᵢ(x*) = 0                 (効いていない制約の乗数は 0)
```

幾何的には「これ以上目的を下げる方向は、すべて制約の壁が塞いでいる」という力の釣り合いであり、凸問題（＋制約想定）では必要十分条件になる。**乗数 λᵢ は制約 i の「壁を押す力」**——制約を1単位緩めたら目的関数がどれだけ改善するかの感度であり、次節の影の価格の実体である。相補性は「触れていない壁は力を出さない」——アクティブでない制約は解に影響しない——を代数として述べたものである。

### 4. 双対性 — 制約の値段 (duality and shadow prices)

ラグランジュ関数を x について最小化した g(λ) = minₓ L(x, λ) を**双対関数**と呼ぶ。任意の λ ≥ 0 で g(λ) は元の問題（主問題）の最適値の下界を与え（弱双対性）、双対問題 max_{λ≥0} g(λ) はその最良の下界を探す。凸問題では（Slater 条件: 厳密に実行可能な点の存在、の下で）**強双対性**——主・双対の最適値が一致——が成り立つ (Boyd & Vandenberghe 2004, Ch. 5)。

双対性の工学的な顔は3つある:

- **最適性の証明書**: 主・双対の目的値の差（双対ギャップ）は「あと最大どれだけ改善しうるか」の上界であり、ソルバの停止判定はこのギャップで行う。**内点法**は主・双対を同時に追い込みながら実行可能領域の内部を進む現代 LP/SOCP の主力であり、双対性はその構造そのものである。
- **影の価格**: 需給一致制約付きの発電コスト最小化（経済負荷配分）を解くと、需要制約の双対変数は「需要をもう 1 MWh 増やしたときの追加コスト」＝**限界費用**になる。送電容量制約まで入れた各母線の双対変数が**地点別限界価格 (LMP)** であり、電力市場は文字どおり最適化問題の双対解を電気の値段として公表している——数学的対象が制度として実装された稀有な例である。
- **分解**: 双対に上がると、結合制約で繋がっていた問題が個別問題に分離することがある（双対分解）。TCP の輻輳制御は、ネットワーク全体の効用最大化 (network utility maximization) を各フローが自律的に解く分散アルゴリズムとして読める (Kelly et al. 1998)——リンクの混雑信号（損失・遅延・ECN）が双対変数（リンクの影の価格）に対応し、「公平性の種類 (proportional fairness など)」は効用関数の選択に対応する。

### 5. 凸化の工学と非凸の現実 (convexification, MPC, and the nonconvex world)

実問題はしばしば非凸で生まれ、実務の腕は**凸への落とし込み**に現れる。教科書的な成功例がロケット着陸誘導の **lossless convexification** (Açıkmeşe & Ploen 2007): 推力の大きさに下限がある制約（0 か 30% 以上——非凸の環）を、緩和した凸制約に置き換えても**最適解は必ず元の制約を満たす**ことを証明した上で SOCP として解く。凸性の保証（大域最適・有限時間収束・実行可能性判定）が、**機上の計算機でリアルタイムに、人命・機体を賭けて**使える品質を与える——「凸なら解ける」が安全性の議論に格上げされる瞬間である。

**モデル予測制御 (MPC)** は最適化を制御器にする様式である: 現在状態から有限ホライズンの軌道最適化（多くは QP）を解き、最初の一手だけ実行し、次の時刻に解き直す。制約（アクチュエータ飽和・安全領域）を明示的に扱える点が古典制御との決定的な差で、化学プラントから自動運転・歩行ロボットまでの標準装備になった。成立条件は「ホライズン内に QP を確実に解き切る」こと——リアルタイム制約と最適化の確実性（凸性・ウォームスタート・反復上限時の実行可能解）がここで直結する。

非凸のまま挑む領域の代表が深層学習である。保証は失われるが、(i) SGD のノイズと過剰パラメータ化が悪い局所解を実質的に回避すること、(ii) 求めるのが「大域最適」ではなく「汎化する良い解」であること、の2点が実務的成功を支える。**凸理論は、その保証がなぜ・どこで破れているかを言語化する基準線**として、非凸の実務にも効き続けている。

## 工学への接続 (Engineering Connections)

- **「解ける」の設計** (§1, §5): 定式化の段階で LP/QP/SOCP に落とせるかどうかが、その機能をリアルタイム系・安全系に載せられるかを決める。凸化は前処理ではなく安全性設計である。
- **条件数という共通言語** (§2): 学習率の調整、特徴量の正規化、前処理行列——「収束が遅い」のかなりの部分は κ の言葉で診断できる。
- **価格＝双対変数** (§4): 電力市場の LMP、スポットインスタンスの価格形成、ネットワークの輻輳信号は、いずれも「制約の希少性を価格に写す」双対性の実装である。市場設計・プロトコル設計を最適化の言葉で読む入口になる。
- **感度解析** (§3–4): 「この制約を緩めたら幾ら得か」は λ を見れば分かる——容量増強・設備投資の優先順位付けは、解そのものより双対変数が答えることが多い。
- **繰り返し解く様式** (§5): MPC・毎5分の経済負荷配分・広告オークションは「同じ形の最適化を高頻度で解き直す」系であり、ウォームスタート・インクリメンタル求解が実装の主戦場になる。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/power-generation/electricity-markets-dispatch.md`](../../../hardware/power-generation/electricity-markets-dispatch.md) | 経済負荷配分＝制約付きコスト最小化、LMP＝需給・送電制約の双対変数 (§3–4) |
| [`hardware/aerospace/avionics-gnc.md`](../../../hardware/aerospace/avionics-gnc.md) | 着陸誘導の lossless convexification と機上 SOCP 求解 (§1, §5) |
| [`hardware/robotics/control-systems.md`](../../../hardware/robotics/control-systems.md) | MPC＝ホライズン付き QP を制御周期ごとに解く様式、制約の明示的取り扱い (§5) |
| [`software/ml-systems/ml-training-systems.md`](../../../software/ml-systems/ml-training-systems.md) | SGD・学習率・条件数と正規化——非凸領域での勾配法の実務 (§2, §5) |
| [`software/networking/tcp-transport-congestion.md`](../../../software/networking/tcp-transport-congestion.md) | 輻輳制御＝効用最大化の双対分解、混雑信号＝リンクの影の価格 (§4) |
| [`hardware/aerospace/orbital-mechanics.md`](../../../hardware/aerospace/orbital-mechanics.md) | 軌道遷移・ランデブーの Δv 最小化——軌道設計を最適制御として読む土台 (§1, §5) |

## 参考文献 (References)

- Boyd, S., & Vandenberghe, L. (2004). *Convex Optimization*. Cambridge University Press. DOI: 10.1017/CBO9780511804441 — 凸最適化の標準教科書。凸性・KKT・双対性の主参照（著者サイトで全文公開）
- Nocedal, J., & Wright, S. J. (2006). *Numerical Optimization* (2nd ed.). Springer. DOI: 10.1007/978-0-387-40065-5 — 数値解法（直線探索、L-BFGS、内点法）の標準教科書
- Kuhn, H. W., & Tucker, A. W. (1951). Nonlinear Programming. *Proceedings of the Second Berkeley Symposium on Mathematical Statistics and Probability*, 481–492. — KKT 条件の原典（Karush 1939 の修士論文が先行）
- Robbins, H., & Monro, S. (1951). A Stochastic Approximation Method. *The Annals of Mathematical Statistics*, 22(3), 400–407. DOI: 10.1214/aoms/1177729586 — 確率的勾配法の原典
- Kelly, F. P., Maulloo, A. K., & Tan, D. K. H. (1998). Rate Control for Communication Networks: Shadow Prices, Proportional Fairness and Stability. *Journal of the Operational Research Society*, 49(3), 237–252. DOI: 10.1057/palgrave.jors.2600523 — 輻輳制御＝効用最大化の双対分解の原典
- Açıkmeşe, B., & Ploen, S. R. (2007). Convex Programming Approach to Powered Descent Guidance for Mars Landing. *Journal of Guidance, Control, and Dynamics*, 30(5), 1353–1366. DOI: 10.2514/1.27553 — lossless convexification の原典
- Rawlings, J. B., Mayne, D. Q., & Diehl, M. (2017). *Model Predictive Control: Theory, Computation, and Design* (2nd ed.). Nob Hill Publishing. — MPC の標準教科書
