# AUTOSAR と車載ソフトウェア基盤 (AUTOSAR & Vehicle Software Platforms)

## 概要 (Overview)

自動車は「100 個の ECU で分散リアルタイムシステムを 20 年間・数千万台規模で運用する」という、組み込みソフトウェア最大級の産業実践である。**AUTOSAR (AUTomotive Open System ARchitecture)** は、この規模を可能にするために業界が合意した標準アーキテクチャであり、`rtos-architectures.md` で見た設計空間——静的構成の小型 RTOS からPOSIX ベースの動的プラットフォームまで——が、**Classic Platform** と **Adaptive Platform** という2つの標準として制度化されたものと読める。

本ドキュメントは、OSEK/VDX から AUTOSAR Classic への系譜（静的・スケジュール表ベースの世界）、サービス指向へ舵を切った Adaptive Platform（POSIX・SOME/IP の世界）、両者を貫く安全機構（干渉からの自由・E2E 保護）、そして標準の外側で進む SDV (software-defined vehicle) と内製 Vehicle OS の潮流までを扱う。ハードウェア側の対応物は [`../../hardware/automotive/`](../../hardware/automotive/)（E/E アーキテクチャ・車載ネットワーク・機能安全）であり、本ドキュメントはそのソフトウェア側の合流点——hardware/software 両スタックを束ねる橋——である。

CLAUDE.md の方針に従い分析深度は Level 3（完全）を既定とし、各主張には規格一次資料（AUTOSAR 仕様・ISO 規格）を典拠として付す。

## 方法論 (Methodology)

- AUTOSAR を API のカタログではなく、**産業構造への応答**として読む: OEM とサプライヤの分業（誰が何を作り、どう統合するか）が先にあり、アーキテクチャはそれを可能にする形に決まっている。
- Classic と Adaptive を「新旧」ではなく**静的保証と動的柔軟性のトレードオフの二点**として対比し、車両内での共存（ドメイン別の使い分け）を前提に扱う。
- 安全は `../../hardware/automotive/functional-safety-iso26262.md` の ISO 26262 を前提とし、本ドキュメントでは**ソフトウェアアーキテクチャに現れる実装機構**（パーティショニング・E2E）に絞る。
- SDV の動向は時事性が高いため、確立した構造（ゾーン化・HPC 集約・OTA）と流動的な部分（各社の内製 OS の成否）を区別する。

## 発見事項 (Findings)

1. **AUTOSAR の第一の目的は技術ではなく「分業の接続仕様」である。** OEM が車両機能を定義し、Tier-1 がコンポーネントを実装し、別のベンダがベーシックソフトウェアを供給し、それらが単一 ECU に統合される——この多社分業は、アプリケーションを実行基盤から切り離す標準層 (**RTE**, runtime environment) と、交換可能な構成記述 (ARXML) があって初めて成立する。アーキテクチャ図のすべての層は、契約の境界線として読むのが正しい。
2. **Classic Platform は「実行時に決めない」ことで保証を買う設計の極致である。** タスク・優先度・スケジュール表・通信信号・メモリ割当のすべてをビルド時に静的構成し、動的なタスク生成もヒープも持たない。OSEK OS（固定優先度 + 優先度シーリング内蔵）を核に、`realtime-fundamentals.md` の解析（応答時間・ブロッキング上界）が**構成ファイルから機械的に実施できる**世界を作っている[^osek][^autosar-cp]。
3. **Adaptive Platform は「ECU の中にサーバを持ち込む」ための第二標準である。** ADAS・自動運転の要求（高性能 SoC、動的なアプリ配備、OTA、機械学習ワークロード）は静的世界と両立しないため、AUTOSAR は Classic を置き換えず、POSIX (PSE51) ベース・C++・**サービス指向通信 (SOME/IP)** の Adaptive Platform を並立させた[^autosar-ap]。実行管理・状態管理がアプリのライフサイクルを動的に扱う点で、`rtos-architectures.md` の「Linux/QNX 側の島」の車載標準化といえる。
4. **通信は「信号」から「サービス」へ移行しつつ、車両内で二層共存する。** CAN 的な信号ベース通信（静的に割り付けられたビットフィールドの周期放送——`../../hardware/automotive/in-vehicle-networks.md`）と、SOME/IP によるサービス指向通信（発見・購読・RPC——`../networking/` の世界観）が、それぞれ制御系ドメインと知覚・計画系ドメインを担う。`../../hardware/automotive/automotive-ethernet-tsn.md` の車載 Ethernet がこの二層目の物理基盤である。
5. **「干渉からの自由 (freedom from interference)」が混合 ASIL 統合の鍵概念である。** ISO 26262 は、異なる安全度 (ASIL) のソフトウェアを同居させる場合、低 ASIL 側が高 ASIL 側を**空間（メモリ）・時間（実行時間）・通信**の3面で妨害できないことの論証を要求する。実装機構は OS アプリケーション単位の MPU パーティショニング、実行時間バジェット監視、そして **E2E (end-to-end) 保護**——CRC・シーケンスカウンタ・タイムアウトをアプリ層で付け、経路上のどの層の故障も受信側で検出する——である[^autosar-cp]。`rtos-architectures.md` の ARINC 653 と同じ問題への、車載流の（より漸進的な）解といえる。
6. **SDV の実体は「ECU の地理的統合」と「ソフトウェア更新の恒常化」である。** 機能別に散らばった 100 ECU を、ゾーン ECU（配線集約）+ 少数の HPC (high-performance computer) に集約し（`../../hardware/automotive/ee-architecture.md`）、HPC 上ではハイパーバイザ（`../virtualization/`）とコンテナで Classic 的な島と Adaptive/Linux 的な島を同居させ、OTA で継続更新する——Tesla が先行して示し（`../../hardware/automotive/ev/tesla-software-manufacturing.md`）、業界全体が追随している構図である。
7. **標準と内製のせめぎ合いが現在の主戦場である。** AUTOSAR は分業の接続仕様として成功したが、その分業構造自体が「ソフトウェアを競争力の中心に置く」戦略と衝突する。Tesla の完全内製、各 OEM の Vehicle OS 構想（成否は流動的）、車載 Linux・QNX・ハイパーバイザベンダの寡占化など、**「標準で作る部分」と「内製で差別化する部分」の線引き**が各社の戦略変数になっている。この対立軸は `../../hardware/robotics/humanoid-robots-analysis.md` の垂直統合の議論と同型である。

