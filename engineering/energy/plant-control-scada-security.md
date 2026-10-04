# 発電所・系統の制御システムとセキュリティ (Plant Control Systems: SCADA / DCS / IEC 61850 & Security)

## 概要 (Overview)

[`generation-grid-fundamentals.md`](generation-grid-fundamentals.md) は、発電・系統の物理的な制御点（ガバナ、AGC、AVR、保護リレー）が最終的にソフトウェアへ集約されることを示した。本ドキュメントはその「集約先」— **発電所・電力系統を監視制御する制御システム (control system)** — を主題とする。

電力の制御システムは典型的な **OT (Operational Technology)** であり、一般の IT とは設計思想が根本的に異なる。物理プロセスを止めないことが最優先であり、リアルタイム性・可用性・決定性 (determinism) が機密性より重視される。本ドキュメントは、(1) 制御システムの階層構造（フィールド機器 → PLC/RTU/IED → SCADA/DCS → EMS）、(2) それらを結ぶ通信プロトコル（Modbus、DNP3、IEC 60870-5-104、IEC 61850）、(3) リアルタイム要求と安全計装、(4) これらが「無認証・長寿命・物理直結」という特性ゆえに抱えるセキュリティ課題を、防御・教育の観点から整理する。

OT の制御という点で、本ドキュメントは [`../automotive/in-vehicle-networks.md`](../automotive/in-vehicle-networks.md)（無認証バス CAN）や [`../automotive/functional-safety-iso26262.md`](../automotive/functional-safety-iso26262.md)（安全が支配的要件）と同じ構造的課題を、社会インフラ規模で扱う。

## 方法論 (Methodology)

- 制御システムを **Purdue モデル (Purdue Enterprise Reference Architecture)** に沿って階層化し、各層の機器・役割・時間スケールを対応づける。
- 通信を「何を運ぶか（テレメトリ／制御指令／保護トリップ）」「決定性の要求」「認証の有無」の3軸で分解し、各プロトコルがどの象限を埋めるかを示す。
- 安全 (safety) と保安 (security) を区別する。安全は「偶発故障から物理を守る」、保安は「意図的攻撃から守る」であり、両者は独立した層として設計される。
- セキュリティは公開済みの実インシデント（Stuxnet、ウクライナ電力網攻撃）を題材に、攻撃の「構造」と防御原則のみを扱い、再現可能な攻撃手法は記述しない（リポジトリの倫理方針に準拠）。
- 数値・世代は代表的な運用の桁感を示すものであり、実システムは規模・規制・ベンダで大きく異なる。

## 発見事項 (Findings)

1. **OT の脅威モデルは IT と逆。** 一般 IT が機密性 (Confidentiality) を最優先するのに対し、電力 OT は **可用性 (Availability) → 完全性 (Integrity) → 機密性** の順で重視する。プロセスを止めること自体が最大の被害になりうるため、「とりあえず再起動」「パッチ即適用」といった IT の常識がそのまま使えない。

2. **制御は階層化され、時間スケールで分業される。** マイクロ秒〜ミリ秒の保護リレー、ミリ秒〜秒のローカル閉ループ制御 (PLC/DCS)、秒〜分の監視制御 (SCADA)、分〜時間の系統最適化 (EMS) が層をなす。上位ほど広域・低頻度、下位ほど局所・高頻度かつ決定的。

3. **主要プロトコルの多くは認証を前提に設計されていない。** Modbus、DNP3、IEC 60870-5-104 の原仕様は、信頼された閉域網を前提とし、送信者認証・暗号を持たない。電力網が IP 化・遠隔化されたことで、この前提が成り立たないまま露出した。

4. **IEC 61850 は変電所自動化を標準化したが、決定性ゆえの脆さも持つ。** GOOSE や Sampled Values はイーサネット上をマルチキャストで低遅延伝送するが、L2 ブロードキャストで認証が薄く、保護トリップという致命的機能を運ぶ。

