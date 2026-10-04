# 車載ネットワークと ECU (In-Vehicle Networks & ECU)

## 概要 (Overview)

現代の自動車は、数十〜百を超える **ECU (Electronic Control Unit, 電子制御ユニット)** が協調して動く分散システムである。エンジン制御、ブレーキ、エアバッグ、ドアロック、計器表示——それぞれを担う組込みコンピュータが、車載ネットワーク (in-vehicle network) で結ばれて情報を交換する。

本ドキュメントは、車載ネットワークの中核である **CAN (Controller Area Network)** を軸に、LIN・FlexRay・Automotive Ethernet を含むプロトコル群と ECU の役割を整理する。組込みソフトウェアと物理（電気・タイミング・安全）の接点として、ソフトエンジニアの視点から「なぜこの設計か」を明らかにする。

## 方法論 (Methodology)

- 車内の通信要求を「コスト・速度・リアルタイム性・安全性」の4軸で捉え、各プロトコルがどの象限を埋めるかを対応づける。
- 代表として CAN のフレーム構造・調停 (arbitration)・誤り検出を分解し、その設計判断（マルチマスタ、優先度、堅牢性）の根拠を示す。
- ネットワーク全体（ドメイン/ゾーンアーキテクチャ、ゲートウェイ）と、セキュリティ（CAN には認証がない問題）まで通す。
- 電気的特性・タイミングはプロトコル仕様の代表値に基づく（CAN: ISO 11898）。

## 発見事項 (Findings)

1. **単一の万能バスは存在しない。** 要求が多様（数円のスイッチ〜カメラ映像）なため、コストと性能の異なる複数ネットワークを使い分ける。
2. **CAN は「優先度付きマルチマスタ・ブロードキャスト」。** アドレス指定ではなく**メッセージID＝優先度**で、衝突を破壊せずに調停する独特の設計。
3. **CAN は堅牢だが低速かつ無認証。** 差動信号 (differential) と多重誤り検出で電気ノイズに強い一方、帯域は最大1Mbps（CAN FD で改善）、そして**送信者認証がない**——これが車載セキュリティの根本課題。
4. **アーキテクチャは「ドメイン型」から「ゾーン型」へ移行中。** ECU 統合と Automotive Ethernet 化が進み、車載ソフトは大規模分散システムに近づいている。
5. **物理とソフトが密結合。** ビットタイミング、終端抵抗、バス長が通信成立に直結し、ソフトだけでは閉じない。

## アーキテクチャ (Architecture)

### プロトコルの使い分け

| プロトコル | 速度 (代表) | 特性 | 主な用途 |
|---|---|---|---|
| **LIN** | ~20 kbps | 単線・低コスト・マスタ/スレーブ | ドアミラー、窓、シート等の低速ボディ系 |
| **CAN** | ~1 Mbps | 差動2線・堅牢・優先度調停 | パワートレイン、シャシ、ボディ全般 |
| **CAN FD** | ~2–8 Mbps | CAN 互換でペイロード/速度拡張 | CAN の帯域不足の緩和 |
| **FlexRay** | ~10 Mbps | 時分割・決定論的・冗長 | X-by-wire 等の高安全・高同期用途 |
| **Automotive Ethernet** | 100M–1G+ bps | 点対点・高帯域 | カメラ/レーダ、ADAS、車載バックボーン |
| **MOST**（旧） | — | 光・高帯域 | 旧世代インフォテインメント |

要点: **低コスト＝LIN、汎用堅牢＝CAN、決定論＝FlexRay、高帯域＝Ethernet**。1台の車に複数が同居し、ゲートウェイ ECU が橋渡しする。

### ECU の構成

典型的な ECU は組込みマイコン (microcontroller) を中核に持つ。

```
        ┌─────────────────────── ECU ───────────────────────┐
        │  ┌──────────┐   ┌──────────┐   ┌───────────────┐   │
  入力 ──▶│  ADC/GPIO │──▶│   MCU     │──▶│ CAN/LIN コント │──┼──▶ バスへ
 (センサ)  │ (信号取込) │   │ (制御演算) │   │ ローラ+トランシ │   │
        │  └──────────┘   │ + Flash/RAM│   │ ーバ           │   │
  出力 ◀──┤  ドライバ回路 ◀──┤  + 割込み   │   └───────────────┘   │
 (アクチュ │  (PWM等)    │   └──────────┘                        │
  エータ) └────────────────────────────────────────────────────┘
```

