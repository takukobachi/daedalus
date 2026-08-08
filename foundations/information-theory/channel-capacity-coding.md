# 通信路容量と誤り訂正の限界 (Channel Capacity and the Limits of Error Correction)

## 概要 (Overview)

情報理論シリーズの第2回・最終回。前回はノイズのない世界での圧縮の下限を導いた。本回はノイズのある通信路 (channel) を渡る話——そして Shannon (1948) の最も反直観的な定理を扱う。**ノイズがあっても、誤り率を犠牲にせずに通信できる**。伝送レートが通信路固有の定数——**通信路容量 (channel capacity)** C——未満でありさえすれば、符号を工夫することで誤り率はいくらでも 0 に近づけられ、C を超えれば どんな符号でも必ず破綻する。「ノイズが多いなら声を大きく・ゆっくり話すしかない」という素朴な直観に対し、この定理は「賢く話せば、ある速度までは完璧に伝わる」と答える。

工学的に重要なのは、この限界が**先に証明され、実装が半世紀かけて追いついた**ことである。定理は最適符号の存在を示すだけで構成しない。Hamming に始まる符号設計の歴史は「シャノン限界まであと何 dB」を詰める競走であり、Turbo (1993)・LDPC (1962 発明、1990 年代再発見)・Polar (2009) がその到達点である。無線リンクの MCS 選択も NAND の ECC も、この「限界との距離」を通貨として設計されている。

## 前提知識 (Prerequisites)

- [`entropy-source-coding.md`](entropy-source-coding.md) — エントロピー、条件付きエントロピー、相互情報量
- [`../mathematics/number-theory-algebra/finite-fields-coding.md`](../mathematics/number-theory-algebra/finite-fields-coding.md) — 符号の**構成側**（Reed–Solomon、LDPC の代数）。本回はその**限界側**
- [`../mathematics/fourier-signal-processing/sampling-dft-fft.md`](../mathematics/fourier-signal-processing/sampling-dft-fft.md) — §3 の帯域幅とサンプリングの関係

## 本論 (Main Content)

### 1. 通信路のモデル (channel models)

通信路を条件付き分布 p(y|x)——「x を入れたとき y が出る確率」——としてモデル化する。2 つの教科書的通信路がほとんどの直観を与える:

```text
BSC(p)  二元対称通信路: 各ビットが確率 p で反転        容量 C = 1 − H(p)
BEC(ε)  二元消失通信路: 各ビットが確率 ε で「消失」     容量 C = 1 − ε
        (どこが失われたかは分かる)
```

BSC の容量が 1 − H(p) であること——反転確率 p = 0.11 で容量は半分、p = 0.5 で 0（コイン投げと区別がつかない）——は、**受信者の不確かさ H(p) がそのまま帯域の税として引かれる**ことを意味する。消失位置が分かる BEC の方が同じ損失率でも容量が大きい。パケットロス（どのパケットが落ちたか分かる）が消失モデル、ビット化け（どこが誤ったか分からない）が対称誤りモデルに対応し、イレイジャーコーディングと ECC という対処の違い（[`../mathematics/number-theory-algebra/finite-fields-coding.md`](../mathematics/number-theory-algebra/finite-fields-coding.md) §5 の RS 符号の両面）はこのモデル差の反映である。

### 2. 通信路符号化定理 (noisy-channel coding theorem)

**容量**を C = max_{p(x)} I(X;Y) と定義する——入力分布を最良に選んだときの、出力が入力について運ぶ相互情報量である。定理は2つの半分から成る:

- **達成可能性**: R < C なる任意のレートに対し、ブロック長 n を大きくすれば誤り率を任意に小さくできる符号が存在する
- **逆定理**: R > C では、いかなる符号でも誤り率は 0 から離れたままである

達成側の証明が有名な**ランダム符号化 (random coding)** 論法である——2^(nR) 個の符号語を*ランダムに*選んでも、平均的には成功する。ゆえに良い符号は存在する。だが「ランダムに選んだ符号」には構造がなく、復号は総当たりになる。**存在証明と構成のこの断絶**が、その後 50 年の符号理論——構造（復号可能性）とランダム性（性能）の両立の探求——を規定した。