## アーキテクチャ (Architecture)

### Classic Platform の層構造

```mermaid
flowchart TB
    subgraph ecu["ECU (Classic Platform)"]
        subgraph app["アプリケーション層"]
            SWC1[SWC: ソフトウェア<br/>コンポーネント]
            SWC2[SWC]
        end
        RTE[RTE (Runtime Environment)<br/>ポート接続をコードとして生成]
        subgraph bsw["ベーシックソフトウェア (BSW)"]
            OS[OSEK 系 OS<br/>固定優先度 + シーリング]
            COM[通信スタック<br/>CAN/LIN/FlexRay/Ethernet]
            MEM[メモリ・診断・<br/>ウォッチドッグサービス]
            MCAL[MCAL: マイコン抽象層<br/>（ベンダ供給ドライバ）]
        end
    end
    SWC1 & SWC2 <--> RTE
    RTE <--> OS & COM & MEM
    COM & MEM --> MCAL
    MCAL --> HW[マイコン<br/>hardware/hw-sw-interface/]
```

- **SWC (software component)** は再配置可能な単位: ポート（送受信インターフェース）だけを宣言し、どの ECU に載るか・誰と通信するかは統合時の構成で決まる。RTE はその構成から**通信コードを生成**する層であり、同一 ECU 内なら変数コピー、ECU 間なら通信スタック呼び出しに同じポートアクセスが展開される。
- この「配置の遅延束縛 + 生成による静的化」が、分業（発見事項 1）と静的保証（発見事項 2）を同時に満たす中核機構である。

### Classic と Adaptive の対比

| 観点 | Classic Platform | Adaptive Platform |
|---|---|---|
| OS | OSEK 系（静的タスク・優先度シーリング） | POSIX PSE51（QNX・Linux 等） |
| 言語 / API | C・生成された RTE API | C++・`ara::com` ほかサービス API |
| 構成 | 全て静的（ビルド時 ARXML） | 実行時のアプリ配備・起動を許す |
| 通信 | 信号ベース（CAN 等の周期送信） | サービス指向 (SOME/IP・DDS オプション) |
| メモリ | ヒープなし・静的割当 | 動的確保あり |
| 更新 | ECU 全体の再フラッシュ | アプリ単位の OTA 配備 |
| 主用途 | パワートレイン・シャシー・ボディ制御 | ADAS・自動運転・コックピット・接続 |
| 保証の性格 | 構成から機械的に解析可能 | プロセス・分離機構で論証 |

- 対比は `rtos-architectures.md` の「小さく固い島 / 大きく柔らかい島」の車載版であり、1台の車の中で両者がゾーン/ドメイン別に共存する。

## 技術詳細 (Technical Details)

### 1. OSEK OS — 静的世界のカーネル (The Static Kernel)

