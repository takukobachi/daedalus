# 無線通信の基礎 — 電波伝搬・変調・OFDM・MIMO (Radio Fundamentals & Modulation)

## 概要 (Overview)

`../peripherals/wifi-architecture.md` は Wi-Fi を MAC 層——共有媒体の交通整理——から解剖し、`../peripherals/bluetooth-pairing-protocol.md` は低消費電力側の設計を扱った。本ドキュメントはその**一段下**、すべての無線標準が共有する物理層 (PHY) の理論——電波がどう減衰し、ノイズが何で決まり、1 Hz の帯域に何ビット載せられ、マルチパスとどう戦うか——を扱う。Wi-Fi の 1024-QAM も 5G NR の OFDM も GNSS のスペクトラム拡散も、**同じ物理・同じ情報理論の上の異なる構成**であり、この土台を押さえれば各標準の PHY 仕様は「選択の記録」として読める。

概念上の兄弟は有線側の [`../signal-integrity/serdes-link-architecture.md`](../signal-integrity/serdes-link-architecture.md) である。銅線の SerDes が表皮効果・誘電損と戦うように、無線 PHY は自由空間損失・フェージングと戦う——敵の名前は違うが、ISI (符号間干渉)・ノイズ・SNR という会計は同一である。電磁波そのものの物理（Maxwell 方程式・アンテナ放射）は [`../../foundations/physics/electromagnetism/maxwell-equations-em-waves.md`](../../foundations/physics/electromagnetism/maxwell-equations-em-waves.md) が供給し、本ドキュメントはその上の**通信工学の層**に徹する。分析深度は Level 3。

## 方法論 (Methodology)

- **伝搬（チャネルの物理）→ 情報理論（シャノンの天井）→ 変調・符号化（天井への漸近手段）→ OFDM / MIMO（マルチパスへの応答）→ RF チェーン（回路実装）** の順に、制約が設計を決める向きで積み上げる。
- 全体を貫く道具として **dB の算術**を採用する——リンクバジェットは掛け算を足し算に変えた帳簿であり、各機構の貢献（アンテナ利得・符号化利得・ダイバーシチ利得）はすべて dB でこの帳簿に記帳できる。
- 数値は規格の公称値（3GPP TS 38.101 の UE 送信電力、IEEE 802.11 の変調次数など）と教科書的な典型値（受信機雑音指数など）を区別して示す。仕様上の最大値と実効値の乖離は `wifi-architecture.md` の注意がそのまま適用される。
- 標準の網羅ではなく部品の理解を目的とし、実標準（Wi-Fi・LTE/5G・GNSS・Bluetooth）は各部品の**構成例**として対応表で扱う。

## 発見事項 (Findings)

