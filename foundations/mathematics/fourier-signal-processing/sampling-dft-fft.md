# サンプリング定理と DFT/FFT (Sampling Theorem, DFT, and FFT)

## 概要 (Overview)

フーリエ解析・信号処理シリーズの第2回。前回はあらゆる信号を正弦波の重ね合わせとして読む言語を整備した——本回はその言語を使って、**連続な物理世界と離散的な計算機を橋渡しする条件**を確定させる。橋の名は **ADC (analog-to-digital converter)** と **DAC (digital-to-analog converter)** であり、マイクの電圧も、アンテナに届いた電波も、イメージセンサの光量も、計算機に入る瞬間に必ず2つの離散化を通る: 時間軸を刻む**サンプリング (sampling)** と、振幅軸を刻む**量子化 (quantization)**。

驚くべき主張は、第一の離散化が——条件さえ守れば——**情報を一切失わない**ことである(サンプリング定理)。条件を破ったときの罰がエイリアシングであり、第二の離散化(量子化)の代価はノイズとして定量化できる。後半では、離散化された信号のためのフーリエ変換——**DFT**——と、その計算量を O(N²) から O(N log N) へ潰した **FFT** を扱う。FFT は単なる高速化ではない: スペクトル解析と OFDM 変復調を「毎サンプル実行できる安価な演算」に変えたことで、現代の無線・オーディオ・画像処理の経済性そのものを成立させたアルゴリズムである。前回の道具立て——デルタ関数 (§5)、畳み込み定理 (§4)、双対性 (§3)——がここで全て回収される。

## 前提知識 (Prerequisites)

- [`fourier-series-transform.md`](fourier-series-transform.md) —
  本シリーズ第1回。デルタ関数、畳み込み定理、変換対と双対性を全面的に使う
- [`../calculus-analysis/limits-derivatives-integrals.md`](../calculus-analysis/limits-derivatives-integrals.md) —
  積分と極限操作、級数の収束
- [`../probability-statistics/probability-random-variables.md`](../probability-statistics/probability-random-variables.md) —
  量子化誤差を一様分布の確率変数としてモデル化する (§4) ための確率論

## 本論 (Main Content)

### 1. インパルス列サンプリング — スペクトルの複製 (impulse-train sampling replicates the spectrum)

信号 x(t) を間隔 T_s(サンプリング周波数 f_s = 1/T_s)で標本化する操作は、数学的には**インパルス列 (impulse train)** `s(t) = Σₙ δ(t − nT_s)` との**積**である。ここで前回の2つの事実が合流する: インパルス列のフーリエ変換もまたインパルス列(間隔 f_s の線スペクトル)であり、時間領域の積は周波数領域の**畳み込み**である(畳み込み定理の双対、前回 §4)。デルタ関数との畳み込みは平行移動だから、次式が成り立つ。

```text
x_s(t) = x(t)·Σₙ δ(t − nT_s)   ⟷   X_s(f) = (1/T_s)·Σₖ X(f − k·f_s)
```

すなわち**サンプリングとは、元のスペクトル X(f) を f_s の整数倍ごとに無限に複製する操作**である。原信号の情報は失われていない——複製が互いに重ならない限り、k = 0 の複製(ベースバンド)がそのまま原スペクトルだからである。

### 2. サンプリング定理と再構成 (the sampling theorem and reconstruction)

複製の間隔は f_s、各複製の幅は信号の帯域で決まる。信号が **帯域制限 (band-limited)**——最高周波数 f_max 以上の成分を持たない——なら、複製が重ならない条件は次式で与えられる。

```text
f_s > 2·f_max   (サンプリング定理。f_s/2 をナイキスト周波数 (Nyquist frequency) と呼ぶ)
```

このとき再構成は概念的に単純である: 複製されたスペクトルから k = 0 だけを**理想ローパスフィルタ**(遮断 f_s/2 の矩形)で切り出せばよい。周波数領域の矩形は時間領域では sinc(前回 §3 の変換対の逆向き)だから、時間領域で見ると次式になる。

```text
x(t) = Σₙ x(nT_s)·sinc(π(t − nT_s)/T_s)   (sinc 補間 — 標本点の間を「唯一の帯域制限信号」で埋める)
```

各標本点に sinc を立てて足すと、標本点上では自分以外の sinc が全て零点を踏むため標本値が厳密に再現され、標本点の間は帯域制限が許す唯一の滑らかな波形で補間される。**離散的な数列が連続波形を一意に決める**——これが DAC の数学的正当化であり、「CD の 44.1 kHz は 22.05 kHz までの音を完全に記録している」という言明の正確な意味である。実在の DAC は理想 sinc を打ち切ったフィルタで近似するが、原理はこの1行に尽きる。

### 3. エイリアシング — 折り返しという偽名 (aliasing: frequencies in disguise)

