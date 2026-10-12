# 電動機・モータドライブ (Motors)

電動機 (electric motor) とその駆動装置（インバータ・制御）を題材とした工学分析。世界の電力の 4〜5 割を消費する「電気から回転への変換」を、トルク発生の物理・dq 座標の制御・パワーエレクトロニクス・規格と安全の各層に分けて分析する。

EV のトラクション（[`../automotive/ev/`](../automotive/ev/)）、ロボット関節（[`../robotics/`](../robotics/)）、系統側の電力変換（[`../energy/power-electronics-grid-converters.md`](../energy/power-electronics-grid-converters.md)）に分散しているモータと駆動の記述について、共通する部分をまとめるハブである。用途固有の話題（電池・車両統合、減速機・関節設計、系統連系制御）は各ディレクトリに任せ、本ディレクトリは機種・制御・ドライブ・規格の横断的な部分を扱う。

## ドキュメント

- [`electric-motor-drives.md`](electric-motor-drives.md) — 電動機とモータドライブ (electric motors & motor drives)。トルク発生の物理、機種比較（DC/BLDC/SPM/IPM/IM/SynRM/EESM/SRM/アキシャルフラックス）、Clarke/Park 変換と FOC カスケード、MTPA・弱め界磁、センサとセンサレス (HFI)、SiC の dv/dt・軸受電流・絶縁、損失と減磁、回生、IEC 60034-30-1 (IE1–IE5)・EU 2019/1781・IEC 61800-5-2 (STO)、故障モード、駆動装置のセキュリティと希土類の供給リスク

## 今後のテーマ候補

- 電動機の設計と材料 (machine design & materials) — 巻線方式（ヘアピン・集中巻）、電磁鋼板、磁石の粒界拡散と重希土類の削減、有限要素解析
- 状態監視と予知保全 (condition monitoring) — 電流スペクトル解析 (MCSA)、振動・部分放電の監視、寿命モデル
- 中圧ドライブと大型電動機 (medium-voltage drives) — マルチレベル変換器、圧縮機・ポンプ・船舶推進。[`../energy/power-electronics-grid-converters.md`](../energy/power-electronics-grid-converters.md) の NPC/MMC と接続
- リニアモータと磁気浮上 (linear motors & maglev) — 半導体製造装置のステージ、リニア新幹線