1. **リンクバジェットは dB の帳簿であり、無線リンク設計とは残高を黒字にする会計である。** 送信電力から出発し、アンテナ利得を足し、経路損失とマージンを引き、受信感度と比べる——この一列の足し引きがリンクの成立を決める。自由空間経路損失 (FSPL: free-space path loss) は 2.4 GHz・100 m で約 80 dB、すなわち電力にして 1 億分の 1 であり、無線が「桁の世界」であることを強制する。dB 表記はこの桁の世界を線形の帳簿に変換する装置である。
2. **受信機の下限は物理定数が決める——熱雑音 −174 dBm/Hz はどの標準も動かせない床である。** 雑音電力 kTB（室温で −174 dBm/Hz × 帯域幅）に受信機自身の**雑音指数 (NF: noise figure)**（典型 3–8 dB）が上乗せされ、要求 SNR を加えたものが受信感度になる。帯域を 10 倍にすれば床は 10 dB 上がる——「広帯域＝高速」には常に感度という代金が付く。GNSS はこの床の**下**（地表受信電力約 −128.5 dBm、雑音床より約 20 dB 下）で動く例外であり、拡散利得約 43 dB で床の下から信号を掘り出す。
3. **現代の無線標準はすべてシャノン限界への漸近戦略として読める。** C = B log₂(1+SNR) は、帯域幅 B と SNR から導かれる誤りなし伝送の上限である。log の中に SNR がある以上、SNR を倍にしても容量は 1 bit/s/Hz しか増えない——ゆえに歴史は (1) 変調の高次化（SNR の log 領域での搾取）、(2) 帯域の拡大（線形に効く B の獲得——6 GHz 帯・mmWave）、(3) **MIMO による空間次元の追加**（実効的に C を並列化する唯一の手段）の順に進んだ。FEC (前方誤り訂正) の進化（畳み込み符号 → ターボ → LDPC/Polar）は、要求 SNR を限界へ 1 dB 単位で近づける作業だった。
4. **変調の選択は「1 シンボルのビット数」と「要求 SNR」の交換算術である。** OOK/FSK（振幅・周波数の 2 値）から QPSK・QAM へと、コンスタレーション (constellation) 上の点を増やすほど 1 シンボルに載るビットは増えるが、点間距離が縮み要求 SNR が上がる——QAM では 2 bit 追加（4 倍の点数）ごとに約 6 dB。QPSK が数 dB で動く一方、1024-QAM は 35 dB 級の SNR を要求し、近距離でしか使えない。送信品質はコンスタレーション上の点のずれ **EVM (error vector magnitude)** で測られ、高次 QAM ほど EVM 要求（すなわち RF 回路の線形性要求）が厳しくなる——変調次数は情報理論と RF 回路設計を直結する蝶番である。
5. **OFDM は等化問題を FFT で対角化する。** マルチパス遅延広がりがシンボル長に対して無視できなくなると ISI が生じる。1 本の速いキャリアを等化するのが有線 SerDes の道（DFE・FFE の積層）だとすれば、OFDM は**多数の遅いサブキャリア**に分割し、各サブキャリアではチャネルが単なる複素係数 1 個（平坦フェージング）になるようにする。サイクリックプレフィックス (cyclic prefix) が遅延広がりを吸収して線形畳み込みを巡回畳み込みに変え、FFT がチャネル行列を対角化する——時間領域の等化器の積層が、周波数領域の**係数 1 個の除算 × サブキャリア数**に化ける。OFDMA はこの分割をそのまま多元接続（ユーザーへのサブキャリア割当）に転用したものである。
6. **MIMO はフェージングを「敵」から「資源」に変えた。** マルチパスによるレイリーフェージングは深い落ち込み（数十 dB）を作る敵だが、複数アンテナ間でフェージングが独立なら、(a) 同じデータを複数経路で送って落ち込みを回避する**ダイバーシチ (diversity)**、(b) 経路の独立性を利用して**複数ストリームを同一周波数で同時に送る空間多重 (spatial multiplexing)** が選べる。後者は帯域も電力も増やさずに容量を乗算する、シャノンの式に対する唯一の「抜け道」である。ビームフォーミングは同じアンテナ群を位相制御して電力を空間的に集中させる第三の使い方で、mmWave では高い経路損失を補う必需品になる。
7. **有線 SerDes と無線 PHY は、同じ敵 (ISI・ノイズ・SNR) と戦う別実装である。** チャネルが静的で製造時に測れる有線はトレーニングで等化器係数を固定し、チャネルが移動とともに毎ミリ秒変わる無線はパイロット信号による連続推定と適応変調 (AMC) で追従する。FEC が損失バジェットの通貨であること、受信機が ADC + DSP 化していくこと、性能の価格をプロセス微細化が決めること——`../signal-integrity/` で見た構造がすべて再現される。
8. **アナログとデジタルの境界は ADC の位置で決まり、境界はアンテナへ向かって動き続けている。** スーパーヘテロダインは周波数変換をアナログ多段で行い、ダイレクトコンバージョンは 1 段でベースバンドへ落とし、ソフトウェア無線 (SDR: software-defined radio) は ADC をできる限りアンテナに近づけて残りを DSP にする。SerDes 受信機の ADC-DSP 化と同じ力学——**アナログの問題を計算で買い戻す**——が無線でも進行しており、その終点では「無線機の設計」は信号処理ソフトウェアの設計になる。

## アーキテクチャ (Architecture)

### 受信 RF チェーン — アナログが終わり DSP が始まる場所

