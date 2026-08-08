# スケーリング則 (Scaling Laws)

## 概要 (Overview)

LLM の性能向上を駆動してきた最大の経験則は、**損失がモデルサイズ・データ量・計算量のべき則 (power law) に従って滑らかに下がる**というスケーリング則である。この一見単純な曲線が、「次のモデルにいくら投じるべきか」「パラメータとデータをどう配分するか」「小さいモデルを長く訓練する意味は何か」という数十億ドル規模の意思決定を支配し、データセンター建設（`../../hardware/datacenter-infrastructure/`）までを駆動している。

本ドキュメントは、べき則の形とその含意、計算最適学習 (compute-optimal training; Chinchilla)、推論コストを含めた総費用の最適化、「創発」をめぐる論争、そしてテスト時計算 (test-time compute) という第二のスケーリング軸を、原論文に基づいて整理する。`transformer-architecture.md` の会計式 C ≈ 6ND を前提とする。

## 前提知識 (Prerequisites)

- Transformer の構成とパラメータ・FLOPs 会計（`transformer-architecture.md` §7）
- 交差エントロピー損失＝符号長の解釈（同 §6、`../information-theory/entropy-source-coding.md`）
- べき則・対数スケールでの直線、制約付き最小化の初歩（`../mathematics/optimization/convex-optimization-duality.md` のラグランジュの言語）

記法: N = パラメータ数、D = 学習トークン数、C = 学習計算量 [FLOPs]、L = テスト損失 [nats/token]。

## 本論 (Main Content)

### 1. べき則の形 — 3 つの資源と損失

Kaplan et al. (2020) は、他の資源を十分に与えたとき、損失が各資源の単独べき則に従うことを広範囲（数桁）で示した:

```text
L(N) ≈ (N_c/N)^α_N        L(D) ≈ (D_c/D)^α_D        L(C) ≈ (C_c/C)^α_C
```

指数 α はいずれも 0.05〜0.1 程度の小さい値であり、**性能改善は資源の乗数投入に対して対数的にしか進まない**——損失を一定量下げるコストは指数的に増える——ことを意味する。重要な性質が 3 つある:

- **滑らかさ**: 数桁にわたり折れ目がない。これが「次の 10 倍」への投資を予測可能な賭けに変え、以後のスケール競争の根拠となった。
- **アーキテクチャ細部への鈍感さ**: 幅・深さ等の形状因子は、総パラメータ数 N を固定すれば損失に対して二次的である。「大きさが主、形は従」。
- **既約項**: より一般の形は L = E + (べき則項) であり、E はデータ自身のエントロピー（どれだけ計算しても消えない項; `transformer-architecture.md` §6 の符号長解釈の直接の帰結）。

### 2. 計算最適学習 — Chinchilla の配分則

計算予算 C ≈ 6ND を固定したとき、N と D をどう配分すべきか。Hoffmann et al. (2022) は損失を 2 変数の和で当てはめ:

```text
L(N, D) = E + A/N^α + B/D^β        （当てはめ値: α ≈ 0.34, β ≈ 0.28）
```

制約 C = 6ND の下で L を最小化すると（ラグランジュ乗数、または D = C/6N を代入して微分）、α ≈ β のとき

```text
N_opt ∝ C^0.5,   D_opt ∝ C^0.5   →  D/N はほぼ一定（実測 ≈ 20 トークン/パラメータ）
```

つまり**予算を 100 倍にしたら、モデルを 10 倍・データを 10 倍にせよ**。これは Kaplan らの当初の結論（N 側に大きく傾けよ）を覆した。差の主因は実験計画にあり、Kaplan らは学習率スケジュールを学習長に合わせて調整していなかったため、データ追加の効果を過小評価していた——**経験則は測定手続きの関数である**ことの教訓でもある。Chinchilla (70B, 1.4T tokens) が 4 倍大きい Gopher (280B) を同一計算量で上回った実証が、この配分則を定着させた。

### 3. 推論を含む総費用 — 「過訓練」の経済学

Chinchilla 最適は**学習費用だけ**の最適化である。モデルは学習後に何兆トークンも推論されるため、総費用は

```text
Cost_total = 6ND (学習)  +  2N × D_inference (推論)
```

推論量 D_inference が大きいほど、**N を小さく保つ価値が増す**。小さい N で同じ損失に届くには D を Chinchilla 比で大きく超えて訓練する（べき則の裾を登る）必要があり、学習費用は最適点より割高になるが、その差は推論で回収される。Llama 系（例: 8B に 15T tokens ≈ 1,900 tokens/param）に代表される「小さく・長く」の潮流はこの計算の帰結であり (Touvron et al., 2023; Sardana et al., 2024)、エッジ・オンデバイス推論への圧力がそれを加速している。**「Chinchilla 最適」と「デプロイ最適」は別の点である**。

### 4. データ制約とその克服

D ∝ C^0.5 で伸ばし続けると、高品質テキストの総量が制約になる（枯渇時期の推定は仮定依存で幅が大きい; Villalobos et al., 2024）。応答は 3 系統ある: (a) **反復**——同じデータの複数エポックは約 4 周までは新規データにほぼ等価で、以後急速に逓減する (Muennighoff et al., 2023)、(b) **品質側の改善**——フィルタリング・重複除去は実効的なデータ効率を上げる、(c) **合成データ**——検証可能な領域（数学・コード）では自己生成＋検証のループが機能しやすい。いずれも「D の壁」をべき則の外側から動かす工学である。

