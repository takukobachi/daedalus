# エントロピーと情報源符号化 (Entropy and Source Coding)

## 概要 (Overview)

情報理論シリーズの第1回。「情報の量」を確率から定義し、そこから**可逆圧縮の理論的下限**を導く。出発点は1つの観察である——めったに起きない事象を知らされたときほど「情報を得た」と感じる。これを裏返せば、情報量とは**驚き (surprisal) の定量化**であり、確率だけの関数として定義できる。Shannon (1948) はこの直観を公理化し、期待驚きとしての**エントロピー (entropy)** H を定義して、「記号列を平均 H ビット未満には縮められず、H ビットには漸近できる」（情報源符号化定理）を証明した。

この1つの量が驚くほど多くの顔を持つ。gzip や zstd の圧縮率の下限であり、機械学習の損失関数（クロスエントロピー）であり、言語モデルの評価指標（パープレキシティ）であり、暗号鍵の強度（エントロピー源の質）である。本文書の到達点は「**圧縮と予測は同じ問題である**」——次の記号をよく予測できるモデルは、そのままよい圧縮器になる——という等価性の理解である。

## 前提知識 (Prerequisites)

- [`../mathematics/probability-statistics/probability-random-variables.md`](../mathematics/probability-statistics/probability-random-variables.md) — 確率変数、期待値、独立性、条件付き確率
- [`../mathematics/calculus-analysis/limits-derivatives-integrals.md`](../mathematics/calculus-analysis/limits-derivatives-integrals.md) — 対数の性質、凸関数（Jensen の不等式を §3 で使う）

## 本論 (Main Content)

### 1. 自己情報量 — なぜ対数か (self-information)

事象 x の**自己情報量 (self-information)** を I(x) = −log₂ p(x) と定義する（単位: ビット）。確率 1/2 の事象で 1 ビット、確率 1/1024 の事象で 10 ビット。対数である必然性は2つの要請から来る。

- **単調性**: 確率が低いほど情報量は大きい（確実な事象は情報量 0）
- **加法性**: 独立な事象の同時観測の情報量は和になる — p(x,y) = p(x)p(y) のとき I(x,y) = I(x) + I(y)

「積を和に変える連続関数」は対数しかない。底の選択は単位の選択（底 2 でビット、e でナット）にすぎない。コイン投げ 10 回の結果は 10 ビット、6 面ダイス 1 回は log₂6 ≈ 2.58 ビット——**情報量は「その結果を特定するのに必要な二択の回数」** という操作的な読みを持つ。

### 2. エントロピー — 期待驚きとその形 (entropy)

確率変数 X の**エントロピー**は自己情報量の期待値である。

```text
H(X) = −Σₓ p(x) log₂ p(x)   [ビット/記号]
```

基本性質: (i) H(X) ≥ 0、(ii) 決定的な変数で 0、(iii) n 値の変数では一様分布のとき最大値 log₂n。偏りは情報量を減らす——表が出る確率 p のコインの**二値エントロピー関数** H(p) = −p log₂p − (1−p) log₂(1−p) は p = 0.5 で最大 1 ビット、p = 0.9 では約 0.47 ビットしかない。「表が 9 割のコイン投げの列」が直観的にも「中身が薄い」のは、この意味で厳密に正しい。

英語テキストの文字あたりエントロピーは、26 文字一様なら log₂26 ≈ 4.7 ビットだが、頻度の偏りで約 4.1 ビット、文脈（直前の文字列からの予測）まで使うと 1 ビット前後まで落ちる (Shannon 1951)。**冗長性 (redundancy) = 最大エントロピーとの差**が、圧縮で回収できる余地の正体である。

### 3. 相互情報量と KL ダイバージェンス (mutual information and KL divergence)

2 変数に拡張すると、依存関係が情報量の言葉で測れるようになる。

```text
結合エントロピー    H(X,Y) = H(X) + H(Y|X)          [chain rule]
条件付きエントロピー H(Y|X) = 「X を知った後に残る Y の不確かさ」
相互情報量          I(X;Y) = H(Y) − H(Y|X) = H(X) − H(X|Y)
                          = 「X を知ることで減る Y の不確かさ」(対称!)
```