5. **安全計装系 (SIS) が最後の砦。** 制御系 (DCS/SCADA) が侵害・誤動作しても、独立した安全計装系 (Safety Instrumented System) が物理的限界を超えさせない。ただし TRITON/TRISIS のように SIS 自体を狙う攻撃も観測されており、最後の砦も防御対象である。

6. **物理直結ゆえ、サイバー侵害が物理破壊・広域停電に直結する。** Stuxnet は制御ロジック改ざんで遠心分離機を物理破損させ、ウクライナ攻撃は遠隔操作で実際に停電を起こした。サイバーと物理の境界が薄いことが、この領域のセキュリティを社会インフラ問題にしている。

## アーキテクチャ (Architecture)

### 制御システムの階層 (Purdue モデル)

電力の制御システムは、フィールドの物理機器から企業 IT まで階層化される。下位ほどリアルタイムで決定的、上位ほど広域で情報指向である。

```
 Level 4/5  企業IT (Enterprise / IT)        … ERP, 業務系。インターネット接続あり
 ──────────────── IT / OT 境界 (DMZ, ファイアウォール, 一方向GW) ────────────────
 Level 3   運用管理 (Operations)           … 履歴DB(historian), MES, パッチ/AV配信
 Level 2   監視制御 (Supervisory)          … SCADA HMI, DCS オペレータ卓, EMS
 Level 1   制御 (Control)                  … PLC, RTU, IED, 保護リレー (閉ループ制御)
 Level 0   フィールド (Process / Field)     … センサ, アクチュエータ, 遮断器, 計器用変成器
```

| 層 | 代表機器 | 主機能 | 時間スケール |
|---|---|---|---|
| L0 フィールド | センサ、アクチュエータ、遮断器 (CB) | 物理量の測定・操作 | 連続／μs |
| L1 制御 | PLC、RTU、IED、保護リレー | ローカル閉ループ制御、保護トリップ | μs〜s |
| L2 監視 | SCADA/HMI、DCS、EMS | 集中監視、指令、AGC | s〜min |
| L3 運用 | Historian、MES | 記録、分析、配信 | min〜 |
| L4/5 企業 | ERP、業務系 | 経営・事務 | — |

- **SCADA (Supervisory Control and Data Acquisition)**: 地理的に分散した設備（変電所、発電所、配電）を遠隔監視・制御する。広域・ポーリング指向で、テレメトリ収集と遮断器開閉などの指令が中心。
- **DCS (Distributed Control System)**: 単一プラント内（火力・原子力など）のプロセスを連続制御する。閉ループ制御が密で、調節弁・ポンプ等を秒以下で制御。
- **RTU / PLC / IED**: フィールドとの接点。RTU はテレメトリ中継、PLC はロジック制御、IED (Intelligent Electronic Device) は保護・計測を担うインテリジェント機器（保護リレーが代表）。
- **EMS (Energy Management System)**: 系統全体の最適化。状態推定 (state estimation)、AGC、経済負荷配分 (economic dispatch)、緊急時解析を行う給電所の頭脳。

### 通信プロトコルの位置づけ

```
 [企業/運用]                  [監視: SCADA/EMS]              [制御: RTU/IED]      [フィールド]
                                    │                            │
   業務系 ── DMZ ── Historian ──── EMS/SCADA ── DNP3/T104 ───── RTU ── Modbus ── センサ等
                                    │                            │
                                    └──── IEC 61850 (MMS) ───── IED/保護リレー
                                                                  │  GOOSE/SV (L2 マルチキャスト)
                                                                  └──── 変電所内の他IED・遮断器
```

| プロトコル | 主用途 | 決定性 | 認証(原仕様) | 備考 |
|---|---|---|---|---|
| **Modbus / Modbus-TCP** | PLC・フィールド機器 | 中 | なし | 単純・普及。レジスタ読み書きが基本 |
| **DNP3** | SCADA ↔ RTU（北米・配電） | 中 | なし(後付け SAv5) | イベント駆動・時刻同期に対応 |
| **IEC 60870-5-104 (T104)** | SCADA ↔ 制御所（欧州・送電） | 中 | なし(後付け IEC 62351) | TCP/IP 化された遠制プロトコル |
| **IEC 61850 MMS** | 変電所内・監視制御 | 中 | 弱(IEC 62351 で補完) | オブジェクト指向モデル、自己記述的 |
| **IEC 61850 GOOSE** | 保護トリップ・インタロック | 高(≤4 ms) | 弱 | L2 マルチキャスト、極低遅延 |
| **IEC 61850 Sampled Values (SV)** | 計器用変成器のデジタル化 | 高 | 弱 | 電圧・電流の生波形をストリーム |