```
 アンテナ                アナログ領域                          │ デジタル領域 (DSP)
   ▼                                                          │
 ┌────┐   ┌─────┐   ┌────────┐   ┌────────┐   ┌─────┐   ┌────┐│┌──────────────────────┐
 │ BPF├──▶│ LNA ├──▶│ ミキサ  ├──▶│ IF/BB  ├──▶│ AGC ├──▶│ADC │▶│ デジタルフィルタ      │
 └────┘   └─────┘   └───┬────┘   │ フィルタ│   └─────┘   └────┘│├──────────────────────┤
  帯域選択  低雑音増幅    │        └────────┘   利得制御   量子化 ││ 同期・チャネル推定    │
           (NF を支配)   │ 局部発振 (LO)                        ││ FFT (OFDM 復調)       │
                     ┌───┴────┐                                ││ 等化・MIMO 検出       │
                     │PLL/VCO │◀── 基準クロック (XO/TCXO)       ││ 復調 (QAM デマップ)   │
                     └────────┘                                ││ FEC 復号 (LDPC 等)    │
                                                               │└──────────┬───────────┘
  送信側は逆順: DAC → ミキサ (アップコンバート) → PA → BPF → アンテナ        ▼  MAC 層へ
```

- **LNA (low-noise amplifier)** が連鎖全体の雑音指数をほぼ決める（Friis の縦続雑音式: 初段の NF が支配し、後段の寄与は初段利得で割られる）。**PA (power amplifier)** は送信側の対称物で、電力効率と線形性（高次 QAM の EVM 要求）の交換に苦しむ部品である。**PLL/VCO** は搬送波を生成し、その位相雑音は受信 EVM の床になる——発振器の物理は [`../timekeeping/clocks-time-synchronization.md`](../timekeeping/clocks-time-synchronization.md) と共通である。
- **スーパーヘテロダイン (superheterodyne)** は一旦中間周波数 (IF) へ変換して利得とフィルタリングを稼ぐ古典構成（高性能・部品多・イメージ周波数問題）。**ダイレクトコンバージョン (direct conversion / zero-IF)** は 1 段でベースバンドへ落とす（集積向き・DC オフセットと IQ 不平衡が代償）。現代のスマートフォン SoC はほぼ後者で、補正を DSP が肩代わりする——「アナログの欠陥を計算で買い戻す」の典型である。

### 部品箱と実標準の対応表

| 部品 | Wi-Fi (802.11a→ax) | LTE / 5G NR | GNSS (GPS L1 C/A) | Bluetooth (BR/LE) |
|---|---|---|---|---|
| 帯域 | 2.4/5/6 GHz 免許不要 | 免許帯 sub-6 GHz (FR1) + mmWave 24.25–52.6 GHz (FR2) | 1575.42 MHz | 2.4 GHz 免許不要 |
| 変調 | OFDM、64→1024-QAM (ax) | OFDM(A)、~256-QAM | BPSK + DSSS 拡散 | **GFSK**（+ π/4-DQPSK, 8DPSK） |
| 多元接続 | CSMA/CA → OFDMA (ax) | OFDMA（基地局スケジューリング） | CDMA（衛星ごとの拡散符号） | 周波数ホッピング (FHSS) |
| MIMO | 11n から、MU-MIMO (ac/ax) | 中核部品（massive MIMO・ビームフォーミング） | なし | なし |
| FEC | 畳み込み → LDPC (ax) | ターボ (LTE) → LDPC + Polar (NR) | なし（低レート 50 bps） | なし (BR) / 任意 (LE Coded) |
| 送信電力（規制/規格） | 〜20 dBm 級 (EIRP、地域規制) | UE Power Class 3 = **23 dBm** (TS 38.101) | 衛星→地表 受信 約 −128.5 dBm | 0 dBm 典型（Class 1 は 20 dBm） |
| 設計思想 | 免許不要帯でスループット最大化 | 免許帯で容量・保証を最大化 | 床下 20 dB からの測位・時刻 | 電池寿命最大化——**高 SNR 変調を意図的に捨てる** |

- 読み筋: 同じ部品箱から、**制約（免許の有無・電力予算・要求品質）が構成を選ぶ**。Bluetooth の GFSK は「劣った変調」ではなく、包絡線一定ゆえに PA を非線形・高効率領域で使える**低電力の最適解**である。GNSS は逆に、データレートを 50 bps まで捨てて拡散利得に全額を注いだ構成である。

## 技術詳細 (Technical Details)

### 1. 伝搬とリンクバジェット (Propagation & Link Budget)

