# 状態空間と推定 (State Space & Estimation)

## 概要 (Overview)

制御理論シリーズの第3回 (最終回)。第1回・第2回は、系を入力から出力への伝達関数 1 本で扱った。この見方は1入力1出力の設計には強いが、2つの点で限界がある。第一に、多入力多出力 (MIMO) の系では、どの入力でどの出力を動かすかの組み合わせが増え、ループを1本ずつ整形する方法では扱いきれない。第二に、伝達関数は系の内部を見ない。出力に現れない内部の変数が発散していても、伝達関数からは分からない。

**状態空間 (state space)** の見方は、系の内部の変数 (状態) を明示的に扱うことで、この2つに答える。本回は (1) 状態方程式と伝達関数の関係、(2) 状態を入力で動かせるか (**可制御性**) と出力から知れるか (**可観測性**)、(3) 状態フィードバックによる極配置、(4) 最適制御としての LQR とその保証された安定余裕、(5) 状態を推定する**オブザーバ**と、制御との双対性、(6) 雑音のある系で最適なオブザーバとしての**カルマンフィルタ**、(7) 推定と制御を組み合わせる LQG と分離定理、そして LQG が LQR の余裕を引き継がないことを扱う。

カルマンフィルタの式と非線形拡張 (EKF・UKF・パーティクルフィルタ) の実装は [`../../engineering/robotics/perception-state-estimation.md`](../../engineering/robotics/perception-state-estimation.md) が、ベイズ推定の確率論は [`../mathematics/probability-statistics/estimation-hypothesis-testing.md`](../mathematics/probability-statistics/estimation-hypothesis-testing.md) が扱っている。本回はそれらを制御理論の側から読み、可観測性・定常ゲイン・閉ループの安定余裕との関係に焦点を当てる。

## 前提知識 (Prerequisites)

- [`feedback-transfer-functions.md`](feedback-transfer-functions.md)・[`frequency-response-stability-margins.md`](frequency-response-stability-margins.md) — 伝達関数と極、安定余裕 (シリーズ第1回・第2回)
- [`../mathematics/linear-algebra/eigenvalues-diagonalization.md`](../mathematics/linear-algebra/eigenvalues-diagonalization.md) — 固有値、行列の階数
- [`../mathematics/calculus-analysis/ordinary-differential-equations.md`](../mathematics/calculus-analysis/ordinary-differential-equations.md) — 連立線形系 dx/dt = Ax の安定性
- [`../mathematics/probability-statistics/probability-random-variables.md`](../mathematics/probability-statistics/probability-random-variables.md) — 分散・共分散、正規分布 (§6 で使う)

## 本論 (Main Content)

### 1. 状態方程式 (state equations)

状態 x は、現在の値と今後の入力が分かれば未来がすべて決まる、最小限の変数の組である。線形時不変系は次の形に書ける。

```text
dx/dt = A x + B u     (状態方程式: n 次元の状態 x、m 次元の入力 u)
y     = C x + D u     (出力方程式: p 次元の出力 y)

伝達関数との関係:  G(s) = C (sI − A)⁻¹ B + D
                   G(s) の極 ⊆ A の固有値
```

本回を通して使う例は**2重積分器 (double integrator)** である。力 u で動く質量 1 の物体は、位置 x₁ と速度 x₂ を状態として次のように書ける。

```text
A = [0 1; 0 0],  B = [0; 1],  C = [1 0] (位置を測る)
G(s) = 1/s²       極は s = 0 の重根
```

摩擦のない台車、宇宙空間での衛星の並進や姿勢、慣性航法の加速度から位置への積分は、いずれもこの形をしている。極が原点にあるので、何もしなければ初期速度のまま動き続ける。

伝達関数の極が A の固有値の「部分集合」になっている点が重要である。伝達関数を作るときに極と零点が打ち消し合うと、その固有値は G(s) から消える。消えた固有値が右半面にあれば、出力は落ち着いて見えても内部の状態は発散する。状態空間の見方では、この**隠れた不安定**を A の固有値として直接扱える。