直観は典型集合（前回 §4）で立つ。長さ n の受信語に対し、送信されえた典型的な原因はおよそ 2^(nH(X|Y)) 通りに絞られる。符号語を 2^(nI(X;Y)) 個以下しか使わなければ、「原因の玉」同士が重ならず一意に復元できる——**相互情報量は「1 記号あたり何ビットぶん原因を特定できるか」**である。

### 3. AWGN と Shannon–Hartley — アナログ世界の容量 (the AWGN channel)

物理層の実像は連続値である。帯域幅 B [Hz]、信号電力 S、雑音電力 N の加法的白色ガウス雑音 (AWGN) 通信路の容量が **Shannon–Hartley の公式** (Shannon 1949):

```text
C = B log₂(1 + S/N)   [bit/s]
```

読み方が3つある。(i) **帯域は線形、電力は対数**——帯域を倍にすれば容量は倍だが、電力を倍にしても log₂ の中身が動くだけ。周波数帯が「不動産」と呼ばれる理由である。(ii) SNR が高いほど 1 シンボルに多ビット載る——Wi-Fi や 5G が SNR に応じて QPSK から 1024-QAM まで変調を切り替える（適応変調符号化, AMC）のは、この式を運転していることに等しい。(iii) B → ∞ の極限でも容量は S/N₀ln2 で頭打ちになり、そこから**ビットあたり最小エネルギー Eb/N₀ = ln2 = −1.59 dB** という究極の限界が出る。深宇宙通信のリンク設計はこの限界の数 dB 以内で行われる。

### 4. 限界へ迫る実装の系譜 (closing the gap)

「シャノン限界まであと何 dB」を物差しに、符号の歴史は一本の収束列として読める:

```text
1950  Hamming 符号        単一誤り訂正。限界まで ~10 dB
1955– 畳み込み符号+Viterbi 逐次復号。衛星・深宇宙の主力に    ~4–6 dB
1993  Turbo 符号           反復復号の衝撃 (Berrou et al.)     ~0.5 dB
1996– LDPC 再発見          Gallager (1962) の再評価           ~0.1 dB級
2009  Polar 符号           初の「証明つき」容量達成構成 (Arıkan)
```

Turbo/LDPC の核心は**反復的な確率伝播 (belief propagation)** ——復号器が硬判定 (0/1) ではなく確率（軟判定, soft decision）をやり取りし、推定を往復で磨く——にある。§2 のランダム符号化が要求した「構造のなさ」を、疎なランダムグラフ（LDPC の検査行列）として復号可能な形で実現したものと読める。採用面では、LDPC が Wi-Fi (802.11n 以降)・5G データチャネル・10G Ethernet・NAND の主力 ECC を占め、Polar が 5G 制御チャネルに入った。NAND コントローラが読み出しリトライで参照電圧を振って軟判定情報を集めるのは、§2 の「硬判定は情報を捨てている」（データ処理不等式）の実装的裏返しである。

### 5. レート歪みと分離定理 — 残る2枚の地図 (rate-distortion and separation)

**レート歪み理論 (rate-distortion theory)** は非可逆圧縮側の限界地図である (Shannon 1959)。許容歪み D のもとで必要な最小レート R(D) = min I(X;X̂) ——「どこまで壊してよいか」を決めると「どこまで縮むか」が決まる。JPEG や動画符号化のレート制御（[`../../software/compression/media-codecs.md`](../../software/compression/media-codecs.md) §技術詳細 5）は、この曲線上の動作点選択として読める。

**情報源・通信路分離定理 (source-channel separation)** は「圧縮してから誤り訂正符号を掛ける」2 段構成が、漸近的には最適であることを保証する。プロトコルスタックが圧縮 (アプリ層) と FEC (物理層) を独立に設計できる理論的免罪符である。ただし成立条件は「ブロック長 → ∞、遅延無制限」であり、低遅延・有限ブロック長の領域（音声通話、URLLC）では分離が最適でなくなる——リアルタイム系で圧縮と誤り耐性が絡み合った設計（不均等誤り保護など）が現れる理由もまた、この定理の**破れ方**から説明される。

## 工学への接続 (Engineering Connections)