- OSEK/VDX（1990年代の欧州標準、ISO 17356 として国際規格化）が定めるのは、静的に宣言されたタスク（基本/拡張）、固定優先度スケジューリング、そして**優先度シーリングプロトコル内蔵のリソース**である[^osek]。`realtime-fundamentals.md` で「強い保証だが静的情報が要る」とした優先度シーリングが、ここでは**構成ファイルが静的情報そのものである**ため自然に成立する。
- **スケジュール表 (schedule table)** と時間監視（OSEKtime / AUTOSAR OS のタイミング保護）により、タスクの起動時刻・実行時間バジェットを表として固定でき、時間の干渉からの自由（発見事項 5）の実装手段になる。
- カウンタ・アラーム・フック（エラー処理）まで含め、**「実行時に構造が変わらない」ことが最大の機能**である——解析可能性・認証・再現性のすべてがそこから導かれる。

### 2. 通信スタックと E2E 保護 (Communication & End-to-End Protection)

- 信号ベース通信では、送信側 SWC の書き込みが COM 層で PDU (protocol data unit) にパックされ、周期送信される。受信側ではアンパックされ RTE 経由で届く。この経路は多層のソフトウェアを通るため、**個々の層を高 ASIL に上げる代わりに、経路全体を末端で検証する**のが E2E 保護である:

```
E2E プロファイルの典型構成（送信側で付与 → 受信側で検査）:
  CRC          — 経路上のビット化け・変換バグの検出
  シーケンス番号 — 欠落・重複・順序入れ替わりの検出
  データ ID     — 誤配線（別信号の混入）の検出
  タイムアウト   — 鮮度（古い値の使い回し）の検出
```

- 検出後の挙動（代替値・縮退・安全状態遷移）は受信側アプリケーションの安全設計に属する。「**通信路を信頼せず、受信側で品質を判定する**」という構図は、`../networking/tls-internals.md` のエンドツーエンド原則の機能安全版である。
- SOME/IP はサービス発見 (SOME/IP-SD)・メソッド呼び出し・イベント購読を提供する車載特化のサービスプロトコルで、`../../hardware/automotive/automotive-ethernet-tsn.md` の上に載る。Adaptive では DDS（`../../hardware/robotics/robot-software-stack.md` の ROS 2 と同じ規格）もバインディングとして選択でき、ロボットと車載のミドルウェアが同じ規格に合流しつつある点は示唆的である。

### 3. Adaptive Platform の実行管理 (Adaptive: Execution & State Management)

- **実行管理 (Execution Management)** がアプリケーション（Adaptive Application）のライフサイクル——起動依存関係・状態遷移（起動/更新/診断モード）——を管理し、**プラットフォーム健全性管理 (Platform Health Management)** が監視（生存・締切・論理シーケンス）を担う。静的なスケジュール表の代わりに、**監視 + 縮退**で時間的な安全性を論証する構造である[^autosar-ap]。
- `ara::com` はサービスのプロキシ/スケルトンを生成する通信抽象で、トランスポート（SOME/IP・DDS・プロセス内）を差し替え可能にする——RTE が静的世界でやったこと（配置の遅延束縛）の動的世界版といえる。
- OS は規定しない（POSIX PSE51 準拠を要求するのみ）。実務では安全系に QNX、情報系に Linux という `rtos-architectures.md` の分業がそのまま現れ、単一 SoC 上ではハイパーバイザで共存させる。

### 4. SDV — ゾーン・HPC・OTA (Software-Defined Vehicle)

- **E/E アーキテクチャの再編**（`../../hardware/automotive/ee-architecture.md`）: 機能別 ECU → ドメイン統合 → ゾーン + 中央 HPC。ソフトウェア側の帰結は、(1) Classic の島が HPC 内の仮想 ECU・ゾーン ECU に移住する、(2) 機能の実体が特定 ECU から「車両計算機上のサービス」に変わる、の2点である。
- **OTA 更新**は SDV の経済的核心（機能の後売り・リコールのソフト化）だが、技術的には `rtos-architectures.md` で述べた**認証の凍結と更新の恒常化の衝突**を最前線で引き受ける: 更新単位の分離（干渉からの自由が更新境界にもなる）、A/B パーティションによる原子的更新とロールバック、そして ISO 26262（安全）と ISO/SAE 21434（サイバーセキュリティ—`../../hardware/automotive/cybersecurity-iso21434.md`）・UN-R156（ソフトウェア更新管理システムの認可）への同時適合が要る。
- Tesla の内製スタック（`../../hardware/automotive/ev/tesla-software-manufacturing.md`）は「AUTOSAR 的分業を持たない垂直統合ならどこまで速く動けるか」の対照実験であり、標準陣営の SDV 構想はその速度への応答として読める。