### 2. 可制御性と可観測性 (controllability and observability)

状態を自由に動かせるか、状態を出力から知れるかは、行列の階数で判定できる (Kalman 1960b)。

```text
可制御 (controllable): 任意の初期状態を任意の状態へ入力で移せる
  ⇔ 可制御行列 [B  AB  A²B … Aⁿ⁻¹B] の階数が n

可観測 (observable): 入力と出力の履歴から初期状態を一意に決められる
  ⇔ 可観測行列 [C; CA; CA²; … ; CAⁿ⁻¹] の階数が n
```

2重積分器で確かめる。可制御行列は [B AB] = [0 1; 1 0] で階数 2 なので、力だけで位置と速度の両方を任意に動かせる。可観測性は何を測るかで変わる。位置を測る C = [1 0] では可観測行列が [1 0; 0 1] (階数 2) となり、位置の変化から速度も分かる。一方、速度だけを測る C = [0 1] では可観測行列が [0 1; 0 0] (階数 1) となり、位置は可観測でない。速度をいくら正確に測っても、積分の起点となる初期位置は分からない。慣性航法が加速度と角速度の積分だけでは位置を確定できず、GPS のような位置の観測を必要とする理由は、この階数落ちとして表せる。

伝達関数で極と零点が打ち消し合うのは、系が可制御でないか、可観測でないときに限られる。§1 の「隠れた不安定」は、不安定な固有値が可制御でないか可観測でない部分にある状態である。そうした不安定はどんな制御則でも安定化できない。系全体が可制御・可観測でなくても、不安定な固有値に対応する部分が入力で動かせるなら**安定化可能 (stabilizable)**、出力に現れるなら**検出可能 (detectable)** と呼ぶ。この2つが、§4〜§7 の理論が成り立つための最低条件になる。

### 3. 状態フィードバックと極配置 (state feedback and pole placement)

状態がすべて測れるなら、入力を状態の線形結合 u = −Kx で決められる。閉ループは dx/dt = (A − BK)x となり、極は A − BK の固有値になる。系が可制御であれば、K を選ぶことで閉ループの極を任意の位置に置ける (**極配置, pole placement**)。

```text
2重積分器で K = [k₁ k₂] とすると
  A − BK = [0 1; −k₁ −k₂]
  特性方程式: s² + k₂ s + k₁ = 0

目標の ω_n, ζ を与えれば  k₁ = ω_n²,  k₂ = 2ζω_n
例: ω_n = 2 rad/s, ζ = 0.7  →  K = [4, 2.8]
```

k₁ は位置の誤差に比例するばね、k₂ は速度に比例するダンパとして働く。これは PD 制御と同じ形であり、2次系では極配置と PD 調整が一致する。状態が n 次元になっても、可制御なら n 個の極をすべて指定できる。ただし、極を左に遠く置くほどゲインと入力が大きくなり、アクチュエータの飽和や雑音の増幅という第1回・第2回の制約に行き当たる。極をどこに置くべきかは、極配置そのものからは決まらない。

### 4. LQR — 極の位置を重みで決める (the linear quadratic regulator)

**LQR (linear quadratic regulator)** は、極の位置を直接指定する代わりに、状態のずれと入力の大きさの重みを指定し、その二次コストを最小にする K を求める (Kalman 1960a)。

```text
J = ∫₀^∞ (xᵀQx + uᵀRu) dt    を最小化する  u = −Kx
K = R⁻¹ Bᵀ P,  P は代数リカッチ方程式の正定解:
  AᵀP + PA − PBR⁻¹BᵀP + Q = 0

2重積分器、位置にだけ重み Q = diag(q, 0)、R = 1:
  K = [√q, √(2√q)]
  閉ループ: s² + √2·q^{1/4} s + √q = 0 → ω_n = q^{1/4}, ζ = 1/√2 ≈ 0.707
  q = 1 → K = [1, 1.41],  q = 16 → K = [4, 2.83]
```