条件 f_s > 2f_max が破れると複製が重なり、f_s/2 を超えた成分がベースバンドへ**折り返す (folding)**。周波数 f の成分は `|f − k·f_s|` のうちナイキスト帯域に落ちる周波数として観測される——高い周波数が低い周波数の**偽名 (alias)** を名乗る。映画で車輪が逆回転して見える**ワゴンホイール効果**が原型的直観である: コマ撮り(= サンプリング)の間に車輪がほぼ1回転すると、脳は「少し戻った」という最も低い周波数の解釈を選ぶ。イメージセンサの画素格子より細かい縞模様が粗い縞——**モアレ (moiré)**——に化けるのは、同じ現象の2次元空間周波数版である。

折り返しの致命的な点は、**サンプリング後には除去不能**なことである——偽名を名乗った成分は正規の低周波成分と区別がつかない。したがって対策はサンプリングの**前**、アナログ領域に置くしかない。

```text
アンチエイリアシングフィルタ (anti-aliasing filter):
  ADC の直前に置くアナログローパス。f_s/2 以上を「サンプリングされる前に」殺す
```

逆に、折り返しを**意図的に使う**こともできる。帯域幅 B の信号が高い搬送波周波数に乗っている場合、f_s > 2B(搬送波周波数の2倍ではなく帯域幅の2倍)を満たす低いレートでサンプリングし、折り返しで信号をベースバンドへ落とす——**アンダーサンプリング (undersampling, 帯域通過サンプリング)** であり、ソフトウェア無線の RF フロントエンドがミキサの代わりに ADC の折り返しを周波数変換器として使う技法である (Lyons 2011, Ch. 2)。

### 4. 量子化 — 振幅の離散化とノイズ (quantization: amplitude discretization as noise)

時間の離散化が(条件付きで)無損失だったのに対し、振幅を N ビット・2^N 段に丸める**量子化**は不可逆な誤差を必ず残す。誤差は決定論的だが、信号が量子化ステップ Δ に比べて複雑に動く限り、**±Δ/2 の一様分布に従う独立な確率変数**としてモデル化してよい(前提知識の確率論)。一様分布の分散は Δ²/12 であり、フルスケールの正弦波に対する信号対雑音比 (SNR) を計算すると次式を得る。

```text
SNR ≈ 6.02·N + 1.76  [dB]   (1 ビット追加するごとに約 6 dB 改善)
```

「16 bit オーディオ ≈ 98 dB」「12 bit ADC ≈ 74 dB」という換算はこの式である。モデルが破綻するのは信号が小さく Δ と同程度になるとき——誤差が信号と相関し、ノイズではなく歪みとして聞こえ/見える。対策が**ディザ (dither)**: 量子化の前に微小な乱数を加えて誤差の相関を意図的に壊し、歪みを無害な広帯域ノイズに交換する(概観に留める)。

さらに強力なのが**オーバーサンプリング (oversampling)** との合わせ技である。量子化ノイズの総パワーは一定だが、f_s を上げるとそれが広い帯域に薄く撒かれるため、信号帯域内のノイズは減る。**ノイズシェーピング (noise shaping)** はこれを一歩進め、フィードバックループでノイズのスペクトルを高域へ押し出す——粗い量子化器(極端には 1 bit)を高速に回してループで整形する構成が **ΔΣ (delta-sigma) 変調**であり、オーディオ ADC/DAC の主流方式である。直観だけ述べれば: **振幅分解能を時間分解能で買う**取引であり、§2 の双対性の実務的な現金化である。

### 5. DFT — 有限・離散の両端を閉じる (the DFT: finite and discrete on both axes)

計算機が扱えるのは有限個の数列だけである。時間側を N 点に打ち切り、周波数側も N 点に離散化した変換が**離散フーリエ変換 (discrete Fourier transform, DFT)** である。

```text
X[k] = Σₙ₌₀^{N−1} x[n]·e^{−i2πkn/N}   (k = 0, …, N−1)
周波数分解能: Δf = f_s/N   (= 1/記録長 — 長く録るほど細かく見える)
```

構造は前回のフーリエ級数の係数式そのものであり、暗黙の仮定も同じ——**DFT は N 点の記録が周期 N で無限に繰り返すと仮定して**スペクトルを求める。この仮定が2つの実務的帰結を生む。第一に**分解能と記録長の交換**: Δf = 1/(N·T_s) だから、10 Hz 刻みでスペクトルを見たければ最低 0.1 秒の記録が要る——時間と周波数の双対性(前回 §3)の離散版である。第二に**スペクトル漏れ (spectral leakage)**: 記録の端で信号が「つながらない」と、周期延長の継ぎ目に人工的な不連続——前回 §5 の言葉で言えば偽のエッジ——が生まれ、その高周波成分が本来無いはずの周波数ビンへ漏れ出す。対策が**窓関数 (window function)**: 記録の両端を滑らかに 0 へ絞る重み(Hann、Blackman 等)を掛けて継ぎ目のエッジを消す。代価は主ローブの広がり——漏れの抑圧と分解能はここでも交換関係にある (Oppenheim & Schafer 2010, Ch. 10)。

### 6. FFT — 分割統治が変えた経済性 (the FFT: divide and conquer changes the economics)

