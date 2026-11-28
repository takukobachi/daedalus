# ラグランジュ形式と多体系の動力学 (Lagrangian Mechanics and Multibody Dynamics)

## 概要 (Overview)

古典力学シリーズの第2回。第1回のニュートン力学は物体ごとに自由体図を描き、拘束力を含むすべての力をベクトルで足し合わせる方法だった。1つの質点や1つの剛体ならこれで十分だが、6 自由度のロボットアームや脚式ロボットになると、関節ごとの拘束力が未知数として大量に現れ、式を立てるだけで手に余る。**ラグランジュ形式 (Lagrangian mechanics)** は、変数の選び方を変えてこの問題を解く。系の配置を独立な**一般化座標 (generalized coordinates)** q(関節角など)で表し、運動エネルギー T とポテンシャルエネルギー V というスカラー2つから運動方程式を機械的に導く。理想的な拘束力は仕事をしないので、最初から式に現れない。

本回は次の順に進む。(1) 一般化座標と拘束によって、自由度を数え、拘束力を消す。(2) 変分原理 (最小作用の原理) からオイラー=ラグランジュ方程式を導く。(3) 1リンクと2リンクの例で、ニュートン力学と同じ答えが手続き的に得られることを確かめる。(4) 一般の多リンク系で、運動方程式が必ず**マニピュレータ方程式** M(q)q̈ + C(q,q̇)q̇ + g(q) = τ の形になることを示し、慣性行列の正定値性や Ṁ − 2C の歪対称性といった、制御が依拠する構造を導出する。(5) ネーターの定理で保存則と対称性の関係を見たうえで、非ホロノミック拘束と、計算量 O(n) の再帰アルゴリズムを紹介する。

## 前提知識 (Prerequisites)

- [`newton-laws-oscillations.md`](newton-laws-oscillations.md) — 運動方程式、エネルギー保存、剛体回転と慣性モーメント。本回はその一般化である
- [`../../mathematics/calculus-analysis/multivariable-calculus.md`](../../mathematics/calculus-analysis/multivariable-calculus.md) — 偏微分、連鎖律、勾配(∂L/∂q の計算)
- [`../../mathematics/linear-algebra/vectors-matrices-linear-maps.md`](../../mathematics/linear-algebra/vectors-matrices-linear-maps.md) — 行列、2次形式、ヤコビアン
- [`../../mathematics/calculus-analysis/ordinary-differential-equations.md`](../../mathematics/calculus-analysis/ordinary-differential-equations.md) — 2階連立 ODE。マニピュレータ方程式はその一例

## 本論 (Main Content)

### 1. 一般化座標と拘束 (generalized coordinates and constraints)

N 個の質点からなる系は、制約がなければ 3N 個の座標をもつ。剛体のリンクや関節は座標の間に関係式を課す。この**拘束 (constraint)** が位置だけの等式 φ(x, t) = 0 で書けるとき、**ホロノミック拘束 (holonomic constraint)** と呼ぶ。k 本のホロノミック拘束があれば、独立に動ける自由度は n = 3N − k になり、その n 個を一般化座標 q = (q₁, …, q_n) に選べば、拘束は自動的に満たされる (Goldstein et al. 2002, Ch. 1)。

```text
例: 長さ L の振り子 (平面)
  直交座標 (x, y) + 拘束 x² + y² = L²  →  自由度 1、一般化座標 q = θ
例: n 関節の直列リンクロボット
  各リンクは剛体 (6 自由度) だが、各回転関節が 5 自由度を拘束  →  q = (θ₁, …, θ_n)
```

拘束を維持する力(振り子の糸の張力、関節の軸受けの反力)を**拘束力 (constraint force)** という。拘束が滑らかで摩擦がなければ、拘束力は拘束が許す運動の方向(**仮想変位 (virtual displacement)**)に直交し、仕事をしない。これがダランベールの原理で、ラグランジュ形式で拘束力が式から消える理由である。関節トルクのように仕事をする力だけが**一般化力 (generalized force)** Q として残る。

### 2. 最小作用の原理とオイラー=ラグランジュ方程式 (least action and the Euler–Lagrange equations)

**ラグランジアン (Lagrangian)** を L(q, q̇, t) = T − V と定義し、経路に沿った積分を**作用 (action)** S = ∫ L dt と呼ぶ。端点を固定した経路のうち、実際に実現する運動は S を停留させる(**ハミルトンの原理**)。経路を微小に変えたときの S の変化をゼロと置いて部分積分すると、次式が得られる (Goldstein et al. 2002, Ch. 2)。

```text
d/dt (∂L/∂q̇_i) − ∂L/∂q_i = Q_i     (i = 1, …, n)
  Q_i: ポテンシャルで書けない一般化力 (関節トルク τ_i、摩擦など)
  p_i = ∂L/∂q̇_i: 一般化運動量 (q_i が角度なら角運動量)
```

