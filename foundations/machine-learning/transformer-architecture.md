# Transformer の数理 (The Mathematics of the Transformer)

## 概要 (Overview)

Transformer (Vaswani et al., 2017) は、現代の LLM のほぼ唯一のアーキテクチャである。その全体は、線形代数の言葉で驚くほど短く書ける——**トークン列をベクトル列に埋め込み、「注意」で列内の情報を混ぜ、「MLP」で各位置を変換する層を N 回重ね、最後に語彙上の確率分布を読み出す**。本ドキュメントはこの各段を定義から導出し、`../../software/ai-llm/llm-internals-prompting.md` が挙動レベルで述べた事実——固定計算、注意＝検索、文脈長の代償——に数式の裏付けを与える。

導出の副産物として、実務で使う会計式が全て手に入る: パラメータ数 ≈ 12Ld²、学習 FLOPs ≈ 6ND、KV cache サイズ、注意の O(N²) コスト。これらは `../../software/ml-systems/` の推論・学習システム分析が前提とする量であり、本ドキュメントはその導出元である。

## 前提知識 (Prerequisites)

- 行列積とベクトル空間、内積の幾何（`../mathematics/linear-algebra/vectors-matrices-linear-maps.md`）
- 低ランク・射影の直観（`../mathematics/linear-algebra/svd-low-rank.md`）
- 確率分布、期待値・分散、最尤推定（`../mathematics/probability-statistics/probability-random-variables.md`、`estimation-hypothesis-testing.md`）
- 勾配と連鎖律（`../mathematics/calculus-analysis/multivariable-calculus.md`）
- 交差エントロピーと符号長（`../information-theory/entropy-source-coding.md`）

記法: 語彙サイズ V、系列長 N、モデル次元（残差流の幅）d、層数 L、ヘッド数 h、ヘッド次元 d_k = d/h。行ベクトル規約を用い、X ∈ R^{N×d} は N トークン分の表現を並べた行列とする。

## 本論 (Main Content)

### 1. 埋め込みと残差流 — 全体の見取り図

トークン t_i（語彙中の整数）は埋め込み行列 W_E ∈ R^{V×d} の第 t_i 行として d 次元ベクトルになる。以後、全ての層はこの d 次元空間——**残差流 (residual stream)**——への読み書きとして動作する。

```text
x⁰_i = Embed(t_i) + Pos(i)
x^{l+1} = x^l + Attn^l(LN(x^l)) + MLP^l(LN(x^l))    (pre-LN 構成)
logits = LN(x^L) W_U          W_U ∈ R^{d×V}（読み出し行列）
P(next token) = softmax(logits)
```

要点は 2 つ。第一に、各層は残差流を**置き換える**のではなく**加算する** (residual connection)。恒等写像が既定であるため勾配が深さ方向に減衰しにくく、数十〜百層超の学習を可能にする（pre-LN がこの性質をさらに安定化させることは Xiong et al., 2020）。第二に、残差流は全部品が共有する**通信バス**であり、注意は「他の位置から情報を運び込む」部品、MLP は「その位置で情報を変換する」部品、と役割が直交する（この分解視点は Elhage et al., 2021 の数理的枠組みによる）。

### 2. 注意機構の導出 — スケール化内積注意

位置 i が文脈中の位置 j ≤ i から情報を集めたい。素朴には重み付き平均 Σ_j w_{ij} v_j を取ればよく、問題は重み w_{ij} をどう決めるかに帰着する。Transformer の答えは**内積による関連度**である。

```text
Q = XW_Q,  K = XW_K,  V = XW_V        W_Q, W_K ∈ R^{d×d_k}, W_V ∈ R^{d×d_v}

score_{ij} = (q_i · k_j) / √d_k                  … 関連度（スカラー）
w_{ij} = softmax_j(score_{ij} + mask_{ij})        … 行方向に正規化
out_i = Σ_j w_{ij} v_j                            … Value の凸結合
```

各要素の意味は次のとおりである。

- **Q/K/V の分業**: 同じ x_j から、「何を探しているか」(query)、「自分は何であるか」(key)、「見つかったら何を渡すか」(value) を**別々の線形写像**で作る。検索のインデックスと中身を分離した、微分可能な連想配列 (soft key-value store) である。
- **1/√d_k の由来**: q, k の成分が独立に平均 0・分散 1 なら、内積 q·k は d_k 項の和で分散 d_k を持つ。√d_k で割らないと次元が大きいほどスコアの絶対値が育ち、softmax が飽和して勾配が消える。スケーリングは分散を 1 に正規化する統計的な補正である。
- **因果マスク (causal mask)**: 自己回帰生成では位置 i は j > i を見てはならない。mask_{ij} = −∞ (j > i) を加えて softmax 後の重みを 0 にする。**学習時に全位置の予測を並列計算できる**のはこのマスクのおかげであり（各位置が「そこまでしか見ていない」状態を 1 回の行列積で同時に作れる）、Transformer が RNN を置き換えた実務上の最大の理由がこの並列性である。
- **softmax の役割**: 重みを正規化するだけでなく、温度付きの「argmax の緩和」として少数の位置への集中を可能にする。一様平均と 1 点選択の間を連続的に補間する。