### 5. 創発をめぐる論争 — 何が本当に不連続か

Wei et al. (2022) は、算術・多段推論などの下流タスク性能がある規模で急に立ち上がる「創発的能力 (emergent abilities)」を報告した。Schaeffer et al. (2023) はその多くが**指標の人工物**であると反論した——完全一致 (exact match) のような不連続指標は、トークンごとの正解率が滑らかに上がっていても、全トークン正解の確率 p^k がしきい値を跨ぐ瞬間だけ急変して見える。連続指標（対数尤度）で測り直すと多くのタスクは滑らかである。

実務的な整理: **土台の損失は滑らかに、体感の能力は段差を持って**現れる。「あるタスクが規模でいつ解けるようになるか」の予測は、損失の予測（べき則で高精度）よりはるかに難しい。能力予測をしたい場合は、不連続指標の背後にある連続量（正解トークンの対数尤度、pass@k の k 依存性）を測るのが正しい計器の選び方である。

### 6. テスト時計算 — 第二のスケーリング軸

学習時の C だけでなく、**推論時に費やす計算**も性能の軸である。同一モデルでも、思考トークンの延長・複数サンプル生成と検証器による選別・探索により、下流性能は推論計算に対してべき則的に改善しうる (Snell et al., 2024)。強化学習で思考の質自体を訓練した推論モデル（o1 系、拡張思考）は、この軸を製品パラメータ（思考予算）に変えた。

含意は配分問題の再来である: 固定の総予算の下で「大きいモデルを短く考えさせる」か「小さいモデルを長く考えさせるか」はタスク難易度依存のトレードオフになり、学習時スケーリング（§2–3）とテスト時スケーリングは代替と補完の関係に立つ。`../../software/ai-llm/llm-internals-prompting.md` §2 の「思考の外部化」、および pass@k と選別器の関係（`../../software/ai-llm/agent-architectures.md` 技術詳細 3）は、この軸の挙動レベル・エージェントレベルの現れである。

## 工学への接続 (Engineering Connections)

- **投資判断の計器盤。** べき則の滑らかさが「次の 10 倍」の性能を事前に見積り可能にし、スケール競争と AI インフラ投資（GPU・データセンター・電力）の合理的根拠を与えた。逆に指数 α の小ささは、性能の線形改善に指数的コストがかかることを警告する。
- **モデルサイズの選定は総費用最適化である。** §3 の式は「サービングする製品」のモデルを選ぶ実務の中心にある。推論量の見積りが N の選択を変え、それが `../../software/ml-systems/ml-inference-systems.md` のサービング経済（KV cache、バッチング）と `../../hardware/gpu-accelerators/` の調達に波及する。
- **評価設計への警告。** §5 は「指標の不連続性が能力の不連続性に見える」罠を示す。ベンチマーク設計・能力予測・安全性評価のいずれでも、背後の連続量を測る計器選びが結論を左右する。
- **電力・設備への写像。** C ≈ 6ND は FLOPs であり、実効演算効率と電力効率を介してデータセンターの MW とキャンパス規模に翻訳される（`../../hardware/datacenter-infrastructure/datacenter-power.md`）。スケーリング則は建設計画の需要予測関数でもある。

## Daedalus内の応用 (Applications within Daedalus)

- `transformer-architecture.md` — 前提となる会計 C ≈ 6ND、損失＝符号長と既約項 E
- `../../software/ai-llm/llm-internals-prompting.md` — 挙動レベルの出口（思考の外部化＝テスト時計算、事前学習の意味）
- `../../software/ai-llm/agent-architectures.md` — pass^k・検証器と §6 のテスト時スケーリングの接続
- `../../software/ml-systems/ml-training-systems.md` — C を実時間・クラスタ規模に落とす側、`ml-inference-systems.md` — §3 の推論費用項の中身
- `../../hardware/gpu-accelerators/interconnects-scaleout-training.md`、`../../hardware/datacenter-infrastructure/` — スケーリング則が駆動する物理側

## 参考文献 (References)

- Kaplan, J. et al. (2020). Scaling Laws for Neural Language Models. arXiv:2001.08361
- Hestness, J. et al. (2017). Deep Learning Scaling is Predictable, Empirically. arXiv:1712.00409
- Hoffmann, J. et al. (2022). Training Compute-Optimal Large Language Models. *NeurIPS 2022*. arXiv:2203.15556
- Touvron, H. et al. (2023). LLaMA: Open and Efficient Foundation Language Models. arXiv:2302.13971
- Sardana, N. et al. (2024). Beyond Chinchilla-Optimal: Accounting for Inference in Language Model Scaling Laws. *ICML 2024*. arXiv:2401.00448
- Muennighoff, N. et al. (2023). Scaling Data-Constrained Language Models. *NeurIPS 2023*. arXiv:2305.16264
- Villalobos, P. et al. (2024). Will we run out of data? Limits of LLM scaling based on human-generated data. *ICML 2024*. arXiv:2211.04325
- Wei, J. et al. (2022). Emergent Abilities of Large Language Models. *TMLR*. arXiv:2206.07682
- Schaeffer, R. et al. (2023). Are Emergent Abilities of Large Language Models a Mirage? *NeurIPS 2023*. arXiv:2304.15004
- Snell, C. et al. (2024). Scaling LLM Test-Time Compute Optimally can be More Effective than Scaling Model Parameters. arXiv:2408.03314