重み q を大きくすると速く (ω_n が大きく) なるが、減衰比は q によらず 0.707 に保たれる。重みの比を変えるだけで、振動の少ない応答を保ったまま速さを調整できる。多入力の系でも、設計者は「どの状態のずれをどれだけ嫌うか」を Q と R で書けばよく、ゲイン行列の各要素を個別に決める必要はない。

LQR にはもう一つ重要な性質がある。1入力の LQR は、どんな Q と R を選んでも、**位相余裕 60° 以上、ゲインを 1/2 倍より大きい範囲でいくら変えても安定**という余裕を自動的に持つ。多入力でも、R が対角行列なら各入力のループで同じ余裕が成り立つ (Safonov & Athans 1977)。第2回で見た余裕の不足は、状態がすべて正確に測れる限り、LQR では起きない。

### 5. オブザーバと双対性 (observers and duality)

状態がすべて測れることはまれである。測れない状態は、モデルと測定から推定する。**オブザーバ (observer, 状態観測器)** は、プラントのモデルを計算機上で並走させ、実際の出力とモデルの出力の差で推定値を修正する (Luenberger 1964)。

```text
dx̂/dt = A x̂ + B u + L (y − C x̂)
           モデルの予測      出力の食い違いによる修正

推定誤差 e = x − x̂ の動き:  de/dt = (A − LC) e
```

推定誤差の収束の速さは A − LC の固有値で決まる。系が可観測であれば、L を選んでこの固有値を任意に配置できる。これは §3 の極配置と同じ問題の転置になっている。A − LC の固有値は転置 Aᵀ − CᵀLᵀ の固有値と等しいので、「(A, C) のオブザーバを設計する」ことは「(Aᵀ, Cᵀ) の状態フィードバックを設計する」ことと同じになる。可制御性と可観測性も、同じ転置の関係で対応する。この**双対性 (duality)** のおかげで、制御の設計法はそのまま推定に使える。オブザーバの極は、制御の極より数倍速く置くのが通例である (Franklin et al. 2019)。推定値が先に真値へ追いつかなければ、それを使う制御が正しく働かないためである。

### 6. カルマンフィルタ — 雑音の中の最適なオブザーバ (the Kalman filter)

§5 のオブザーバでは L を自由に選べるが、測定に雑音があると、L の選び方にトレードオフが生じる。L を大きくすると推定は速く真値に追いつくが、測定雑音をそのまま推定値に取り込む。L を小さくすると雑音は抑えられるが、モデル誤差や外乱による状態の変化に追いつかない。**カルマンフィルタ (Kalman filter)** は、プロセス雑音 (分散 Q) と測定雑音 (分散 R) の統計を与えれば、推定誤差の分散を最小にする L を時刻ごとに決める (Kalman 1960c)。式の形は離散時間で「予測」と「更新」を繰り返すもので、[`perception-state-estimation.md`](../../engineering/robotics/perception-state-estimation.md) §2 に示されている。

制御理論から見ると、カルマンフィルタには2つの性質がある。第一に、時間が経つと誤差共分散とゲインは一定値に収束する。この定常ゲインは、§4 の LQR と双対なリカッチ方程式の解から求まり、そのとき定常カルマンフィルタは L を固定した §5 のオブザーバになる。第二に、収束の条件は (A, C) の検出可能性などであり、§2 の構造の性質がそのまま推定の可否を決める。

スカラーの例で定常ゲインの意味を確かめる。ランダムウォークする量 x を雑音のある測定 z で追う。

```text
x_{k+1} = x_k + w_k    (w の分散 Q: 真値が1ステップで動く大きさ)
z_k     = x_k + v_k    (v の分散 R: 測定雑音)

定常の予測誤差分散 P⁻ = (Q + √(Q² + 4QR)) / 2,  定常ゲイン K = P⁻ / (P⁻ + R)
推定の更新:  x̂_k = x̂_{k−1} + K (z_k − x̂_{k−1})   (指数平滑と同じ形)

Q = 1, R = 100 → K ≈ 0.095  (測定を 1 割弱だけ取り入れ、強く平滑化する)
Q = 1, R = 1   → K ≈ 0.618  (測定を 6 割取り入れ、変化に素早く追従する)
```