I(X;Y) ≥ 0 であり、0 になるのは独立のときに限る。「相関係数と違って**任意の**（非線形の）依存を捉える依存尺度」であることが、特徴選択や通信路容量（第2回）での役割を支える。

**KL ダイバージェンス (Kullback–Leibler divergence)** D(p‖q) = Σ p(x) log₂(p(x)/q(x)) は「真の分布 p を、モデル q で近似したときの非効率」を測る。基本不等式 D(p‖q) ≥ 0（等号は p = q、証明は log の凹性への Jensen の不等式一発）から、エントロピーの諸性質と §4 の符号化下限がすべて流れ出る。I(X;Y) = D(p(x,y)‖p(x)p(y)) ——相互情報量は「結合分布が独立からどれだけ離れているか」の KL である。

### 4. 情報源符号化定理 — 圧縮の下限 (source coding theorem)

記号を可変長のビット列（符号語）に割り当てて圧縮することを考える。語尾を見ずに復号できる（どの符号語も他の接頭辞にならない）**接頭符号 (prefix code)** の符号長 {ℓₓ} は **Kraft の不等式** Σₓ 2^(−ℓₓ) ≤ 1 を満たし、逆にこれを満たす任意の長さの組は接頭符号として実現できる (Kraft 1949; McMillan 1956)。符号設計とは、この制約下で期待符号長 L = Σ p(x)ℓₓ を最小化する最適化問題であり、その答えが次の情報源符号化定理である。

```text
H(X) ≤ L_min < H(X) + 1        [情報源符号化定理]
```

下限側は Kraft 制約と D(p‖q) ≥ 0 から（q(x) = 2^(−ℓₓ)/Σ2^(−ℓ) と置く）、上限側は ℓₓ = ⌈−log₂ p(x)⌉ という素朴な割当て（Shannon 符号）で達成される。**最適符号長は −log₂ p(x)、つまり自己情報量そのもの**——「頻出記号に短い符号」というモールス信号以来の直観が、厳密な最適性として正当化される。記号単位で最適な割当てを構成するのが Huffman 符号 (Huffman 1952)、ブロック化して +1 のオーバーヘッドを 1 記号あたり +1/n に薄めていくと L → H に漸近する。実装側の各論（Huffman・算術符号・ANS・LZ 系）は [`../../software/compression/lossless-compression.md`](../../software/compression/lossless-compression.md) が扱う。

大数の法則の情報理論版として、長さ n の記号列はほぼ確率 2^(−nH) の**典型系列 (typical sequences)** 約 2^(nH) 本に集中する（漸近等分割性, AEP）。「2^(nH) 本しかないものを番号で指せば nH ビット」——これが定理のもう1つの証明であり、「エントロピー＝実効的な場合の数の対数」という読みを与える。

### 5. クロスエントロピー — 圧縮と予測の等価 (cross-entropy: compression = prediction)

真の分布 p の記号列を、モデル分布 q に最適な符号（長さ −log₂ q(x)）で圧縮すると、期待符号長は**クロスエントロピー (cross-entropy)** になる。

```text
H(p, q) = −Σ p(x) log₂ q(x) = H(p) + D(p‖q)
```

つまり**モデルの不正確さ D(p‖q) が、そのまま圧縮の超過ビットとして支払われる**。機械学習でクロスエントロピー損失を最小化することは、H(p) が定数なので D(p‖q) の最小化であり、「訓練データを最も短く圧縮できるモデルを探すこと」と同値である。言語モデルの**パープレキシティ** = 2^(クロスエントロピー) は「モデルが次の記号を実効何択まで絞れているか」を意味する。この等価は比喩ではなく機構である——確率予測を出すモデルは算術符号（[`../../software/compression/lossless-compression.md`](../../software/compression/lossless-compression.md) §技術詳細 3）を介してそのまま圧縮器に変換でき、逆もまた成り立つ——この往復の実装が本シリーズと圧縮分析文書を繋ぐ。

