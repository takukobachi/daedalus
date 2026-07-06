# 自動車 (Automotive)

自動車を題材としたハードウェア分析。電子制御 (ECU) と車載ネットワーク (in-vehicle network) を中心に、組込みソフトウェアと物理システムの接点を扱う。電動化 (electrification) まで対象を広げ、EV と物理・ソフトの統合を扱う。

## ドキュメント

- [`in-vehicle-networks.md`](in-vehicle-networks.md) — 車載ネットワークと ECU (in-vehicle networks & ECU)：CAN を軸に調停・堅牢性・無認証問題
- [`automotive-ethernet-tsn.md`](automotive-ethernet-tsn.md) — 車載 Ethernet と TSN (Automotive Ethernet & TSN)：SPE・TSN・FlexRay・LIN・CAN FD/XL・SOME/IP
- [`ee-architecture.md`](ee-architecture.md) — E/E アーキテクチャ総論 (E/E architecture)：分散 → ドメイン → ゾーン＋セントラルの変遷、配線ハーネスの物理、eFuse 配電、mixed-criticality、SDV/OTA
- [`functional-safety-iso26262.md`](functional-safety-iso26262.md) — 機能安全 (functional safety, ISO 26262)
- [`cybersecurity-iso21434.md`](cybersecurity-iso21434.md) — 自動車サイバーセキュリティ (automotive cybersecurity)：ISO/SAE 21434・UN R155/R156・HSM・SecOC・セキュア OTA

## サブディレクトリ

- [`ev/`](ev/) — 電気自動車 (electric vehicles)：EV パワートレインの基礎工学と **テスラ**（バッテリ/4680・パワートレイン・E/E アーキ・製造）

## 今後のテーマ候補

現状のカバレッジ（CAN・車載 Ethernet/TSN・E/E アーキテクチャ・機能安全・EV）を踏まえ、優先度順に調査対象を整理する。

### 優先度 中 (物理システムを厚くする)

- **ADAS / 自動運転** — センサ物理（カメラ・mmWave レーダ・LiDAR・超音波・IMU/GNSS）、センサ融合、SAE J3016 レベル、SoC（NVIDIA Orin/Thor・Mobileye EyeQ・Qualcomm Ride・Tesla FSD）。
- **シャシー制御と X-by-wire** — ABS/ESC/brake-by-wire、EPS/steer-by-wire、アクティブサスペンション、冗長とフォールバック（機能安全と直結）。
- **パワートレイン制御（ICE / ハイブリッド）** — 燃料噴射・点火制御、OBD-II/排出ガス、ハイブリッド方式（Toyota THS 等のパワースプリット）。
- **車載半導体** — 車載 MCU（Infineon AURIX・Renesas RH850・NXP S32）、ロックステップコア、パワー半導体（SiC/GaN）、AEC-Q100 認定。

### 優先度 低〜中 (周辺を仕上げる)

- **車載ソフト基盤** — AUTOSAR Classic/Adaptive、SOME/IP、DDS、ハイパーバイザ（mixed-criticality）。
- **検証・車載環境** — HIL/SIL/MIL、ISO 16750（環境試験）、EMC/EMI、振動・温度。
- **HMI / デジタルコックピット** — IVI、Android Automotive、QNX。