定常カルマンフィルタは、この例では平滑化係数を K とする指数平滑 (EWMA) に一致する。経験で決めがちな平滑化係数が、「真値がどれだけ動くか」と「測定がどれだけ揺れるか」の比 Q/R から決まる。逆に言えば、指数平滑を使っている観測系は、暗にランダムウォークのモデルと特定の Q/R を仮定している。

カルマンフィルタは、線形かつ雑音が正規分布に従う場合に最小分散の推定を与える。雑音が正規分布でなくても、線形な推定器の中では誤差の分散が最小になる (Simon 2006)。非線形の系では線形化 (EKF) や他の近似が必要になり、最適性の保証は失われる。実用の最初期の例は Apollo 計画である。Kalman の論文の直後に NASA Ames の Schmidt らがその有用性に気づき、非線形な航法方程式への適用と、搭載計算機の短い語長に耐える平方根形式の開発を進めた (McGee & Schmidt 1985)。

### 7. LQG と分離定理 — 余裕は引き継がれない (LQG, separation, and lost margins)

状態を測れないとき、カルマンフィルタで推定した x̂ を LQR のゲインに入れて u = −K x̂ とする構成を **LQG (linear quadratic Gaussian)** 制御と呼ぶ。

```text
プラント:   dx/dt = Ax + Bu + w,   y = Cx + v
制御器:     u = −K x̂               (K は §4 の LQR ゲイン)
推定器:     dx̂/dt = Ax̂ + Bu + L(y − Cx̂)   (L は §6 のカルマンゲイン)

閉ループの極 = {A − BK の固有値} ∪ {A − LC の固有値}
```

閉ループの極は、制御側の極と推定側の極を合わせたものになる。このため K と L は互いに独立に設計できる。これが**分離定理 (separation principle)** であり、雑音が正規分布に従う線形系では、この組み合わせが確率的な二次コストの意味でも最適になる (Anderson & Moore 1990)。

ただし、分離定理が保証するのは公称モデルでの安定性と最適性であり、安定余裕ではない。§4 の LQR の余裕 (位相余裕 60° 以上) は、状態を正確に測れるという前提に依存している。推定器を挟んだ LQG にはこの保証がない。Doyle (1978) はこれを反例で示し、論文の要旨は「保証された余裕はあるか」という問いへの答えとして「There are none.」の一文だけだった。カルマンフィルタを重く効かせた LQG は、モデルがわずかにずれるだけで不安定になりうる。その後、推定器を挟んだまま LQR のループ特性に近づける**ループ伝達回復 (loop transfer recovery, LTR)** や、モデル誤差を直接扱う **H∞ 制御**などのロバスト制御が発展した。最適性と頑健さは別の性質であり、第2回のループ整形と安定余裕の確認は、状態空間で設計した制御器にも欠かせない。

## 工学への接続 (Engineering Connections)

