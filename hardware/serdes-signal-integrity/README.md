# 高速シリアルリンクとシグナルインテグリティ (SerDes & Signal Integrity)

PCIe (`../hw-sw-interface/bus-interconnect.md`)・USB/ディスプレイ (`../peripherals-connectivity/`)・
Ethernet (`../network-hardware/`)・車載 Ethernet (`../automotive/`) が論理層で共有している
**共通の物理層**の分析。数十 Gbit/s の信号を損失だらけの銅線で運ぶための送受信器
(SerDes: serializer/deserializer) の構造と、リンクが「本当に動く」ことを定義・検証する
工学 (signal integrity engineering) を扱う。

伝送線路の理論（電信方程式・反射・終端・損失の物理）は
[`../../foundations/physics/electromagnetism/transmission-lines-signal-integrity.md`](../../foundations/physics/electromagnetism/transmission-lines-signal-integrity.md)
が供給する。本領域はその上の**工学の層**である回路・符号・プロトコル・測定に徹する。

## ドキュメント

- [`serdes-link-architecture.md`](serdes-link-architecture.md) — SerDes リンクアーキテクチャ：
  パラレルバスの死とシリアル化、符号化の系譜 (8b/10b → 64b/66b → 128b/130b)、
  NRZ vs PAM4 の損益計算、イコライゼーションの積層 (TX FFE / CTLE / DFE / ADC-DSP)、
  CDR とクロック埋め込み、リンクトレーニング、FEC (RS-FEC)、リタイマ、リンク暗号化 (PCIe IDE)
- [`signal-integrity-engineering.md`](signal-integrity-engineering.md) — シグナルインテグリティ
  工学：BER という唯一の真の仕様、ジッタの分解 (RJ/DJ) とバスタブ曲線、アイダイアグラムと
  マスク、S パラメータによるチャネルバジェット (IL/ILD/リターンロス/クロストーク)、
  ビアスタブと不連続、COM (channel operating margin)、測定器とコンプライアンス試験、
  電源インテグリティとの一体性

## 今後のテーマ候補

- 電源インテグリティの各論 (power integrity) — PDN 設計、ターゲットインピーダンス、
  デカップリング階層。`signal-integrity-engineering.md` で導入した「リターン電流の道」の深掘り
- パッケージとチップレット間リンク (UCIe & package-level links) — 2.5D/3D 実装での
  超短距離・超広幅リンク。`../semiconductor-manufacturing/` の先端パッケージングとの合流点
- 光への移行の工学 — リニアドライブ光学 (LPO)・共封止光学 (CPO) の電気側要件。
  `../network-hardware/` の光インターコネクト候補の SerDes 側