計算量は score 行列が N×N であることから **O(N²·d)**。文脈長を 2 倍にすると注意の計算・メモリは 4 倍——長文脈の代償はこの 1 行から出る（工学への接続参照）。

### 3. マルチヘッド注意 — 部分空間での並列検索

1 つの注意は 1 種類の関連度しか表せない（構文的近さ・照応・同一トークンの反復は別々の「検索基準」である）。マルチヘッド注意は d 次元を h 個の部分空間に分け、各ヘッドが独立に QKV 検索を行い、結果を連結して W_O ∈ R^{d×d} で残差流に書き戻す。

```text
head_r = Attn(XW_Q^r, XW_K^r, XW_V^r)     r = 1..h,  各 d_k = d/h 次元
MultiHead(X) = concat(head_1, …, head_h) W_O
```

パラメータ総量は 1 ヘッドの full 次元注意と同じ（4d² per 層）で、表現の多様性だけが増える。個々のヘッドが解釈可能な役割（直前トークン参照、文脈中の反復パターンの延長＝induction head 等）に分化することは、機構的解釈可能性研究が実証している (Olsson et al., 2022)。

### 4. 位置情報 — 内積に相対位置を埋め込む

注意は集合演算であり、順序を知らない。位置情報は外から注入する必要がある。原論文の正弦波・学習埋め込みを経て、現代の主流は **RoPE (Rotary Position Embedding; Su et al., 2021)** である: q, k を 2 次元ずつのペアに分け、位置 m に応じた角度 mθ で回転させる。回転行列 R_m の性質から

```text
(R_m q) · (R_n k) = q · R_{n−m} k
```

となり、**内積が相対位置 n−m だけの関数になる**。絶対位置を教えずに「どれだけ離れているか」だけをスコアに入れる、内積構造と整合した設計である。周波数 θ を組ごとに変えることで近距離〜遠距離の複数スケールを同時に符号化する。文脈長の外挿・拡張（θ の再スケーリング）が RoPE の上の工学として発達した。

### 5. MLP ブロック — 各位置での変換と連想記憶

注意が位置間で情報を運んだ後、MLP が各位置**独立に**変換する。

```text
MLP(x) = σ(x W_in) W_out       W_in ∈ R^{d×4d}, W_out ∈ R^{4d×d}（4 倍拡張が慣例）
```

σ は GELU や SwiGLU 系のゲート付き非線形。この 2 段は「キーとの照合 → 値の書き出し」と読める: W_in の各列が検出パターン（キー）、W_out の対応行が発火時に残差流へ書き込む内容（値）であり、MLP 層は巨大な**キー・値の連想記憶 (key-value memory)** として機能する (Geva et al., 2021)。事実知識の多くが MLP 重みに局在するという実証は、この見方を支持する。パラメータ会計上は MLP が 8d² per 層と注意（4d²）の 2 倍を占め、**モデルの「知識容量」の主体は MLP 側**にある。

### 6. 学習目的 — 交差エントロピー＝符号長の最小化

学習は、各位置で次トークンの予測分布 p_θ(t_{i+1} | t_{≤i}) を出させ、交差エントロピー損失を最小化する。

```text
L(θ) = − (1/N) Σ_i log p_θ(t_{i+1} | t_{≤i})     [nats/token]
```

情報理論の言葉では、−log p は理想符号でのその トークンの符号長であり（`../information-theory/entropy-source-coding.md`）、**損失の最小化はデータの圧縮率の最大化と同値**である。perplexity = exp(L) は「毎ステップ実効何択に迷っているか」。この符号長解釈は次の 2 点で実用的である: (a) 損失の下限はデータ自身のエントロピー（べき則の「既約項」、`scaling-laws.md`）、(b) 「よく圧縮する＝規則性を捉えている」という学習の意味付けが、暗記と汎化を同じ物差しに載せる。

最適化は確率的勾配降下の系譜 (Adam 系) による。目的関数は非凸で、`../mathematics/optimization/convex-optimization-duality.md` の保証は直接には適用できないが、勾配法・学習率・条件数の言語はそのまま使われる。逆伝播は連鎖律の動的計画法であり、計算グラフを逆順に 1 回なめて全パラメータの勾配を得る（順伝播の約 2 倍の FLOPs——次節の係数 6 の由来）。

### 7. 会計 — パラメータ・FLOPs・KV cache

ここまでの構成から、実務の全ての見積り式が導出できる。