## 工学への接続 (Engineering Connections)

- **圧縮率の相場観** (§2, §4): 「このデータはあとどれだけ縮むか」への原理的な答えがエントロピー推定である。圧縮器の出力がさらに縮まない（高エントロピー化する）こと、暗号文・乱数が圧縮不能であることは、いずれも H の飽和として読める。
- **エントロピーという名の資源** (§2): 暗号鍵の強度は「攻撃者にとっての実効エントロピー」で測る。パスワードの文字種×長さの見積り、TRNG の健全性検査、`/dev/urandom` のエントロピープール——「乱数の質」の定量化はすべて §2 の量である。
- **予測が利くところ圧縮あり** (§5): 分岐予測・プリフェッチ・キャッシュはいずれも「プログラムの挙動のエントロピーが低い」ことを搾取する機構であり、予測精度の限界はその挙動のエントロピーで抑えられる。
- **ログ・メトリクスの情報密度** (§2, §4): 時系列メトリクスが 10 倍以上縮む（Gorilla 型差分符号化）のは、隣接サンプルの条件付きエントロピー H(xₜ|xₜ₋₁) が周辺エントロピーよりはるかに小さいからである。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`software/ai/ml-training-systems.md`](../../software/ai/ml-training-systems.md) | クロスエントロピー損失の最小化 = KL 最小化 = 訓練データの最短圧縮 (§5) |
| [`hardware/wireless/radio-fundamentals-modulation.md`](../../hardware/wireless/radio-fundamentals-modulation.md) | 変調シンボルあたりのビット数・符号化率の分母にある「情報量」の定義 (§1–2) |
| [`hardware/cpu/branch-prediction-speculation.md`](../../hardware/cpu/branch-prediction-speculation.md) | 分岐履歴の条件付きエントロピーが予測精度の理論限界を与える (§3) |
| [`hardware/security/side-channels-tee-secure-boot.md`](../../hardware/security/side-channels-tee-secure-boot.md) | 鍵・乱数の実効エントロピー、サイドチャネル漏洩の「漏れた情報量」的読み (§2–3) |
| [`software/observability/observability-performance-engineering.md`](../../software/observability/observability-performance-engineering.md) | メトリクス圧縮 (Gorilla 型) が搾取する時系列の低い条件付きエントロピー (§3–4) |
| [`foundations/mathematics/probability-statistics/probability-random-variables.md`](../mathematics/probability-statistics/probability-random-variables.md) | エントロピー・KL の土台となる確率変数と期待値 (§1–3) |

## 参考文献 (References)

- Shannon, C. E. (1948). A Mathematical Theory of Communication. *Bell System Technical Journal*, 27(3), 379–423. DOI: 10.1002/j.1538-7305.1948.tb01338.x — 分野の原典。エントロピー・情報源符号化定理・通信路容量を一挙に導入
- Shannon, C. E. (1951). Prediction and Entropy of Printed English. *Bell System Technical Journal*, 30(1), 50–64. DOI: 10.1002/j.1538-7305.1951.tb01366.x — 英語のエントロピー実測（§2 の数値の出所）
- Cover, T. M., & Thomas, J. A. (2006). *Elements of Information Theory* (2nd ed.). Wiley. DOI: 10.1002/047174882X — 標準教科書。AEP・符号化定理の厳密な証明の主参照
- MacKay, D. J. C. (2003). *Information Theory, Inference, and Learning Algorithms*. Cambridge University Press. — 情報理論と機械学習の等価性 (§5) を最も明快に扱う教科書
- Huffman, D. A. (1952). A Method for the Construction of Minimum-Redundancy Codes. *Proceedings of the IRE*, 40(9), 1098–1101. DOI: 10.1109/JRPROC.1952.273898 — 記号単位最適符号の原典
- McMillan, B. (1956). Two Inequalities Implied by Unique Decipherability. *IRE Transactions on Information Theory*, 2(4), 115–116. DOI: 10.1109/TIT.1956.1056818 — Kraft 不等式の一意復号可能符号への拡張
