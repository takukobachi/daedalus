# ミクストシグナル (Mixed-Signal)

アナログの物理世界とデジタルの計算機の境界にある回路の分析。中心は AD/DA 変換器で、
標本化と量子化の数学 ([`../../foundations/mathematics/fourier-signal-processing/`](../../foundations/mathematics/fourier-signal-processing/))
を前提に、変換器の方式、性能指標、熱雑音とジッタという物理の限界、混載 SoC での実装を扱う。
変換器を使う側の各論、つまりオーディオ・センサ・カメラは `../peripherals/`、無線の RF チェーンは
`../wireless/`、ADC 型 SerDes 受信器は `../signal-integrity/` にあり、本領域はそれらが共有する
「境界の部品」を受け持つ。標本化クロックの源は `../timekeeping/` と `../hw-sw-interface/clock-power-reset.md` が扱う。

## ドキュメント

- [`adc-dac-converters.md`](adc-dac-converters.md) — AD/DA 変換器：標本化と量子化の要点、
  性能指標 (SNR・SINAD・ENOB・SFDR・INL/DNL・NSD)、熱雑音 (kT/C) とクロックジッタによる限界、
  ADC の方式 (フラッシュ・SAR・パイプライン・ΔΣ・タイムインターリーブ)、Walden/Schreier の性能指数と
  Murmann の ADC Performance Survey、DAC の方式とグリッチ、基準電圧とクロック、混載 SoC の雑音結合、
  応用 (オーディオ・センサ・直接 RF 標本化・ADC 型 SerDes)、境界を突く攻撃 (エイリアシング・雑音結合)

## 今後のテーマ候補

- PLL と周波数合成 (PLLs & frequency synthesis) — 位相雑音とジッタの生成側。本領域の「ジッタの壁」の源を
  回路から扱う。`../timekeeping/` の発振器と `../hw-sw-interface/clock-power-reset.md` の間を埋める
- アナログフロントエンド (analog front ends) — 低雑音増幅器、PGA、計装アンプ、センサの読み出し回路。
  ADC の前段で決まる雑音とオフセット