**パラメータ数**（埋め込みを除く本体）: 1 層あたり注意 4d²（W_Q, W_K, W_V, W_O）+ MLP 8d²（4 倍拡張の往復）= 12d²。よって

```text
N_params ≈ 12 L d²
```

**学習 FLOPs**: 行列積 [n×k]·[k×m] は 2nkm FLOPs（積和で 2）。パラメータ N のモデルの順伝播は 1 トークンあたり ≈ 2N（全パラメータが 1 回ずつ積和に参加）、逆伝播がその約 2 倍。合わせて 1 トークン ≈ 6N、D トークンの学習で

```text
C ≈ 6 N D    [FLOPs]
```

この 1 行が、スケーリング則 (`scaling-laws.md`) で計算予算 C を N と D に配分する議論の土台である。

**KV cache**: 生成時、過去トークンの K, V を再計算しない代わりに保持する。1 トークンあたり

```text
KV bytes/token = 2 × L × h_kv × d_k × (bytes/element)
```

（2 は K と V）。`../../software/ml-systems/ml-inference-systems.md` の容量式はこの導出の適用であり、GQA/MQA が h_kv < h とヘッドを共有するのは K, V 側の射影だけを細くしてこの項を圧縮する設計である (Ainslie et al., 2023)。

**推論の 2 相**: prefill は X が N 行の行列で行列×行列（計算律速）、decode は 1 行で行列×ベクトル（帯域律速）。`ml-inference-systems.md` が出発点とした二相構造は、同じ式の N = 系列長 と N = 1 の違いにすぎない。

## 工学への接続 (Engineering Connections)

- **O(N²) が長文脈工学を駆動する。** スコア行列の実体化を避けるタイリング (FlashAttention)、KV cache のページング (PagedAttention)、注意の疎化・線形化の研究は、すべて §2 の計算量への応答である。「文脈を増やせば解決」に物理的な価格表を与えるのがこの式である。
- **ほぼ全てが行列積である事実がハードウェアを決めた。** §7 の会計が示す通り、計算の大半は GEMM に落ちる。Tensor Core・TPU の脈動配列 (`../../hardware/gpu-accelerators/`) は「Transformer ＝行列積機械」への特化であり、逆に新アーキテクチャの実用性は「行列積に落ちるか」でほぼ決まる。
- **会計式は容量計画の道具である。** 12Ld² と 6ND と KV 式だけで、モデルサイズからメモリ・学習費用・サービング同時実行数の桁が見積れる。`ml-training-systems.md` の「16 バイト/パラメータ」等の会計はこの延長にある。
- **残差流という見方はデバッグと解釈の座標系を与える。** 「どの層が何を書き込んだか」を残差流への加算として追う機構的解釈可能性は、幻覚・ジェイルブレイクの機構的理解や、活性化操作 (activation steering) の基礎になっている。

## Daedalus内の応用 (Applications within Daedalus)

- `../../software/ai-llm/llm-internals-prompting.md` — 本ドキュメントの挙動レベルの出口。固定計算（§1–5 の固定深さ）、注意＝検索（§2–3）、条件付き分布（§6）の実務的帰結
- `../../software/ml-systems/ml-inference-systems.md` — §7 の KV cache 式・二相構造の消費者。FlashAttention・PagedAttention は §2 の O(N²) への応答
- `../../software/ml-systems/ml-training-systems.md` — 6ND と 12Ld² を前提とする分散学習の会計
- `../../hardware/gpu-accelerators/` — 「ほぼ行列積」という計算特性が規定するハードウェア側
- `../mathematics/linear-algebra/` — 射影・低ランク・テンソルの言語、`../information-theory/entropy-source-coding.md` — 損失＝符号長の同値、`../mathematics/optimization/` — 勾配法の言語

## 参考文献 (References)

- Vaswani, A. et al. (2017). Attention Is All You Need. *NeurIPS 2017*. arXiv:1706.03762
- Bahdanau, D. et al. (2015). Neural Machine Translation by Jointly Learning to Align and Translate. *ICLR 2015*. arXiv:1409.0473
- Xiong, R. et al. (2020). On Layer Normalization in the Transformer Architecture. *ICML 2020*. arXiv:2002.04745
- Su, J. et al. (2024). RoFormer: Enhanced Transformer with Rotary Position Embedding. *Neurocomputing*, 568. arXiv:2104.09864
- Geva, M. et al. (2021). Transformer Feed-Forward Layers Are Key-Value Memories. *EMNLP 2021*. arXiv:2012.14913
- Elhage, N. et al. (2021). A Mathematical Framework for Transformer Circuits. *Transformer Circuits Thread*. https://transformer-circuits.pub/2021/framework/
- Olsson, C. et al. (2022). In-context Learning and Induction Heads. *Transformer Circuits Thread*. arXiv:2209.11895
- Ainslie, J. et al. (2023). GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints. *EMNLP 2023*. arXiv:2305.13245