ニュートンの F = ma がベクトルの方程式だったのに対し、これはスカラー L を偏微分する手続きである。座標系を極座標や関節角に変えても式の形は変わらないので、計算者は「T と V を一般化座標で書く」ことだけに集中できる。見かけの力(遠心力・コリオリ力)も T を回転座標で書けば自動的に現れる。

### 3. 例: 振り子と2リンクアーム (examples: pendulum and two-link arm)

単振り子では T = ½ m L² θ̇²、V = −m g L cos θ なので、上式に代入すると m L² θ̈ + m g L sin θ = τ となり、第1回 §5 の振り子方程式に一致する。自由体図も張力の未知数も使っていない。

鉛直面内の2リンクアーム(リンク長 l₁, l₂、質量 m₁, m₂ を各リンク先端の質点で近似)では、リンク2先端の速度に第1関節の回転が足されるため、T に θ̇₁θ̇₂ cos θ₂ の**交差項**が現れる (Spong et al. 2020, Ch. 6)。

```text
T = ½ (m₁+m₂) l₁² θ̇₁² + ½ m₂ l₂² (θ̇₁+θ̇₂)² + m₂ l₁ l₂ cos θ₂ · θ̇₁(θ̇₁+θ̇₂)
慣性行列 M(q):
  M₁₁ = (m₁+m₂) l₁² + m₂ l₂² + 2 m₂ l₁ l₂ cos θ₂
  M₁₂ = M₂₁ = m₂ l₂² + m₂ l₁ l₂ cos θ₂,   M₂₂ = m₂ l₂²
速度2次項: h = m₂ l₁ l₂ sin θ₂
  τ₁ 側: −h (2 θ̇₁θ̇₂ + θ̇₂²)   (コリオリ項 + 遠心項)
  τ₂ 側: +h θ̇₁²              (遠心項)
```

M₁₁ が θ₂ に依存することは、肘を曲げると肩から見た慣性モーメントが小さくなるという直観に対応する。腕を縮めて回るフィギュアスケートと同じ現象である。M₁₂ ≠ 0 は一方の関節を加速すると他方の関節にトルクが伝わる**動的干渉**を表す。高減速比のギアを入れた産業用ロボットでは、モータ側から見た慣性が支配的になってこの干渉が相対的に小さくなるので、関節ごとの独立 PID でも動く。ダイレクトドライブや高速動作ではこの項を無視できない。

### 4. マニピュレータ方程式とその構造 (the manipulator equation and its structure)

一般の n 関節系でも、運動エネルギーは q̇ の2次形式 T = ½ q̇ᵀ M(q) q̇ になり、ポテンシャルは重力だけなら V(q) である。これをオイラー=ラグランジュ方程式に代入すると、運動方程式は必ず次の形にまとまる (Siciliano et al. 2009, Ch. 7)。

```text
M(q) q̈ + C(q, q̇) q̇ + g(q) = τ
  M(q): 慣性行列 (n×n, 対称正定値)
  C(q, q̇) q̇: コリオリ・遠心項 — q̇ の2次
  g(q) = ∂V/∂q: 重力項
C の成分はクリストッフェル記号で与えられる:
  C_kj = Σ_i ½ (∂M_kj/∂q_i + ∂M_ki/∂q_j − ∂M_ij/∂q_k) q̇_i
```

この形から、制御の安定性証明が使う性質が導かれる。

- **M(q) は対称正定値**: 運動エネルギーが運動していれば正だからである。したがって M は常に逆行列をもち、順動力学 q̈ = M⁻¹(τ − Cq̇ − g) が解ける。
- **Ṁ − 2C は歪対称**: 上のクリストッフェル記号で C を選ぶと、任意の x について xᵀ(Ṁ − 2C)x = 0 が成り立つ。これはエネルギー保存の言い換えで、dT/dt = q̇ᵀ(τ − g) から確かめられる。トルク入力から関節速度出力への写像が**受動的 (passive)** であり、エネルギーを生み出さないことを意味し、リアプノフ関数を使った PD + 重力補償の安定性証明の要になる。
- **パラメータについて線形**: M, C, g は質量・慣性モーメント・重心位置といった物理パラメータの1次結合で書ける (τ = Y(q, q̇, q̈) π)。適応制御や、実測データからのパラメータ同定が最小二乗法に落ちるのはこの性質による。

計算トルク法はこの方程式を逆に使い τ = M(q)(q̈_d + K_d ė + K_p e) + C q̇ + g で非線形項を打ち消す。打ち消した後の誤差系は線形2階系になり、第1回 §4 の減衰振動の言葉で設計できる。

### 5. 対称性と保存則、非ホロノミック拘束、計算法 (symmetry, nonholonomic constraints, and algorithms)