自由空間での受信電力は **Friis の伝達公式**で与えられ、dB 化した経路損失は次式になる。

```
FSPL [dB] = 20 log₁₀(d) + 20 log₁₀(f) + 32.44   (d: km, f: MHz)

例: 2.4 GHz, 100 m → 約 80 dB   /  28 GHz (mmWave), 100 m → 約 101 dB
距離 2 倍ごとに +6 dB、周波数 10 倍ごとに +20 dB
```

リンクバジェットはこれを軸にした帳簿である。

```
受信電力 = TX 電力 + TX アンテナ利得 − 経路損失 − 諸損失 + RX アンテナ利得
雑音床   = −174 dBm/Hz + 10 log₁₀(B) + NF
リンクマージン = 受信電力 − 雑音床 − 要求 SNR      ← これが黒字なら繋がる

例: Wi-Fi 20 dBm 送信、80 MHz 帯域 (雑音床 ≈ −95 dBm + NF 6 dB ≈ −89 dBm)、
    1024-QAM の要求 SNR ~35 dB → 許容経路損失 ≈ 20 − (−89 − 35) ≈ 74 dB ≈ 数十 m
    QPSK (~5 dB) へ落とせば +30 dB ≈ 距離 30 倍超——適応変調の存在理由
```

実環境では FSPL に**シャドウイング (shadowing)**（障害物による緩慢な変動、対数正規で数 dB〜十数 dB）と**マルチパスフェージング (multipath fading)**（波長スケールの高速変動、レイリー/ライス分布で深さ数十 dB）が重畳する。マージンとは、この確率的変動に対する保険料である。

### 2. シャノン容量と符号化 (Shannon Capacity & Coding)

C = B log₂(1+SNR) の含意は三つ。(1) SNR は log でしか効かない——高 SNR 域では 3 dB 増やして 1 bit/s/Hz。(2) B は線形に効く——だが帯域は規制資源であり、雑音床も B に比例して上がる。(3) 限界は「達成可能」であって「自動的に達成される」わけではない——限界に近づく実務装置が **FEC** であり、LDPC（1960 年代の再発見、Wi-Fi 6/5G NR data で採用）と Polar 符号（5G NR 制御チャネル）はシャノン限界まで 1 dB を切る距離に迫る。**符号化利得 (coding gain)** は「同じ BER をどれだけ低い SNR で達成できるか」の dB 差であり、リンクバジェットにそのまま記帳できる——SerDes の RS-FEC と同じ会計である（[`../signal-integrity/serdes-link-architecture.md`](../signal-integrity/serdes-link-architecture.md) 発見事項 7）。

### 3. OFDM の機構 (OFDM Mechanics)

シンボルレート R のシングルキャリアでは、遅延広がり τ が 1/R に比べ無視できないと ISI が生じる（`../signal-integrity/signal-integrity-engineering.md` の有線 ISI と同型）。OFDM は帯域を N 本の直交サブキャリアに分割し、各サブキャリアのシンボル長を N 倍に伸ばす。

- **サブキャリア間隔**: 802.11a/g/n/ac は 312.5 kHz（64-point FFT / 20 MHz）、802.11ax は 78.125 kHz、LTE は 15 kHz、5G NR は 15/30/60/120 kHz の**可変**（セル半径と位相雑音——mmWave では広い間隔が要る——の交換をパラメータ化した）。
- **サイクリックプレフィックス**: シンボル末尾のコピーを先頭に付け、遅延広がり分の「のりしろ」にする（802.11 で 0.8 µs、LTE 通常 CP で約 4.7 µs——屋内と屋外セルの遅延広がりの違いがそのまま数字に出る）。帯域の数 %〜十数 % を捨てて等化を対角化する、明示的な税金である。
- 実装は IFFT（送信）/FFT（受信）そのもので、O(N log N) の FFT がなければ OFDM は経済的に成立しなかった——1990 年代以降の無線標準は、DSP の価格低下が解禁した設計である。
- 代償: サブキャリアの重ね合わせは振幅変動の大きい波形（高 PAPR）を作り、PA を線形領域（低効率）で使うことを強いる。5G NR の上りが DFT-s-OFDM（シングルキャリア的波形）を残すのは、端末 23 dBm の電力予算での PA 効率のためである。

