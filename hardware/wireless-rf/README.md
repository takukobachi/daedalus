# 無線・RF (Wireless & RF)

**広域・高速側の無線**の分析。近距離無線の各論 (Wi-Fi の MAC 層・Bluetooth) は
`../peripherals-connectivity/` でカバー済みであり、本領域はその**下の物理層理論**
である電波伝搬・リンクバジェット・シャノン容量・変調・OFDM・MIMO と、
セルラー (LTE/5G NR)・GNSS といった広域システムを扱う。

電磁波・アンテナの物理は
[`../../foundations/physics/electromagnetism/maxwell-equations-em-waves.md`](../../foundations/physics/electromagnetism/maxwell-equations-em-waves.md)
が供給する。有線側の同型問題 (ISI・等化・FEC) は、兄弟領域の
`../serdes-signal-integrity/` が扱う。

## ドキュメント

- [`radio-fundamentals-modulation.md`](radio-fundamentals-modulation.md) — 無線通信の
  基礎：電波伝搬とリンクバジェット (FSPL/Friis・kTB・雑音指数)、シャノン容量、
  変調の系譜 (OOK/FSK → QAM) と EVM、OFDM/OFDMA (サイクリックプレフィックスと
  FFT による等化の対角化)、MIMO・ダイバーシチ・ビームフォーミング、
  RF チェーン構成 (スーパーヘテロダイン vs ダイレクトコンバージョン、SDR への方向)、
  実標準 (Wi-Fi・LTE/5G・GNSS・Bluetooth) への対応表

## 今後のテーマ候補

- 5G NR システムアーキテクチャ — RAN/コアの分割、ネットワークスライシング、
  スケジューリング。基礎文書の「部品箱」がセルラーでどう構成されるかの各論
- GNSS 深掘り — 測位方程式・拡散符号・受信機構造・補強系 (RTK/SBAS)。
  `../timekeeping/clocks-time-synchronization.md` の時刻源、`../aerospace/avionics-gnc.md`
  の航法との合流点
- mmWave とビームマネジメント — FR2 の伝搬特性とアレイ設計。massive MIMO の実装側