## 技術詳細 (Technical Details)

### IEC 61850 の決定性とトレードオフ

IEC 61850 は変電所自動化を標準化し、機器を**論理ノード (Logical Node)** という自己記述的オブジェクトでモデル化する。通信は用途で3系統に分かれる。

- **MMS (Manufacturing Message Specification)**: クライアント/サーバ型。HMI からの監視・設定・制御に使う。TCP 上で動き、遅延要求は緩い。
- **GOOSE (Generic Object Oriented Substation Event)**: 保護トリップやインタロックを運ぶ。レイヤ2でマルチキャストし、**4 ms 以下**の伝送を要求。重要イベント時は同一メッセージを短間隔で再送し、到達性を高める（TCP の再送を待てないため）。
- **Sampled Values (SV)**: 計器用変成器 (CT/VT) の出力をデジタル波形としてストリーム配信。保護リレーが配線でなくネットワーク経由で電流・電圧を取得できる。

```
  従来 (hardwired)                     IEC 61850 (process bus)
  CT/VT ── 銅線 ── 保護リレー          CT/VT ── マージングユニット ─SV→ プロセスバス ── 保護リレー
  リレー ── 銅線 ── 遮断器             保護リレー ─GOOSE→ プロセスバス ── 遮断器
   (点対点配線、物理的に隔離)           (イーサネットで集約、配線激減・柔軟だが攻撃面拡大)
```

トレードオフ: プロセスバス化で配線が激減し柔軟性が上がる一方、保護という致命的機能がイーサネット上のマルチキャストに乗る。**決定性のために L2 ブロードキャストかつ軽量化した設計が、認証の薄さと攻撃面の拡大という代償を伴う**。これは CAN が「優先度調停と堅牢性のために無認証ブロードキャストにした」のと同型のトレードオフである（[`../automotive/in-vehicle-networks.md`](../automotive/in-vehicle-networks.md) 参照）。

### リアルタイムと決定性の要求

- **保護 (protection)**: 故障電流の検出からトリップまで数ミリ秒〜数十ミリ秒。遅れれば機器が焼損するため、最も厳しい決定性が要る。ここに汎用 OS の不確定な遅延は許されない。
- **閉ループ制御 (DCS)**: 制御周期は数十〜数百ミリ秒。スキャン → 演算 → 出力を周期的に回す。
- **監視制御 (SCADA)**: ポーリング周期は秒オーダ。広域のテレメトリを集約する。
- この階層により、**速い決定的制御は下位（フィールド近く）で閉じ**、上位の不確定な通信に依存しない。下位が自律的に安全を保てる設計が原則である。

### 広域監視と時刻同期 — PMU / シンクロフェーザ (WAMS)

SCADA のポーリング（秒オーダ・タイムスタンプは粗い）では、系統の**動的な現象**——地域間振動、外乱直後の位相角の開き——は見えない。これを可視化するのが **PMU (Phasor Measurement Unit, IEEE/IEC 60255-118-1)** である。

- 各変電所の PMU が電圧・電流の**フェーザ（振幅と位相角）** を毎秒 30〜60 サンプルで測定し、**GPS 同期のタイムスタンプ**を付けて中央 (PDC: Phasor Data Concentrator) へ送る。
- 地理的に何百 km 離れた測定点の**位相角を直接比較できる**のは、全点がマイクロ秒精度で同じ時刻を共有しているからである。これにより系統の「動揺」がリアルタイムで見える（**WAMS: Wide Area Monitoring System**）。2025 年のイベリア半島大停電の調査で 0.2 Hz の地域間振動を跡付けたのはこの計測層である（[`generation-grid-fundamentals.md`](generation-grid-fundamentals.md) の事例参照）。