### 4. MIMO・ダイバーシチ・ビームフォーミング (MIMO, Diversity & Beamforming)

- **ダイバーシチ**: 独立にフェージングする経路が L 本あれば、全部が同時に落ち込む確率は各々の L 乗で消える。時間（再送・インタリーブ）・周波数（ホッピング・OFDM のサブキャリア分散）・空間（複数アンテナ）のどの軸でも取れる。
- **空間多重**: N×M MIMO のチャネルは行列 H であり、散乱が豊かで H のランクが高ければ、min(N, M) 本の並列ストリームが**同じ周波数・同じ時間**に通る——容量が min(N, M) 倍にスケールする（Tse & Viswanath の中心主題）。受信側はチャネル推定で H を知り、行列を「解いて」ストリームを分離する。
- **ビームフォーミング**: 同じアンテナ群の第三の使い方——各素子の位相を揃えて特定方向へ利得を合成する。5G mmWave では FSPL の +20 dB 超をアレイ利得で買い戻すための前提条件であり、「セルが面ではなくビームの束になる」というシステム設計の転換（massive MIMO）を駆動した。
- 三者は同じハードウェアの配分問題である: 経路の独立性をダイバーシチ（信頼性）に使うか、多重（容量）に使うか、利得（距離）に使うか——チャネル状態がその都度の最適配分を決める。

## セキュリティ考察 (Security Considerations)

- **ジャミング (jamming)**: 共有媒体である無線は、物理層では妨害を排除できない——雑音床を人為的に持ち上げればリンクバジェットの黒字は消える。耐性の通貨は拡散利得と冗長性（周波数ホッピング・広帯域拡散はもともと軍用の耐妨害技術である）だが、民生 GNSS のように受信電力が床下 20 dB の信号は本質的に脆弱で、安価な妨害器の実害が空港・港湾で繰り返し報告されている。
- **スプーフィング (spoofing)**: GPS L1 C/A の民生信号は**認証を持たない平文**であり、正規より強い偽信号で受信機を「捕獲」して偽の位置・時刻を注入できる。測位だけでなく時刻同期（[`../timekeeping/clocks-time-synchronization.md`](../timekeeping/clocks-time-synchronization.md) の GNSS 時刻源、金融・電力網）への攻撃面でもある。対策は複数系統（Galileo OSNMA のような認証付き信号）・慣性との統合（[`../../engineering/aerospace/avionics-gnc.md`](../../engineering/aerospace/avionics-gnc.md) のセンサ融合）・到来方向の妥当性検査に置かれる。
- **盗聴と物理層セキュリティ**: 電波は壁で止まらず、放送である。ビームフォーミングによる空間的絞り込みや物理層秘匿 (physical-layer security) の研究はあるが、**機密性の保証は PHY では成立しない**——盗聴者のアンテナ利得・位置は制御できないからである。ゆえに暗号は MAC 層以上に住む（Wi-Fi の WPA3、セルラーの AKA、その上の TLS）——`wifi-architecture.md` の「Wi-Fi 層を信用せず上位で守る」がここで原理として基礎づけられる。
- **偽基地局 / IMSI キャッチャ**: 端末は電波の強い基地局を選ぶが、基地局は自分を証明しない（Wi-Fi の evil twin と同型）。ネットワークが端末を認証しても**端末がネットワークを認証する前**の手続き（識別子の平文送出）が突かれ、IMSI キャッチャによる追跡・ダウングレード誘導が成立してきた。5G は SUCI（公開鍵による識別子の暗号化）で平文 IMSI を廃したが、電波を強く出す者が最初の接触を握るという構造自体は残る。
- **周波数規制は可用性のガバナンスである**: 免許制度・送信電力規制（Wi-Fi の EIRP 上限、UE の 23 dBm）は、共有資源としてのスペクトラムに対する「混雑制御」であり、干渉＝サービス拒否を法制度で抑える仕組みと読める。免許不要帯の設計（`wifi-architecture.md` 発見事項 1）は、この規制設計の直接の帰結である。

## 結論 (Conclusions)

