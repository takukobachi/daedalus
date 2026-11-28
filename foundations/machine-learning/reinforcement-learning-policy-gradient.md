# 強化学習の基礎: MDP から PPO・RLHF まで (Reinforcement Learning: From MDPs to PPO and RLHF)

## 概要 (Overview)

機械学習系列の第3回。Transformer の回 ([`transformer-architecture.md`](transformer-architecture.md)) とスケーリング則の回 ([`scaling-laws.md`](scaling-laws.md)) は、正解のトークン列を与えて交差エントロピーを下げる**教師あり学習**を扱った。しかし、指示に従う LLM の整形 (RLHF)、推論モデルの思考の訓練、脚式ロボットの歩行制御では、正解の行動が与えられない。与えられるのは、行動の結果としての**報酬 (reward)** だけである。行動を選び、結果を見て、より多くの報酬が得られるように方策を改善する。この枠組みが**強化学習 (reinforcement learning, RL)** である。

本回は次の順に進む。(1) 問題を**マルコフ決定過程 (MDP)** として定式化する。(2) 価値関数と**ベルマン方程式**を導き、最適制御・動的計画法と同じ構造であることを確認する。(3) モデルを知らずにサンプルから学ぶ TD 学習と Q 学習、それを深層ネットワークで近似した DQN を扱う。(4) 方策を直接微分する**方策勾配定理**と、その分散を下げるベースライン・アドバンテージ・GAE を導く。(5) 更新幅を制限する TRPO と、その実用版である **PPO** のクリップ付き目的関数を導出する。(6) 最後に **RLHF** を、報酬モデル(ブラッドリー=テリーモデル)と KL 正則化付きの方策最適化として定式化する。この式から DPO が閉形式で導けること、報酬ハッキングが構造的に起きることも示す。あわせて、検証可能な報酬を使う推論モデルの強化学習 (GRPO) にも触れる。

## 前提知識 (Prerequisites)

- [`../mathematics/probability-statistics/markov-chains-queueing.md`](../mathematics/probability-statistics/markov-chains-queueing.md) — マルコフ連鎖と遷移確率。MDP はこれに行動と報酬を加えたもの
- [`../mathematics/probability-statistics/probability-random-variables.md`](../mathematics/probability-statistics/probability-random-variables.md) — 期待値・分散。方策勾配は期待値の勾配の推定である
- [`../mathematics/optimization/convex-optimization-duality.md`](../mathematics/optimization/convex-optimization-duality.md) — 勾配法、制約付き最適化とラグランジュ乗数(TRPO と KL 制約で使う)
- [`../information-theory/entropy-source-coding.md`](../information-theory/entropy-source-coding.md) — エントロピーと KL ダイバージェンス
- [`transformer-architecture.md`](transformer-architecture.md) — 方策としての言語モデル (§6 で使う)

## 本論 (Main Content)

### 1. マルコフ決定過程 (Markov decision processes)

エージェントは各時刻 t に状態 s_t を観測し、行動 a_t を選ぶ。環境は次の状態 s_{t+1} と報酬 r_t を返す。次の状態の分布が現在の状態と行動だけで決まる(マルコフ性)とき、この問題を MDP と呼ぶ (Sutton & Barto 2018, Ch. 3)。

```text
MDP = (S, A, P, R, γ)
  P(s' | s, a): 遷移確率,  R(s, a): 報酬,  γ ∈ [0, 1): 割引率
方策 π(a | s): 状態から行動の確率分布への写像
収益 (return): G_t = r_t + γ r_{t+1} + γ² r_{t+2} + …
目的: J(π) = E_π[G_0] を最大化する π を見つける
```

割引率 γ は遠い将来の報酬ほど軽く数えるための係数で、無限和を有限に保つ役割と、実効的な計画の長さ(おおむね 1/(1 − γ) ステップ)を決める役割をもつ。教師あり学習との違いは三つある。第一に、正解の行動ではなく評価だけが与えられる。第二に、報酬が行動のずっと後に届くことがあり、どの行動の手柄かを割り振る**信用割り当て (credit assignment)** が必要になる。第三に、エージェント自身の行動がデータの分布を変えるので、まだ試していない行動を試す**探索 (exploration)** と、分かっている良い行動を使う**活用 (exploitation)** の交換関係が生じる。

### 2. 価値関数とベルマン方程式 (value functions and the Bellman equations)

状態の良さを、そこから方策 π に従ったときの期待収益で測る。収益の再帰構造 G_t = r_t + γ G_{t+1} から、価値関数は1ステップ先の価値で書ける (Bellman 1957; Sutton & Barto 2018, Ch. 3–4)。