ここで生まれる新しい依存が**時刻同期そのもの**である。

- 変電所の時刻源は GNSS (GPS) 受信機が主で、局内配布は IRIG-B や **PTP (IEEE 1588, power profile)** による。
- **GNSS は微弱な無認証電波**であり、妨害 (jamming) や偽装 (spoofing) で PMU のタイムスタンプ＝位相角測定が汚染されうる。位相角の誤りは監視・保護・状態推定への誤入力になる。
- 対策: 複数時刻源（GNSS 複数系・地上配信・原子時計ホールドオーバ）、受信機側の異常検知、そして「時刻を単一障害点にしない」設計。時刻同期は決定性通信 (GOOSE/SV) の前提でもあるため、**時刻が OT の攻撃面の一部**になったことは変電所設計の構造的変化である。

### 安全計装系 (SIS) と多重防護

```
  通常制御層 (BPCS: Basic Process Control System / DCS)
        │  通常運転を維持
        ▼  逸脱
  安全計装系 (SIS: Safety Instrumented System)  ← 独立・別系統
        │  危険状態を検出し安全側へ (例: 緊急停止)
        ▼  なお逸脱
  物理的防護 (機械式リリーフ弁、遮断器の機械的限界 等)
```

- SIS は通常制御 (BPCS) から**独立**して設計される。制御系が誤動作・侵害されても、SIS が物理的限界を超えさせない（IEC 61508 / IEC 61511 の安全度水準 SIL に基づく）。
- これは [`../automotive/functional-safety-iso26262.md`](../automotive/functional-safety-iso26262.md) の「制御が壊れても安全側へ倒す」思想と同根。ただし後述の TRITON のように **SIS 自体が攻撃対象**になった事例があり、安全層も保安の対象である。

## セキュリティ考察 (Security Considerations)

### 構造的な脆弱性

- **無認証プロトコル × IP 化。** Modbus/DNP3/T104 の原仕様は閉域網前提で送信者認証がない。遠隔運用のため IP 網・公衆網に接続された結果、なりすまし指令・リプレイの素地が露出した。補完規格（DNP3 Secure Authentication、IEC 62351）はあるが、既設への適用は進みにくい。
- **長寿命とパッチ困難。** 発電・送配電設備は20〜40年運用され、制御機器の OS・ファームは古く、可用性最優先ゆえ停止を伴うパッチが困難。脆弱性が長期間残存する。
- **フラットなネットワーク。** 歴史的に OT 網はセグメント化が弱く、一点の侵害が横展開しやすい。
- **物理直結。** サイバー侵害が即、物理プロセスの異常（過回転、過電圧、遮断器誤操作）に直結する。これが OT セキュリティを「情報漏洩」でなく「物理破壊・停電」の問題にしている。

### 実インシデントに見る攻撃の構造（公開情報・防御目的）

- **Stuxnet (2010)**: PLC の制御ロジックを改ざんし、遠心分離機を異常運転で物理的に損傷させた。同時にオペレータの HMI には正常値を表示し続け、**監視を欺いた**。教訓は「フィールドの実物理と監視表示が乖離しうる」こと、エアギャップも完全ではないこと。
- **ウクライナ電力網 (2015)**: 遠隔操作で変電所の遮断器を開放し広域停電を発生。正規の SCADA/HMI 操作経路が乗っ取られた。教訓はリモートアクセス経路と認証の重要性。
- **ウクライナ電力網 (2016, Industroyer/CrashOverride)**: 電力プロトコル（IEC 60870-5-101/104、IEC 61850、OPC）を直接話すマルウェアが用いられ、プロトコル知識の武器化を示した。
- **TRITON / TRISIS (2017)**: 安全計装系 (SIS) のコントローラを標的とした。**最後の砦である安全層も攻撃対象**になることを示した重大な転換点。

### 防御原則