DFT の定義通りの計算は N 点につき N² 回の複素乗算を要する。**FFT (fast Fourier transform)** は、N が 2 の冪のとき DFT を偶数番標本と奇数番標本の2つの N/2 点 DFT に分解できることを使う——回転因子 `e^{−i2πkn/N}` の周期性により、半分のサイズの結果を**バタフライ (butterfly)** と呼ばれる「和と差に回転因子を掛ける」2点演算で組み合わせるだけで全体が復元される。分割を log₂N 段繰り返すと、計算量は次のように下がる。

```text
計算量: O(N²) → O(N·log N)
N = 1024: 約 100 万回 → 約 1 万回 (100 倍)。N = 10⁶ なら 5 万倍
```

Cooley & Tukey (1965) の再発見(ガウスに遡る先行はある)が信号処理の分水嶺になったのは、この差が「できる/できない」の差だからである。1024 点のスペクトル解析が 100 倍安くなれば、リアルタイムで毎フレーム実行できる。Wi-Fi や LTE の **OFDM** は数十〜数千本の直交搬送波への同時変復調(前回 §1 の直交射影)を**シンボルごとに** IFFT/FFT 一発で行う——FFT なくして OFDM は計算コスト的に成立せず、逆に言えば、現代の無線 PHY とは「FFT が安価である」という前提の上に設計された変調方式である。

## 工学への接続 (Engineering Connections)

- **ADC/DAC は物理と計算の関所**: センサからプロセッサまでの経路で、本回の4条件——帯域制限、f_s > 2f_max、十分なビット数、再構成フィルタ——が信号チェーンの仕様書そのものになる。「44.1 kHz/16 bit」「4K@60 Hz/10 bit」という製品仕様は、サンプリング定理と 6 dB/bit 則の直接の書き下しである。
- **アンチエイリアシングは物理配置の問題**: フィルタは ADC より前=アナログ側に置くしかない (§3)。カメラの光学ローパスフィルタ(センサの手前の複屈折板)、オーディオ ADC 前段の RC——「デジタルで後から直す」が原理的に効かない稀有な例。
- **ΔΣ の分業**: 緩いアナログフィルタ+高速で粗い量子化器+デジタルの急峻な間引きフィルタ (§4)。高精度アナログ回路の困難を、微細化で安くなる一方のデジタル演算に転嫁する——ADC 設計の過去30年はほぼこの一方向への移動である。
- **FFT ベースの計測と変調**: スペクトラムアナライザ、OFDM、レーダーのパルス圧縮、SerDes の ADC ベース受信器の周波数領域イコライザ——「一旦周波数領域へ行って戻る」設計が成立する損益分岐点は FFT の O(N log N) が決めている (§6)。
- **窓と漏れは測定の教養**: 実測スペクトルの裾が理論より持ち上がっていたら、まず疑うのは信号ではなく窓 (§5)。分解能・漏れ・記録長の三角関係を知らないと、計測器の表示を誤読する。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/peripherals-connectivity/audio-subsystem.md`](../../../hardware/peripherals-connectivity/audio-subsystem.md) | サンプルレートと可聴帯域 (§2–3)、ビット深度と SNR (§4)、ΔΣ 型 ADC/DAC (§4) |
| [`hardware/peripherals-connectivity/camera-image-sensors.md`](../../../hardware/peripherals-connectivity/camera-image-sensors.md) | 画素格子 = 2次元空間サンプリング、モアレと光学ローパスフィルタ (§3) |
| [`hardware/peripherals-connectivity/display-pipeline.md`](../../../hardware/peripherals-connectivity/display-pipeline.md) | スケーリング = 再サンプリングと補間フィルタ (§2)、リサイズ時のエイリアシング (§3) |
| [`hardware/peripherals-connectivity/wifi-architecture.md`](../../../hardware/peripherals-connectivity/wifi-architecture.md) | OFDM の IFFT/FFT 変復調 (§6)、サブキャリア間隔と直交性 (§5) |
| [`hardware/wireless-rf/radio-fundamentals-modulation.md`](../../../hardware/wireless-rf/radio-fundamentals-modulation.md) | RF チェーンの ADC/DAC 配置、帯域通過サンプリング (§3)、SDR の成立条件 |
| [`hardware/serdes-signal-integrity/serdes-link-architecture.md`](../../../hardware/serdes-signal-integrity/serdes-link-architecture.md) | ADC ベース受信器——リンクを「サンプリング+DSP」で受ける現代 SerDes (§1–2, §6) |

## 参考文献 (References)

- Oppenheim, A. V., & Willsky, A. S. (1996). *Signals and Systems* (2nd ed.). Prentice Hall. — インパルス列サンプリングとサンプリング定理 (Ch. 7) の主参照
- Oppenheim, A. V., & Schafer, R. W. (2010). *Discrete-Time Signal Processing* (3rd ed.). Prentice Hall. — DFT・窓関数・FFT アルゴリズムの標準教科書
- Lyons, R. G. (2011). *Understanding Digital Signal Processing* (3rd ed.). Prentice Hall. — エイリアシング・量子化・帯域通過サンプリングの工学的直観の主参照
- Cooley, J. W., & Tukey, J. W. (1965). An Algorithm for the Machine Calculation of Complex Fourier Series. *Mathematics of Computation*, 19(90), 297–301. DOI: 10.2307/2003354 — FFT の原論文