ここで前提となるのが `../../hardware/hw-sw-interface/mmio-dma-interrupts.md` の3機構そのもの——MMIO でペリフェラル（CAN コントローラ、ADC、PWM）を制御し、受信完了を割り込みで受け、まとめ転送に DMA を使う。**車載 ECU は HW・SW境界の実応用例**である。

### CAN フレームの構造（標準フォーマット）

```
 ┌─────┬──────────────┬─────┬─────┬──────────┬─────┬─────┬─────┐
 │ SOF │ Identifier   │ RTR │ ... │ DLC      │Data │ CRC │ ACK │ EOF
 │ 1b  │ 11b (優先度)  │ 1b  │ ctrl│ 4b(0–8B) │0–8B │15b  │     │
 └─────┴──────────────┴─────┴─────┴──────────┴─────┴─────┴─────┘
```

- **アドレスがない**: フレームは宛先ではなく**内容を表す ID** を持ち、全ノードへブロードキャストされる。各 ECU は受信フィルタで必要な ID だけ拾う。
- **ID = 優先度**: ID の数値が小さいほど高優先度。
- **拡張フォーマット (CAN 2.0B)**: 29bit ID 版もあり、商用車の J1939 はこの 29bit に PGN（パラメータグループ番号）と送信元アドレスを埋め込む。

### 調停 (Arbitration) — CAN の核心

複数ノードが同時送信したときの衝突を、**非破壊ビット単位調停 (non-destructive bitwise arbitration)** で解決する。

- バスは**ワイヤード AND**的に働き、`0`（ドミナント, dominant）が `1`（リセッシブ, recessive）に勝つ。
- 各ノードは送信しながら自分の出力とバスを比較し、`1` を出したのに `0` が観測されたら「より高優先度のノードがいる」と判断し**即座に降りる**。
- 結果、最も小さい ID（最高優先度）のフレームが**衝突で壊れずに**そのまま送信完了する。Ethernet の衝突→再送 (CSMA/CD) と違い、**衝突しても1フレームも無駄にしない**のが特徴。リアルタイム性の根拠。

### 誤り検出と堅牢性

- **差動信号 (differential signaling)**: CAN_H / CAN_L の電位差で 0/1 を表現。同相ノイズに強く、車載の過酷な電磁環境に適する。
- **多重の誤り検出**: CRC、ビットスタッフィング、フレームチェック、ACK、ビットモニタリング。検出時はエラーフレームで全ノードに通知し再送。
- **エラー閉じ込め (fault confinement)**: 各ノードが送信エラーカウンタ (TEC) と受信エラーカウンタ (REC) を持ち、故障ノードを段階的に隔離する。

  | 状態 | 条件 | 振る舞い |
  |---|---|---|
  | **Error-Active** | TEC, REC < 128 | 通常動作。エラー検出時はアクティブエラーフラグ（ドミナント）で全ノードに通知 |
  | **Error-Passive** | TEC または REC ≥ 128 | エラーフラグはリセッシブ（他を邪魔しない）。連続送信に待機時間が課される |
  | **Bus-Off** | TEC > 255 | 送信禁止。バスから切り離される |

  Bus-Off からの復帰には「11連続リセッシブビットを128回」観測する必要がある（≒バスが十分アイドルであることの確認）。**自己申告制の隔離**であり、故障したノード自身が正しくカウントすることを前提とする点は覚えておく価値がある（悪意あるノードには効かない——セキュリティ考察参照）。

## 技術詳細 (Technical Details)

### CAN FD と CAN XL — 帯域不足への応答

クラシック CAN の 1 Mbps / 8 バイトは、ソフト更新 (OTA) や高機能 ECU 間通信には足りない。互換性を保ちながら拡張したのが CAN FD、Ethernet との間を埋めるのが CAN XL である。

| | クラシック CAN | **CAN FD** (ISO 11898-1) | **CAN XL** (CiA 610) |
|---|---|---|---|
| ペイロード | 0–8 B | 0–64 B | 1–2048 B |
| データ部速度 | ≤ 1 Mbps | 2–8 Mbps (典型) | ~10–20 Mbps (SIC XL) |
| CRC | 15 bit | 17/21 bit（長ペイロード対応） | 32 bit |
| 特記 | — | リモートフレーム廃止 | Ethernet フレームのトンネリング可 |

- **CAN FD の核心は BRS (Bit Rate Switch)**: 調停フェーズは全ノード参加のため従来速度のまま、**調停が決着した後のデータフェーズだけ高速化**する。調停の仕組み（伝搬遅延がビット時間に収まる必要）を壊さずに帯域を稼ぐ、制約からの逆算設計。
- 移行の現実: FD 非対応ノードは FD フレームをエラーとして扱うため、**バス単位での移行**が必要。ゾーン化 (`automotive-ethernet-tsn.md`) と並行して進む。