- **多層防御とゾーン分割 (IEC 62443)。** Purdue モデルに沿って Zone と Conduit を定義し、IT/OT 境界に DMZ・ファイアウォール、必要に応じ**一方向ゲートウェイ (data diode)** を置く。横展開を物理的・論理的に断つ。
- **最小権限とアクセス制御。** リモートアクセスは多要素認証・踏み台経由・時限化。Stuxnet/ウクライナの教訓はここに集約される。
- **安全と保安の独立。** SIS は制御系から独立させ、SIS への通信・更新経路を厳格に分離・監視する（TRITON の教訓）。
- **監視の信頼性。** Stuxnet の「偽装表示」を踏まえ、フィールドの実測と監視値の整合監視、異常検知 (OT 向け IDS) を導入する。
- **可用性を壊さない更新運用。** パッチは停止計画と整合させ、補償的統制（仮想パッチ、セグメント化）で猶予を作る。IT 流の即時パッチを強制しない。

> 本節は公開済みインシデントの構造と防御原則のみを扱い、攻撃の再現手順・悪用可能な詳細は記述しない。

## 結論 (Conclusions)

- 発電・系統の制御システムは、物理プロセスを止めないことを最優先する OT であり、**可用性・完全性・決定性が機密性に優先する**。IT の常識（即パッチ、暗号前提、再起動）がそのまま通用しない。
- 制御は時間スケールで階層化される。μs の保護、ミリ秒の閉ループ (PLC/DCS)、秒の監視 (SCADA)、分の最適化 (EMS) が分業し、**速く致命的な制御ほど下位で自律的に閉じる**設計が原則である。
- 主要プロトコル（Modbus/DNP3/T104）は閉域網前提で無認証であり、IEC 61850 は決定性のために L2 マルチキャストで保護機能を運ぶ。いずれも「性能・決定性のための割り切り」が認証の薄さという代償を伴う点で、CAN と同型のトレードオフを示す。
- 物理直結ゆえ、サイバー侵害は物理破壊・広域停電に直結する。Stuxnet・ウクライナ・TRITON は、監視の欺瞞・遠隔操作・安全層への攻撃という異なる構造を実証した。
- 防御は IEC 62443 のゾーン分割・多層防御、最小権限のリモートアクセス、安全と保安の独立、監視の信頼性確保、可用性を壊さない更新運用に集約される。発電・系統制御は社会インフラ規模の組込み制御であり、安全 (safety) と保安 (security) を独立した層として両立させることが本質的な要件である。

## 参考文献 (References)

- Stouffer, K., et al. (2023). *Guide to Operational Technology (OT) Security* (NIST Special Publication 800-82 Rev. 3). National Institute of Standards and Technology. DOI: 10.6028/NIST.SP.800-82r3
- International Electrotechnical Commission. (2018–). *IEC 62443 — Security for industrial automation and control systems* (series). IEC.
- International Electrotechnical Commission. *IEC 61850 — Communication networks and systems for power utility automation* (series). IEC.
- International Electrotechnical Commission. *IEC 62351 — Power systems management and associated information exchange — Data and communications security* (series). IEC.
- Falliere, N., Murchu, L. O., & Chien, E. (2011). *W32.Stuxnet Dossier* (Version 1.4). Symantec Security Response.
- [E-ISAC & SANS (2016). Analysis of the Cyber Attack on the Ukrainian Power Grid (TLP: White)](https://www.nerc.com/pa/CI/ESISAC/Documents/E-ISAC_SANS_Ukraine_DUC_18Mar2016.pdf) — Accessed: 2026-06-29
- [Dragos (2017). CRASHOVERRIDE: Analysis of the Threat to Electric Grid Operations](https://www.dragos.com/resources/) — Accessed: 2026-06-29
- [MITRE ATT&CK for ICS](https://attack.mitre.org/matrices/ics/) — Accessed: 2026-06-29
- IEEE/IEC 60255-118-1:2018, *Measuring relays and protection equipment — Part 118-1: Synchrophasor for power systems — Measurements*（旧 IEEE C37.118.1）. — PMU 測定の規範。
- IEEE 1588 (PTP) Power Profile (IEEE C37.238). — 変電所内時刻同期の規範。