- **リンクバジェットの通貨** (§3): 無線・SerDes・衛星リンクの設計は「距離・電力・帯域から SNR を出し、容量と符号の実力差を引いて実効レートを見積もる」計算であり、全項が本回の量で書かれる。
- **適応変調符号化 (AMC)** (§3–4): SNR 実測 → MCS (変調 × 符号化率) 選択のループは、「その瞬間の容量にレートを追従させる」制御である。Wi-Fi のレート制御も Starlink のビームごとのリンク適応も同型。
- **ストレージは時間方向の通信路** (§1, §4): NAND の経年劣化・リテンションは「書いた時点の自分から読む時点の自分への通信」のノイズであり、ECC の強化 (BCH → LDPC) は媒体の通信路容量低下への追随である。
- **FEC と遅延のトレードオフ** (§5): ブロック長を伸ばすほど限界に近づくが遅延が増える。有限ブロック長の性能限界が、URLLC・音声・ゲームストリーミングの符号選択を規定する。
- **再送か前方訂正か** (§1): フィードバックは容量を増やさない（無記憶通信路）が、実装を劇的に単純化する——ARQ/ハイブリッド ARQ と純 FEC の使い分けは、容量ではなく遅延・複雑性の経済で決まる。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/wireless-rf/radio-fundamentals-modulation.md`](../../hardware/wireless-rf/radio-fundamentals-modulation.md) | Shannon–Hartley、Eb/N₀、AMC、LDPC/Polar 採用——本回の限界側の実装カタログ (§3–4) |
| [`hardware/peripherals-connectivity/wifi-architecture.md`](../../hardware/peripherals-connectivity/wifi-architecture.md) | MCS テーブルと適応レート制御 = 容量への追従制御 (§3) |
| [`hardware/aerospace/satellites/starlink-satellite-bus.md`](../../hardware/aerospace/satellites/starlink-satellite-bus.md) | 衛星リンクバジェットと位相配列のビームごとリンク適応 (§3) |
| [`hardware/storage-devices/nand-flash-ftl.md`](../../hardware/storage-devices/nand-flash-ftl.md) | LDPC 軟判定復号・読み出しリトライ = 時間方向通信路の容量追随 (§4) |
| [`hardware/serdes-signal-integrity/serdes-link-architecture.md`](../../hardware/serdes-signal-integrity/serdes-link-architecture.md) | リンク FEC (RS-FEC) の採否と遅延トレードオフ (§4–5) |
| [`foundations/mathematics/number-theory-algebra/finite-fields-coding.md`](../mathematics/number-theory-algebra/finite-fields-coding.md) | 符号の構成論（本回はその存在限界。両者で符号理論の全景になる） (§2, §4) |

## 参考文献 (References)

- Shannon, C. E. (1948). A Mathematical Theory of Communication. *Bell System Technical Journal*, 27(3), 379–423. DOI: 10.1002/j.1538-7305.1948.tb01338.x — 通信路符号化定理の原典
- Shannon, C. E. (1949). Communication in the Presence of Noise. *Proceedings of the IRE*, 37(1), 10–21. DOI: 10.1109/JRPROC.1949.232969 — AWGN 容量 (Shannon–Hartley) の原典
- Shannon, C. E. (1959). Coding Theorems for a Discrete Source with a Fidelity Criterion. *IRE National Convention Record*, 7(4), 142–163. — レート歪み理論の原典
- Cover, T. M., & Thomas, J. A. (2006). *Elements of Information Theory* (2nd ed.). Wiley. DOI: 10.1002/047174882X — 符号化定理・分離定理の厳密な証明の主参照
- Gallager, R. G. (1962). Low-Density Parity-Check Codes. *IRE Transactions on Information Theory*, 8(1), 21–28. DOI: 10.1109/TIT.1962.1057683 — LDPC の原典（30 年早すぎた発明）
- Berrou, C., Glavieux, A., & Thitimajshima, P. (1993). Near Shannon Limit Error-Correcting Coding and Decoding: Turbo-Codes. *Proceedings of IEEE ICC '93*, 1064–1070. DOI: 10.1109/ICC.1993.397441 — Turbo 符号の原典
- Arıkan, E. (2009). Channel Polarization: A Method for Constructing Capacity-Achieving Codes for Symmetric Binary-Input Memoryless Channels. *IEEE Transactions on Information Theory*, 55(7), 3051–3073. DOI: 10.1109/TIT.2009.2021379 — Polar 符号の原典