- **慣性航法と GNSS の統合**: 加速度と角速度の積分だけでは位置が可観測にならず (§2)、誤差が際限なく溜まる。GNSS の位置を測定としてカルマンフィルタに入れると、位置と速度だけでなく、ジャイロや加速度計のバイアスまで状態として推定できる。
- **バッテリの充電状態 (SoC) 推定**: SoC は直接測れない状態量で、電流の積分 (クーロンカウンティング) は誤差が溜まる。端子電圧を測定とするオブザーバやカルマンフィルタで補正する構造は、慣性航法と GNSS の統合と同じ形をしている。
- **観測の故障検出**: カルマンフィルタのイノベーション (測定と予測の差) は、モデルが正しければ既知の分散を持つ。統計的にありえない大きさのイノベーションは、センサの故障や外乱の兆候として検出できる。推定器が故障検出を兼ねるのは航空宇宙の定石である。
- **ソフトウェアの平滑化と予測**: メトリクスの指数平滑、オートスケーラの負荷予測、時刻同期でのクロックのドリフト推定は、§6 のスカラー例と同じ構造を持つ。Q/R の比を明示すれば、平滑化係数を根拠をもって選べる。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`engineering/robotics/perception-state-estimation.md`](../../engineering/robotics/perception-state-estimation.md) | カルマンフィルタの式と EKF・UKF の実装。本回 §6 はその制御理論側の読み方 |
| [`engineering/robotics/control-systems.md`](../../engineering/robotics/control-systems.md) | 状態空間制御と LQR (§4)、観測器と組み合わせた LQG (§7) |
| [`engineering/aerospace/avionics-gnc.md`](../../engineering/aerospace/avionics-gnc.md) | 慣性航法のドリフトとカルマンフィルタによる融合 (§2, §6)、イノベーションによる故障検出 |
| [`engineering/robotics/robotics-fundamentals.md`](../../engineering/robotics/robotics-fundamentals.md) | IMU の積分ドリフトと、エンコーダ・視覚との融合の必要性 (§2) |
| [`engineering/automotive/adas-sensing-perception.md`](../../engineering/automotive/adas-sensing-perception.md) | 複数センサの融合と物体追跡の状態推定 (§6) |
| [`engineering/energy/battery-energy-storage.md`](../../engineering/energy/battery-energy-storage.md) | 直接測れない SoC/SoH のカルマンフィルタ系推定 (§6) |
| [`engineering/energy/plant-control-scada-security.md`](../../engineering/energy/plant-control-scada-security.md) | EMS の状態推定と、改ざんされた測定が推定に与える影響 |

## 参考文献 (References)

- Åström, K. J., & Murray, R. M. (2021). *Feedback Systems: An Introduction for Scientists and Engineers* (2nd ed.). Princeton University Press. — 状態空間、可制御性・可観測性、状態フィードバックとオブザーバの主参照
- Franklin, G. F., Powell, J. D., & Emami-Naeini, A. (2019). *Feedback Control of Dynamic Systems* (8th ed.). Pearson. — 極配置とオブザーバの極の選び方
- Anderson, B. D. O., & Moore, J. B. (1990). *Optimal Control: Linear Quadratic Methods*. Prentice Hall. — LQR、リカッチ方程式、LQG と分離定理
- Simon, D. (2006). *Optimal State Estimation: Kalman, H∞, and Nonlinear Approaches*. Wiley. — カルマンフィルタの導出と性質、非線形拡張
- Kalman, R. E. (1960a). Contributions to the Theory of Optimal Control. *Boletín de la Sociedad Matemática Mexicana*, 5, 102–119. — LQR とリカッチ方程式の原典
- Kalman, R. E. (1960b). On the General Theory of Control Systems. *Proceedings of the First IFAC Congress*, Moscow, 1, 481–492. — 可制御性・可観測性の概念の導入
- Kalman, R. E. (1960c). A New Approach to Linear Filtering and Prediction Problems. *Journal of Basic Engineering*, 82(1), 35–45. DOI: 10.1115/1.3662552 — カルマンフィルタの原典
- Luenberger, D. G. (1964). Observing the State of a Linear System. *IEEE Transactions on Military Electronics*, 8(2), 74–80. DOI: 10.1109/TME.1964.4323124 — オブザーバの原典
- Safonov, M. G., & Athans, M. (1977). Gain and Phase Margin for Multiloop LQG Regulators. *IEEE Transactions on Automatic Control*, 22(2), 173–179. — LQR の保証された安定余裕
- Doyle, J. C. (1978). Guaranteed Margins for LQG Regulators. *IEEE Transactions on Automatic Control*, 23(4), 756–757. — LQG に保証された余裕がないことの反例
- McGee, L. A., & Schmidt, S. F. (1985). *Discovery of the Kalman Filter as a Practical Tool for Aerospace and Industry* (NASA TM-86847). NASA Ames Research Center. — Apollo 航法への適用と平方根形式の経緯
