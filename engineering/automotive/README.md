# 自動車 (Automotive)

自動車を題材とした工学分析。電子制御 (ECU) と車載ネットワーク (in-vehicle network) を中心に、組込みソフトウェアと物理システムの接点を扱う。電動化 (electrification) まで対象を広げ、EV と物理・ソフトの統合を扱う。

## ドキュメント

- [`in-vehicle-networks.md`](in-vehicle-networks.md) — 車載ネットワークと ECU (in-vehicle networks & ECU)：CAN を軸に調停・堅牢性・無認証問題
- [`automotive-ethernet-tsn.md`](automotive-ethernet-tsn.md) — 車載 Ethernet と TSN (Automotive Ethernet & TSN)：SPE・TSN・FlexRay・LIN・CAN FD/XL・SOME/IP
- [`ee-architecture.md`](ee-architecture.md) — E/E アーキテクチャ総論 (E/E architecture)：分散 → ドメイン → ゾーン＋セントラルの変遷、配線ハーネスの物理、eFuse 配電、mixed-criticality、SDV/OTA
- [`functional-safety-iso26262.md`](functional-safety-iso26262.md) — 機能安全 (functional safety, ISO 26262)
- [`cybersecurity-iso21434.md`](cybersecurity-iso21434.md) — 自動車サイバーセキュリティ (automotive cybersecurity)：ISO/SAE 21434・UN R155/R156・HSM・SecOC・セキュア OTA
- [`adas-sensing-perception.md`](adas-sensing-perception.md) — ADAS のセンシングと知覚 (ADAS sensing & perception)：カメラ・77 GHz FMCW/4D イメージングレーダ・LiDAR (ToF/FMCW)・超音波・IMU/GNSS/RTK・HD マップ、早期/後期融合、BEVFormer 等の BEV 知覚、占有ネットワーク、camera-only 対 multi-sensor 論争
- [`autonomous-driving-stack-safety.md`](autonomous-driving-stack-safety.md) — 自動運転スタック・計算基盤・安全保証 (autonomous driving stack, compute & safety)：SAE J3016 と ODD、モジュール型対 end-to-end（UniAD・FSD v12+）、SoC（DRIVE Orin/Thor・EyeQ6・Tesla HW3/HW4/AI5）、ISO 21448 SOTIF・UL 4600・RSS・UN R157、検証手法と 2026 年 10 月時点の展開状況
- [`chassis-control-x-by-wire.md`](chassis-control-x-by-wire.md) — シャシー制御と X-by-wire (chassis control & X-by-wire)：ABS/ESC のスリップ・ヨーレート制御、EHB/EMB、EPS とステア・バイ・ワイヤ、アクティブサスペンション、fail-operational 冗長と ASIL 分解、UN R13-H/R140/R79・FMVSS 126・中国 GB の 2026 年 11 月時点の規制状況

## サブディレクトリ

- [`ev/`](ev/) — 電気自動車 (electric vehicles)：EV パワートレインの基礎工学と **テスラ**（バッテリ/4680・パワートレイン・E/E アーキ・製造）

## 今後のテーマ候補

現状のカバレッジ（CAN・車載 Ethernet/TSN・E/E アーキテクチャ・機能安全・サイバーセキュリティ・ADAS/自動運転・シャシー制御と X-by-wire・EV）を踏まえ、優先度順に調査対象を整理する。

### 優先度 中 (物理システムを厚くする)

- **パワートレイン制御（ICE / ハイブリッド）** — 燃料噴射・点火制御、OBD-II/排出ガス、ハイブリッド方式（Toyota THS 等のパワースプリット）。
- **車載半導体** — 車載 MCU（Infineon AURIX・Renesas RH850・NXP S32）、ロックステップコア、パワー半導体（SiC/GaN）、AEC-Q100 認定。

### 優先度 低〜中 (周辺を仕上げる)

- **車載ハイパーバイザと mixed-criticality の実装比較** — AUTOSAR Classic/Adaptive・SOME/IP・DDS は [`software/embedded/autosar-vehicle-software.md`](../../software/embedded/autosar-vehicle-software.md) で扱い済み。残るのは、安全認証済みハイパーバイザ（QNX Hypervisor 等）の比較と、HPC 上で ASIL 系と QM 系を同居させる実装の詳細。
- **検証・車載環境** — HIL/SIL/MIL、ISO 16750（環境試験）、EMC/EMI、振動・温度。
- **HMI / デジタルコックピット** — IVI、Android Automotive、QNX。