L がある一般化座標 q_i を陽に含まないとき(**循環座標 (cyclic coordinate)**)、オイラー=ラグランジュ方程式から dp_i/dt = 0 となり、対応する一般化運動量が保存する。時間並進の対称性からはエネルギー、空間並進からは運動量、回転からは角運動量が保存する。この対応を一般化したものが**ネーターの定理 (Noether's theorem)** で、第1回で「計算の近道」と呼んだ保存則が、どこから来るかを教えてくれる (Goldstein et al. 2002, Ch. 2, 13)。次回の軌道力学で、中心力の回転対称性から角運動量保存とケプラーの第2法則を導くときにも、この考え方を使う。

拘束が速度を含み積分できない場合を**非ホロノミック拘束 (nonholonomic constraint)** という。代表例は横滑りしない車輪で、車両は真横には動けないが、切り返しを組み合わせれば任意の位置と向きに到達できる。速度の自由度は配置の自由度より少ない。このときは一般化座標の数を減らせないので、拘束をラグランジュ乗数 λ で式に加える。縦列駐車の難しさや、差動二輪ロボットの経路計画が特別な扱いを要するのはこのためである。

最後に計算量の問題がある。ラグランジュ法で M, C, g を閉形式に展開すると、項数は自由度とともに急速に増える。実機の制御では、各リンクの速度・加速度を根元から先端へ、力とトルクを先端から根元へ伝える**再帰ニュートン=オイラー法 (recursive Newton–Euler algorithm, RNEA)** が使われ、逆動力学を O(n) で解く。順動力学にも O(n) の**関節体アルゴリズム (articulated-body algorithm)** がある (Featherstone 2008, Ch. 5, 7)。ラグランジュ形式は「方程式の構造を理解し性質を証明する」ための道具、再帰法は「1 kHz の制御周期で数値を出す」ための道具と役割を分けて覚えるとよい。どちらも同じ運動方程式を与える。

## 工学への接続 (Engineering Connections)

- **ロボット制御則の前提**: 重力補償、計算トルク法、インピーダンス制御、全身制御の QP は、すべてマニピュレータ方程式を前提にしている (§4)。重力項 g(q) だけを補償する PD 制御が安定である証明には、Ṁ − 2C の歪対称性(受動性)を使う。
- **モデル同定と適応制御**: パラメータ線形性 τ = Yπ により、ロボットの質量特性を励振軌道の測定データから最小二乗で推定できる (§4)。シミュレータと実機の差を埋める作業の多くはここにある。
- **シミュレータと最適化の計算基盤**: 物理シミュレータや MPC は毎周期順動力学・逆動力学を何百回も解く。O(n) の再帰アルゴリズムと、その解析微分がリアルタイム最適化を可能にしている (§5)。
- **機構設計の指針**: 慣性行列の形は、モータを根元に寄せる(先端リンクを軽くする)、減速比で関節間の干渉を抑えるといった機構設計の判断に直結する (§3)。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`engineering/robotics/kinematics-dynamics.md`](../../../engineering/robotics/kinematics-dynamics.md) | マニピュレータ方程式の導出 (§4)、M の正定値性・Ṁ − 2C の歪対称性・パラメータ線形性、ラグランジュ法とニュートン=オイラー法の使い分け (§5) |
| [`engineering/robotics/control-systems.md`](../../../engineering/robotics/control-systems.md) | 重力補償と計算トルク法 (§4)、高減速比での独立関節 PID (§3) |
| [`engineering/robotics/locomotion-manipulation.md`](../../../engineering/robotics/locomotion-manipulation.md) | 車輪の非ホロノミック拘束 (§5)、脚式ロボットの全身動力学 |
| [`engineering/robotics/humanoid-robots.md`](../../../engineering/robotics/humanoid-robots.md) | 多自由度の全身動力学と浮遊ベース (§4–5) |
| [`engineering/aerospace/avionics-gnc.md`](../../../engineering/aerospace/avionics-gnc.md) | 機体と柔軟モード・スロッシングを含む多体モデル (§1–2) |
| [`foundations/physics/classical-mechanics/central-force-kepler-orbits.md`](central-force-kepler-orbits.md) | 循環座標と角運動量保存 (§5) を中心力問題に適用する次回 |

## 参考文献 (References)

- Goldstein, H., Poole, C., & Safko, J. (2002). *Classical Mechanics* (3rd ed.). Addison-Wesley. — 拘束と一般化座標 (Ch. 1)、ハミルトンの原理とオイラー=ラグランジュ方程式・保存則 (Ch. 2)。§1–2, §5 の主参照
- Spong, M. W., Hutchinson, S., & Vidyasagar, M. (2020). *Robot Modeling and Control* (2nd ed.). Wiley. — 2リンクアームの運動方程式と動力学の性質 (Ch. 6)。§3 の参照
- Siciliano, B., Sciavicco, L., Villani, L., & Oriolo, G. (2009). *Robotics: Modelling, Planning and Control*. Springer. — マニピュレータ方程式の一般形と構造的性質 (Ch. 7)。§4 の主参照
- Featherstone, R. (2008). *Rigid Body Dynamics Algorithms*. Springer. — 再帰ニュートン=オイラー法と関節体アルゴリズム。§5 の計算法の参照
- Lynch, K. M., & Park, F. C. (2017). *Modern Robotics: Mechanics, Planning, and Control*. Cambridge University Press. — ラグランジュ形式とニュートン=オイラー形式の対応、非ホロノミック拘束