### ネットワークアーキテクチャの進化

- **ドメイン型 (domain architecture)**: 機能ドメイン（パワートレイン/シャシ/ボディ/インフォテインメント）ごとにバスとドメインコントローラを置き、ゲートウェイで相互接続。
- **ゾーン型 (zonal architecture)**: 車内の物理的位置（前/後/左/右ゾーン）でゾーン ECU に集約し、中央の高性能コンピュータ (HPC) と Automotive Ethernet バックボーンで結ぶ。配線重量の削減と、ソフトウェアの中央集約（**SDV: Software-Defined Vehicle**）を狙う。
- ゾーン化により車載は「多数の小 ECU」から「少数の高性能計算機＋分散 I/O」へ向かい、`../../hardware/cpu/` の知見（メモリ階層・並行性・仮想化）がより重要になる。

### 上位プロトコルとソフト基盤

- **CAN は L1/L2 のみ**を定義する。実用では上位プロトコルが乗る: 車載診断の **UDS (ISO 14229)** / **OBD-II**、組込み制御の **CANopen**、商用車の **J1939**。
- **ISO-TP (ISO 15765-2)**: 8バイト超のデータ（診断応答、ファーム転送）を CAN に載せるためのセグメント化・フロー制御層。First Frame → Flow Control → Consecutive Frames の順で、受信側が送信ペース (STmin, block size) を指定する。
- **UDS の代表サービス**（診断・書き換えの実体。攻撃面としても重要）:

  | SID | サービス | 用途 |
  |---|---|---|
  | `0x10` | DiagnosticSessionControl | 診断セッション切替（拡張・プログラミング） |
  | `0x27` | SecurityAccess | シード・キー方式のアンロック（実装が弱いと突破される） |
  | `0x22` / `0x2E` | ReadData / WriteDataByIdentifier | パラメータ読み書き |
  | `0x19` / `0x14` | ReadDTC / ClearDTC | 故障コードの読み出し・消去 |
  | `0x31` | RoutineControl | 自己診断・キャリブレーション等の起動 |
  | `0x34`–`0x37` | RequestDownload〜TransferExit | ECU リプログラミング（ファーム書き換え） |

  近年は Ethernet 上の **DoIP (ISO 13400)** でも同じ UDS を運ぶ（`automotive-ethernet-tsn.md`）。
- **AUTOSAR**: ECU ソフトの標準アーキテクチャ。通信スタック・OS・基本ソフト (BSW) を抽象化し、アプリを SWC として分離。Adaptive AUTOSAR は Ethernet/SOME-IP と高性能 SoC を前提とする。

### 物理的制約（ソフトで閉じない部分）

- **ビットタイミング**: 1ビットは時間量子 (time quantum) 単位で SYNC_SEG / PROP_SEG / PHASE_SEG1 / PHASE_SEG2 に分割され、PHASE_SEG1 と 2 の境界が**サンプル点**（典型 75–87.5%）になる。全ノードがビットレートとこれらのパラメータを共有し、エッジ検出時に SJW (synchronization jump width) の範囲で再同期する。設定ミスは即通信不能。
- **終端抵抗 (termination)**: CAN バス両端に 120Ω を入れて反射を抑える。欠落・過多はエラーの原因。
- **バス長 vs 速度**: 調停が成立するには、信号がバス全体を往復して1ビット時間内に収まる必要がある。ゆえに高ビットレートほどバスを短くする。CiA の目安値を次に示す。

  | ビットレート | 最大バス長の目安 |
  |---|---|
  | 1 Mbps | ~40 m |
  | 500 kbps | ~100 m |
  | 250 kbps | ~250 m |
  | 125 kbps | ~500 m |

  乗用車の制御系で 500 kbps が長く定番だったのは、この物理制約と車体サイズの均衡点だからである。

## セキュリティ考察 (Security Considerations)