```text
状態価値:     V^π(s) = E_π[G_t | s_t = s]
行動価値:     Q^π(s, a) = E_π[G_t | s_t = s, a_t = a]
ベルマン期待方程式:  V^π(s) = Σ_a π(a|s) [R(s,a) + γ Σ_{s'} P(s'|s,a) V^π(s')]
ベルマン最適方程式:  Q*(s, a) = R(s,a) + γ Σ_{s'} P(s'|s,a) max_{a'} Q*(s', a')
最適方策: π*(s) = argmax_a Q*(s, a)
```

右辺の演算子(ベルマン作用素)は γ を係数とする縮小写像なので、任意の初期値から繰り返し適用すると一意の不動点に収束する。これが**価値反復 (value iteration)** で、モデル P, R が既知なら動的計画法で最適方策が求まる。状態が連続で、ダイナミクスが線形、コストが2次という特別な場合には、ベルマン方程式はリカッチ方程式になり、その解が LQR のゲインになる。最適制御と強化学習は同じ方程式を、モデルが既知か未知かという異なる前提で解いている。

### 3. サンプルから学ぶ: TD 学習、Q 学習、DQN (learning from samples)

モデルが未知なら、期待値をサンプルで置き換える。観測した1ステップ (s, a, r, s') を使って、現在の推定値を「1ステップ先の推定値で作った目標」に近づける。これを**時間差分 (temporal-difference, TD) 学習**という (Sutton & Barto 2018, Ch. 6)。

```text
TD 誤差:  δ = r + γ V(s') − V(s)
Q 学習:   Q(s, a) ← Q(s, a) + α [r + γ max_{a'} Q(s', a') − Q(s, a)]
```

Q 学習はデータを集めた方策とは無関係に最適行動価値 Q* を学ぶ(**方策オフ型 (off-policy)**)。Mnih ら (2015) の **DQN** は Q を畳み込みネットワークで近似し、Atari の多数のゲームを画面画像から学習した。関数近似・ブートストラップ(推定値で推定値を更新すること)・方策オフ学習の3つを組み合わせると学習が発散しうるため、DQN は過去の遷移をためて無作為に再利用する**経験再生 (experience replay)** と、目標側の Q を一定期間固定する**ターゲットネットワーク**でこれを抑えた。価値ベースの手法は max_a が計算できる離散行動に向いており、連続の関節トルクや数万語彙のトークン列では、次節の方策勾配法が主役になる。

### 4. 方策勾配定理とアドバンテージ (the policy gradient theorem and advantages)

方策をパラメータ θ のネットワーク π_θ で表し、J(θ) を勾配上昇で直接最大化する。軌道 τ の確率は π_θ と環境の遷移の積だが、log をとって θ で微分すると環境の項が消える。このため、環境のモデルを知らなくても勾配を推定できる (Williams 1992; Sutton et al. 2000)。

```text
方策勾配定理:  ∇_θ J = E_π[ Σ_t ∇_θ log π_θ(a_t|s_t) · Ψ_t ]
  REINFORCE:     Ψ_t = G_t           (不偏だが分散が大きい)
  ベースライン:   Ψ_t = G_t − b(s_t)   (b が行動に依存しなければ不偏性を保つ)
  アドバンテージ: Ψ_t = A^π(s_t, a_t) = Q^π(s_t, a_t) − V^π(s_t)
GAE:  Â_t = Σ_l (γλ)^l δ_{t+l},  λ ∈ [0, 1] でバイアスと分散を調整
```

∇ log π · Ψ は、「良い結果につながった行動の確率を上げ、悪い結果につながった行動の確率を下げる」という操作である。最尤推定の勾配に重み Ψ を掛けた形をしているので、教師あり学習の損失を「報酬で重みづけした対数尤度」に置き換えたものと読める。問題は分散の大きさで、収益 G_t は多数の確率的な選択の積み重ねなので、そのまま使うと勾配の推定がひどく揺れる。平均的な結果 V(s) を差し引いた**アドバンテージ**は「この行動は平均よりどれだけ良かったか」だけを残し、分散を大きく下げる。V を別のネットワーク(**価値関数 (critic)**)で学習し、方策 (actor) の勾配に使う構成を**アクター・クリティック (actor-critic)** と呼ぶ。Schulman ら (2016) の **GAE (generalized advantage estimation)** は TD 誤差の指数加重和で Â を作り、λ = 0 で1ステップ TD(低分散・高バイアス)、λ = 1 でモンテカルロ収益(高分散・不偏)に一致する。

### 5. 信頼領域と PPO (trust regions and PPO)

方策勾配には、1回の更新で方策が大きく変わると、集めたデータがもはや新しい方策の分布を代表しなくなり、性能が崩れるという問題がある。**TRPO** は旧方策との KL ダイバージェンスに上限 δ を課した制約付き最適化として更新を定式化した (Schulman et al. 2015)。**PPO (proximal policy optimization)** は、同じ目的を2次最適化なしに、クリップした代理目的関数で実現する (Schulman et al. 2017)。

```text
確率比:  ρ_t(θ) = π_θ(a_t|s_t) / π_θ_old(a_t|s_t)
PPO-Clip:  L(θ) = E_t[ min( ρ_t Â_t,  clip(ρ_t, 1−ε, 1+ε) Â_t ) ]
  Â_t > 0: ρ_t が 1+ε を超えても得をしない — 良い行動の確率を上げすぎない
  Â_t < 0: ρ_t が 1−ε を下回っても得をしない — 悪い行動の確率を下げすぎない
  原論文の連続制御実験では ε = 0.2 が最も良かった
全体の損失: −L(θ) + c₁ (V_φ − V_target)² − c₂ H[π_θ]  (価値の回帰 + エントロピー正則化)
```

min をとることで、L は本来の代理目的の**悲観的な下界**になる。方策が旧方策から ε 以上離れる方向には勾配が流れないので、同じデータで複数エポックのミニバッチ更新を安全に繰り返せる。実装が簡単で、ハイパーパラメータに比較的鈍感なことから、PPO はロボットの sim-to-real 学習でも、LLM の RLHF でも標準的な手法になった。

### 6. RLHF: 選好からの報酬モデルと KL 正則化 (RLHF: reward models and KL regularization)

言語モデルを方策とみなすと、状態はプロンプトとそれまでの生成トークン、行動は次のトークン、エピソードは1つの応答になる。ただし「良い応答」の報酬関数は書き下せない。**RLHF (reinforcement learning from human feedback)** は、人間に応答のペアを比べさせて報酬モデル r_φ を学習し、それを報酬として方策を最適化する (Christiano et al. 2017; Ouyang et al. 2022)。

```text
報酬モデル (ブラッドリー=テリー):  P(y_w ≻ y_l | x) = σ( r_φ(x, y_w) − r_φ(x, y_l) )
  損失: −E[ log σ(r_φ(x, y_w) − r_φ(x, y_l)) ]  — ペア比較のロジスティック回帰
方策最適化 (KL 正則化):
  max_π E_{x, y~π}[ r_φ(x, y) ] − β · KL( π(·|x) ‖ π_ref(·|x) )
  π_ref: SFT 済みモデル。実装では報酬を r_φ − β log(π/π_ref) に置き換えて PPO で解く
閉形式の最適解:  π*(y|x) ∝ π_ref(y|x) · exp( r_φ(x, y) / β )
```

KL 項は、方策が参照モデルから離れすぎないようにするための正則化である。報酬モデルは有限の比較データで学習した近似にすぎないので、方策がそれを最大化しすぎると、本当の選好ではなく報酬モデルの誤りを突く応答(**報酬ハッキング (reward hacking)**)が出てくる。Gao ら (2023) は、代理報酬を最適化し続けると、真の報酬(ゴールド報酬モデル)が途中で頭打ちになり、その後は下がることを、KL 距離の関数として定量化した。InstructGPT (Ouyang et al. 2022) では、この手順で訓練した 1.3B パラメータのモデルの出力が、175B の GPT-3 より人間に好まれた。一方で、評価者に好まれる応答への最適化は、同意を優先する追従性のような副作用も持ち込む。

閉形式解を逆に解くと、報酬は r = β log(π*/π_ref) + (x だけに依存する項) と方策で表せる。これをブラッドリー=テリーの損失に代入すると x だけの項は打ち消され、報酬モデルも PPO も使わずに選好データから方策を直接学習する損失が得られる。これが **DPO (direct preference optimization)** である (Rafailov et al. 2023)。

数学の答えやコードのテストのように正誤を機械的に判定できる課題では、報酬モデルの代わりに**検証可能な報酬 (verifiable reward)** を使える。DeepSeekMath で提案された **GRPO (group relative policy optimization)** は、同じプロンプトから複数の応答を生成し、そのグループ内の報酬の平均と標準偏差で正規化した値をアドバンテージとして使う。これにより価値関数ネットワークを省く (Shao et al. 2024)。DeepSeek-R1 はこの方式の強化学習で、人間が書いた推論過程の例なしに長い思考連鎖を引き出せることを示した (Guo et al. 2025)。報酬ハッキングの危険は正誤判定の抜け穴(テストを書き換えるコードなど)という形で残る。

## 工学への接続 (Engineering Connections)

- **LLM の事後学習パイプライン**: SFT → 報酬モデル → PPO/DPO/GRPO という段階は、§4–6 の式をそのまま実装したものである。KL 係数 β、クリップ幅 ε、グループサイズは、整合と多様性・安定性の交換関係を調整するつまみになる。
- **推論モデルとテスト時計算**: 長い思考を報酬で強化することで、推論時に多くのトークンを費やすほど正答率が上がる振る舞いが得られる (§6)。スケーリング則の回で扱った「テスト時計算」の軸を、訓練側から作り出しているのが強化学習である。
- **ロボットの学習ベース制御**: GPU 上で数千の環境を並列に動かし、PPO で歩行方策を学習してドメインランダマイゼーションで実機に移す手法が、脚式ロボットの標準的な製法になった (§5)。古典制御のような安定余裕の保証がないため、学習方策の外側に安全層を置く構成が採られる。
- **報酬設計は仕様記述の問題である**: 報酬ハッキング (§6) は、最適化を強めるほど仕様の穴が見つかりやすくなることを示している。報酬・評価指標の設計は、セキュリティの脅威モデルと同様に「最適化する相手が抜け穴を探す」前提で行う必要がある。

## Daedalus内の応用 (Applications within Daedalus)

- [`../../software/ai/llm-internals-prompting.md`](../../software/ai/llm-internals-prompting.md) — §4 の学習パイプライン (SFT/RLHF/DPO) の理論的基礎 (§6)。追従性が KL 正則化付きの選好最適化の副作用である理由
- [`../../software/ai/ml-training-systems.md`](../../software/ai/ml-training-systems.md) — 分散学習の実行基盤。PPO による RLHF は方策・参照・報酬・価値の4モデルを同時に保持し、生成と学習を交互に行う (§5–6) ため、同書のメモリ・計算の会計がさらに重くなる
- [`../../software/ai/llm-evaluation-methodology.md`](../../software/ai/llm-evaluation-methodology.md) — グッドハートの法則と、判定器を報酬に使う場合の報酬ハッキング (§6)
- [`../../engineering/robotics/locomotion-manipulation.md`](../../engineering/robotics/locomotion-manipulation.md) — 不整地歩行の強化学習方策 (§5)
- [`../../engineering/robotics/robot-software-stack.md`](../../engineering/robotics/robot-software-stack.md) — 大規模並列シミュレーションでの PPO 学習と sim-to-real (§5)
- [`../../engineering/robotics/control-systems.md`](../../engineering/robotics/control-systems.md) — ベルマン方程式と LQR・MPC の対応 (§2)、学習方策の検証問題
- [`scaling-laws.md`](scaling-laws.md) — テスト時計算の第二のスケーリング軸と、それを訓練する強化学習 (§6)

## 参考文献 (References)

- Sutton, R. S., & Barto, A. G. (2018). *Reinforcement Learning: An Introduction* (2nd ed.). MIT Press. — MDP とベルマン方程式 (Ch. 3–4)、TD 学習と Q 学習 (Ch. 6)、方策勾配 (Ch. 13)。§1–4 の主参照
- Bellman, R. (1957). *Dynamic Programming*. Princeton University Press.
- Williams, R. J. (1992). Simple Statistical Gradient-Following Algorithms for Connectionist Reinforcement Learning. *Machine Learning*, 8, 229–256. DOI: 10.1007/BF00992696
- Sutton, R. S., McAllester, D., Singh, S., & Mansour, Y. (2000). Policy Gradient Methods for Reinforcement Learning with Function Approximation. *NeurIPS 1999*, 1057–1063.
- Mnih, V. et al. (2015). Human-level control through deep reinforcement learning. *Nature*, 518, 529–533. DOI: 10.1038/nature14236
- Schulman, J., Levine, S., Moritz, P., Jordan, M., & Abbeel, P. (2015). Trust Region Policy Optimization. *ICML 2015*. arXiv:1502.05477
- Schulman, J., Moritz, P., Levine, S., Jordan, M., & Abbeel, P. (2016). High-Dimensional Continuous Control Using Generalized Advantage Estimation. *ICLR 2016*. arXiv:1506.02438
- Schulman, J., Wolski, F., Dhariwal, P., Radford, A., & Klimov, O. (2017). Proximal Policy Optimization Algorithms. arXiv:1707.06347
- Christiano, P. et al. (2017). Deep Reinforcement Learning from Human Preferences. *NeurIPS 2017*. arXiv:1706.03741
- Ouyang, L. et al. (2022). Training language models to follow instructions with human feedback. *NeurIPS 2022*. arXiv:2203.02155
- Gao, L., Schulman, J., & Hilton, J. (2023). Scaling Laws for Reward Model Overoptimization. *ICML 2023*, PMLR 202, 10835–10866. arXiv:2210.10760
- Rafailov, R. et al. (2023). Direct Preference Optimization: Your Language Model is Secretly a Reward Model. *NeurIPS 2023*. arXiv:2305.18290
- Shao, Z. et al. (2024). DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models. arXiv:2402.03300
- Guo, D. et al. (2025). DeepSeek-R1 incentivizes reasoning in LLMs through reinforcement learning. *Nature*, 645, 633–638. DOI: 10.1038/s41586-025-09422-z