## セキュリティ考察 (Security Considerations)

- **信号ベース通信は認証を持たない。** CAN の設計には送信者認証がなく（`../../hardware/automotive/in-vehicle-networks.md`・`cybersecurity-iso21434.md`）、AUTOSAR は **SecOC (secure onboard communication)**——PDU への MAC (message authentication code) 付与と鮮度カウンタ——で後付けの認証を標準化した。E2E 保護が「偶発故障」を検出するのに対し、SecOC は「悪意ある注入」を検出する——**CRC は攻撃者に対して無力**であり、両者は代替関係にない点が設計上の要諦である。
- **境界の多層化**: 外部接続（テレマティクス・コックピット）と制御ドメインの間にゲートウェイ・ファイアウォールを置き、ドメイン分離（ハイパーバイザ・ネットワークセグメント）で侵害の横展開を抑える。構図は `../../hardware/power-generation/plant-control-scada-security.md` の OT 分離、`../../hardware/robotics/control-systems.md` の安全独立経路と同型である。
- **OTA は最大の特権経路**: 車両全体のコードを書き換えられる経路であるため、署名検証の信頼根（HSM: hardware security module、`../../hardware/hardware-security/side-channels-tee-secure-boot.md` の secure boot の車載版）、段階的ロールアウト、ロールバック保護（ダウングレード攻撃の防止）が必須要件になる。UN-R155/156 は、これらを型式認可の条件として制度化した。
- **サプライチェーン**: 多社分業アーキテクチャは、そのままソフトウェアサプライチェーンの攻撃面でもある。BSW・MCAL・ツールチェーンの完全性、SBOM、ツール適格性 (tool qualification) は、`../security/`（ロードマップのサプライチェーン項目）の車載での実体化である。

## 結論 (Conclusions)

AUTOSAR は、`realtime-fundamentals.md` の理論と `rtos-architectures.md` の設計空間が、**産業の分業構造と安全認証の要求に合わせて制度化されたもの**である。Classic Platform は「実行時に決めない」ことで解析可能性と分業を同時に成立させ、Adaptive Platform は ADAS 時代の動的要求を POSIX + サービス指向で受け止め、両者は SDV の HPC 上で島として共存する。そして E2E 保護・SecOC・干渉からの自由といった機構は、いずれも「**下の層を信頼せず、境界で検証する**」という一つの原理の現れであり、これは本リポジトリが networking・security・分散システムで繰り返し見てきた原理の車載版にほかならない。

これで `software/embedded-rtos/` の3本——理論・カーネル・産業実装——が揃い、`hardware/robotics/`・`hardware/automotive/`・`hardware/aerospace/` の各所から参照されてきた「リアルタイム実行基盤」の答えが software 側に整備された。残るテーマ候補（TSN ホスト実装、スケジューラビリティ解析の実務、省電力との両立）は README のロードマップに置く。

---

## 参考文献 (References)

[^osek]: ISO 17356-3:2005. *Road vehicles — Open interface for embedded automotive applications — Part 3: OSEK/VDX Operating System (OS)*. — 静的タスクモデル・優先度シーリングの規格一次資料。

[^autosar-cp]: AUTOSAR. *Classic Platform Release Documentation*（Specification of Operating System, Specification of RTE, E2E Protocol Specification ほか）. https://www.autosar.org/standards/classic-platform — Accessed: 2026-07-12. — 層構造・RTE・E2E プロファイルの規格一次資料。

[^autosar-ap]: AUTOSAR. *Adaptive Platform Release Documentation*（Explanation of Adaptive Platform Design, Specification of Execution Management, Specification of Communication Management ほか）. https://www.autosar.org/standards/adaptive-platform — Accessed: 2026-07-12. — POSIX 基盤・ara::com・実行管理の規格一次資料。

その他の標準的典拠:

- ISO 26262:2018. *Road vehicles — Functional safety*（特に Part 6: ソフトウェア開発、Part 9: ASIL 指向・安全指向解析）. — 干渉からの自由の要求元。
- ISO/SAE 21434:2021. *Road vehicles — Cybersecurity engineering*. — SecOC・OTA セキュリティの上位枠組み。
- UN Regulation No. 155 / 156 (2021). *Cyber security and cyber security management system / Software update and software update management system*. — 型式認可としての制度化。
- Staron, M. (2021). *Automotive Software Architectures: An Introduction* (2nd ed.). Springer. — AUTOSAR の層構造と開発プロセスの教科書的整理。