- **認証の欠如が最大の弱点**: CAN は設計上、送信者を認証しない。バスに物理アクセスできれば任意 ECU になりすまし、ブレーキやステアリングへ偽メッセージを注入しうる。Koscher らの実車解析 (2010) と Miller & Valasek の Jeep 遠隔操作 (2015) が、この弱点が物理アクセスなしでも（テレマティクス経由で）現実の攻撃になることを実証した（参考文献参照）。
- **エラー処理の悪用**: 上記の fault confinement は自己申告制のため、悪意あるノードが意図的にエラーを誘発して**正規 ECU を Bus-Off に追い込む** DoS（bus-off attack）が成立する。プロトコル層では防げず、IDS・物理層監視で検出する。
- **攻撃面の拡大**: インフォテインメント、テレマティクス (telematics)、Wi-Fi/Bluetooth、OBD-II ポートが外部入口となり、そこからゲートウェイ越しに制御系バスへ到達する経路がリスク。
- **対策**:
  - **ネットワーク分離・ゲートウェイ**: 制御系バスと外部接続系を分離し、ゲートウェイでフィルタ・レート制限。
  - **メッセージ認証**: **SecOC (Secure Onboard Communication, AUTOSAR)** による MAC 付与とフレッシュネス値でなりすまし・再送を防ぐ。
  - **侵入検知 (IDS)**: 周期・ID の異常を監視。
  - **セキュアブート / HSM**: ECU ファームの改竄防止、鍵管理を **HSM (Hardware Security Module)** で行う。
  - **標準**: **ISO/SAE 21434**（車両サイバーセキュリティ）、UNECE WP.29 規制が設計プロセスに義務を課す。
- **安全 (safety) と保安 (security) の交差**: 車載では誤動作が人命に直結する。**ISO 26262**（機能安全）と **ISO 21434**（セキュリティ）の両立が必須で、攻撃が安全機構を突破しない設計が求められる。

## 結論 (Conclusions)

- 車は多数の ECU による分散システムであり、要求の多様さから LIN/CAN/FlexRay/Ethernet を使い分ける。万能バスはない。
- CAN の本質は「メッセージID＝優先度による非破壊調停」で、衝突を無駄なく解決しリアルタイム性と堅牢性を両立する。代償は低帯域と無認証。
- アーキテクチャはドメイン型からゾーン型・SDV へ移行し、車載ソフトは大規模分散システム化している。CPU アーキテクチャや HW・SW境界の知見が直接効く領域になりつつある。
- 無認証という設計前提ゆえ、ネットワーク分離・SecOC・IDS・HSM と、安全(ISO 26262)・保安(ISO 21434)の両立がセキュリティの要となる。

### 次の接続先 (Related Topics)

- `automotive-ethernet-tsn.md` — CAN の外側：車載 Ethernet・TSN・FlexRay・LIN・SOME/IP（本書の対）。
- `cybersecurity-iso21434.md` — 無認証問題への対策（SecOC・HSM・分離）と ISO 21434。
- `functional-safety-iso26262.md` — 無認証・分離とゲートウェイが支える安全設計。
- `../../hardware/hw-sw-interface/mmio-dma-interrupts.md` — ECU 内部のデバイス制御（MMIO/DMA/割り込み）。
- `../../hardware/cpu/` — ゾーン型 HPC が要求する計算・並行性・仮想化の基礎。

## 参考文献 (References)

**一次資料 (Primary — 標準・仕様・査読論文)**

- ISO 11898-1:2024, *Road vehicles — Controller area network (CAN) — Part 1: Data link layer and physical coding sublayer*. — [ISO 11898-1:2024](https://www.iso.org/standard/86384.html) — Accessed: 2026-07-02。フレーム構造・調停・エラー処理・CAN FD の規範。
- Robert Bosch GmbH (1991). *CAN Specification Version 2.0*. — クラシック CAN（11/29bit ID）の原典仕様。
- ISO 14229-1, *Road vehicles — Unified diagnostic services (UDS)* ／ ISO 15765-2 (ISO-TP) ／ ISO 13400 (DoIP)。診断・転送層の規範。
- Koscher, K. et al. (2010). Experimental Security Analysis of a Modern Automobile. *IEEE Symposium on Security and Privacy*, pp. 447–462. DOI: 10.1109/SP.2010.34 — [PDF](http://www.autosec.org/pubs/cars-oakland2010.pdf) — Accessed: 2026-07-02。
- Miller, C., & Valasek, C. (2015). *Remote Exploitation of an Unaltered Passenger Vehicle*. — [PDF](http://illmatics.com/Remote%20Car%20Hacking.pdf) — Accessed: 2026-07-02。Jeep 遠隔操作研究の一次報告。

**二次資料 (Secondary)**

- CAN in Automation (CiA), *CAN knowledge pages*. — [can-cia.org](https://www.can-cia.org/can-knowledge) — Accessed: 2026-07-02。ビットタイミング、バス長の目安、CAN FD/XL の解説。標準化団体による技術解説。