無線 PHY は「チャネルの物理は選べない」という制約に対し、**dB の帳簿（リンクバジェット）で会計し、シャノンの天井に向かって変調・符号・空間次元を積む機械**である。FSPL と kTB が床と天井の枠を張り、変調次数は SNR との交換、OFDM はマルチパスの FFT による対角化、MIMO はフェージングの資源化、FEC は dB を稼ぐ通貨——そして RF チェーンでは ADC がアンテナへ向かって前進し、無線機は徐々にソフトウェアになっていく。有線 SerDes と並べて見れば、有線も無線も「汚いチャネルを賢い受信器（計算）で読む」という同じ一文に畳める。個別の標準——Wi-Fi の世代進化、5G NR のシステム構成、GNSS の測位原理——は、この部品箱の構成として本領域の続編で扱う。

### 次の接続先 (Related Topics)

- [`../peripherals/wifi-architecture.md`](../peripherals/wifi-architecture.md) — 本ドキュメントの PHY 部品（OFDM/OFDMA・MIMO・QAM）が MAC 層の設計とどう組み合わされるか。
- [`../peripherals/bluetooth-pairing-protocol.md`](../peripherals/bluetooth-pairing-protocol.md) — GFSK・周波数ホッピングという低電力側の構成例。
- [`../signal-integrity/serdes-link-architecture.md`](../signal-integrity/serdes-link-architecture.md) / [`../signal-integrity/signal-integrity-engineering.md`](../signal-integrity/signal-integrity-engineering.md) — 有線側の同型問題（ISI・等化・FEC・BER）。
- [`../../foundations/physics/electromagnetism/maxwell-equations-em-waves.md`](../../foundations/physics/electromagnetism/maxwell-equations-em-waves.md) — 電波・アンテナの物理的土台。
- [`../timekeeping/clocks-time-synchronization.md`](../timekeeping/clocks-time-synchronization.md) — PLL・発振器・GNSS 時刻。位相雑音は EVM の床。
- [`../../engineering/aerospace/avionics-gnc.md`](../../engineering/aerospace/avionics-gnc.md) — GNSS を使う側（航法とセンサ融合）。
- [`../../software/networking/tcp-transport-congestion.md`](../../software/networking/tcp-transport-congestion.md) — 無線区間のロスを輻輳と誤解するトランスポート層の問題。PHY の確率的誤りが上位層設計に波及する接点。

## 参考文献 (References)

**一次資料 (Primary — 原論文・規格)**

- Shannon, C. E. (1948). A Mathematical Theory of Communication. *Bell System Technical Journal*, 27(3), 379–423 & 27(4), 623–656. DOI: 10.1002/j.1538-7305.1948.tb01338.x — 容量定理 C = B log₂(1+SNR) の原典。
- Friis, H. T. (1946). A Note on a Simple Transmission Formula. *Proceedings of the IRE*, 34(5), 254–256. — 自由空間伝達公式（リンクバジェットの起点）の原典。
- IEEE. *IEEE 802.11 Standards*（802.11a-1999 〜 802.11ax-2021）. — OFDM パラメータ（サブキャリア間隔・CP・変調次数・LDPC）の規範。
- 3GPP. *TS 38 series — NR (5G) 仕様群*（TS 38.101: UE 無線送受信特性、TS 38.211: 物理チャネルと変調）. — UE Power Class 3 (23 dBm)・可変サブキャリア間隔・LDPC/Polar 採用の一次資料。
- GPS Interface Specification IS-GPS-200. — L1 C/A の拡散符号・最小受信電力（−158.5 dBW 級）の規範。

**教科書 (Textbooks — 標準的典拠)**

- Tse, D. & Viswanath, P. (2005). *Fundamentals of Wireless Communication*. Cambridge University Press. — フェージング・ダイバーシチ・MIMO 容量の標準教科書（発見事項 6 の主典拠）。
- Goldsmith, A. (2005). *Wireless Communications*. Cambridge University Press. — 容量・適応変調・OFDM を含む体系的教科書。
- Rappaport, T. S. (2002). *Wireless Communications: Principles and Practice* (2nd ed.). Prentice Hall. — 伝搬モデル（FSPL・シャドウイング・マルチパス）の標準教科書。
- Razavi, B. (2011). *RF Microelectronics* (2nd ed.). Prentice Hall. — RF チェーン（LNA・ミキサ・PLL・受信機アーキテクチャ）の標準教科書。
