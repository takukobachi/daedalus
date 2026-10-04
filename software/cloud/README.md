# Cloud Infrastructure

クラウド基盤 (cloud infrastructure) の分析。仮想化・分散システム・ネットワーキング・
ストレージの一般機構が、**マルチテナント (multi-tenancy) と従量課金 (pay-per-use)**
という制約の下でどう再設計され、製品化されるかを読む。データセンターというコンピュータを、
Android / iOS / ブラウザに続く「第四のプラットフォーム」として内部から分析する。
AWS を全体を通した主要な事例 (running example) とし、標準・論文・オープン実装と相互検証する。

既存スタックとの関係は次のとおりである。

- 実行基盤は [`../virtualization/`](../virtualization/)（Firecracker, コンテナ in VM）の真上
- ネットワークは [`../networking/virtual-networks-overlay.md`](../networking/virtual-networks-overlay.md)（VPC, Nitro オフロード）と対で読む
- 信頼性機構は [`../distributed-systems/`](../distributed-systems/) の合意・一貫性の応用層
- ストレージは [`../object-storage/`](../object-storage/)（S3 型設計）が既に扱う

## 構成

- `serverless-faas-internals.md` — サーバーレス実行基盤 (FaaS) の内部：コールドスタートの
  分解、Firecracker 上のサンドボックス管理、コンテナイメージの遅延ロードと重複排除、
  スナップショット復元と一意性回復、keep-warm とワークロードの歪み
- `cloud-control-planes.md` — コントロールプレーン設計：コントロール/データプレーン分離、
  静的安定性 (static stability)、セルベースアーキテクチャとブラストラディウス、
  シャッフルシャーディング、リトライの増幅とメタステーブル障害、アドミッション制御
- `cloud-iam-authorization.md` — クラウド IAM と認可：SigV4 署名とステートレス認証、
  ポリシー評価論理（明示的拒否・境界・和と積）、STS と一時クレデンシャル、
  ポリシーの形式検証 (Zelkova/Cedar)、confused deputy と IMDS、Zanzibar との対比
- `cloud-databases-aurora-dynamodb.md` — クラウドデータベースの設計系譜：Aurora の
  「ログこそデータベース」（6/4/3 クォーラム, VDL 水位線, リカバリの定常化）と
  DynamoDB の「予測可能性こそ商品」（有界リクエスト, Multi-Paxos, GAC, MemDS,
  ワンショットトランザクション）を「何を約束し続けるか」の対として読む
- `cloud-block-storage.md` — ブロックストレージ (EBS 型)：ブロック意味論の契約と
  チェーン複製, 構成管理の分業 (Physalia), 2011 年再ミラーリングの嵐, スナップショットの
  増分参照構造と lazy restore, SRD/Nitro オフロード, 故障ドメイン分離という商品
- `container-orchestration-kubernetes.md` — コンテナオーケストレーション：宣言的状態と
  level-triggered reconciliation, etcd と楽観並行制御による振り付けなし協調,
  スケジューリングパイプライン, EKS の接合部 (IRSA/VPC CNI/Karpenter),
  namespace はテナント境界ではないという等級論
- `multi-region-global-architecture.md` — マルチリージョンとグローバルサービス：
  リージョン独立性という資産と隠れ結合 (2021-12 us-east-1), 光速が引く同期/非同期境界,
  RTO/RPO の値段表, 競合解決の 4 択 (回避/LWW/CRDT/合意, Spanner 対比),
  フェイルオーバー工学と静的安定な引き金
- `cloud-kms-confidential-computing.md` — 鍵管理と機密計算：鍵を出さず演算を売る KMS,
  エンベロープ暗号化と encryption context, 鍵階層と主権の選択 (BYOK/外部キーストア),
  Nitro Enclaves の隔離と attestation (「コードへの認可」), 保証等級の正直な評価
- `spot-capacity-economics.md` — スポット市場と容量経済：価格モデル＝容量リスクの配分契約,
  入札の時代から中断設計の時代へ (独立検証 ACM TEAC 2013), 120 秒の退去プロトコル,
  プール分散という相関回避, 容量ポートフォリオ, 余剰/逼迫レジームと GPU の予約回帰
- `infrastructure-as-code.md` — Infrastructure as Code：意図・記録・現実の三体問題,
  差分計画のコンパイラという読み方, 置換 (replacement) と事故の機構, drift と常時調停
  (GitOps) の適用範囲, saga 型ロールバック vs 停止, policy as code というガバナンス執行点
- `data-lakehouse-analytics.md` — データレイクハウス：列指向 (Parquet) と「読まない技術」,
  テーブル形式 (Iceberg) によるオブジェクトストア上の ACID の買い戻し (メタデータ木 +
  ポインタ CAS), カタログ＝コントロールプレーン, compaction という LSM の再演,
  分離クエリエンジンとスキャン課金の経済
- `event-driven-architecture.md` — イベント駆動とワークフロー：時間的独立という商品と
  分散トランザクションという代金, 4 つの抽象 (キュー/トピック/バス/ログ) の座標,
  at-least-once + 冪等受信・saga・outbox という規律, オーケストレーション vs
  コレオグラフィ, DLQ と毒メッセージ
- `cloud-network-topology.md` — ネットワークトポロジー：到達性は負債という第一原理,
  ピアリングの非推移性 → TGW のハブ&スポーク → PrivateLink という進化, 経路表という
  ポリシー言語, ハイブリッド接続と分割解決 DNS, CIDR 計画, ゼロトラスト下の境界の位置づけ
- `cloud-observability.md` — クラウド可観測性：外形計測の内蔵と内部計装の二重視点,
  カーディナリティの経済 (Gorilla/Monarch と課金構造), 信号の相互変換, 監査 (CloudTrail)
  と運用 (CloudWatch) の分離, 欠損データの意味論, OTel による計装の脱ベンダー化
- `cloud-design-principles.md` — **総括**：領域 14 本を貫く七つの設計原則 (相関の破壊,
  制御/実行の分離, ログこそデータベース, 計算と記憶の分離, 機能の削除, 文脈の明示化と
  機械検証, 難しさの移動) の帰納と、原則 × 文書の出現マップ・学習経路・設計レビュー
  への変換。まずここから読み、各論へ降りる入口としても使える

なおエッジ配信 (CloudFront 型) は [`../networking/cdn-edge-delivery.md`](../networking/cdn-edge-delivery.md)
が CloudFront を事例に含めて既に Level 3 で扱っており、本領域では重複させない。

## 今後のテーマ候補

- マルチテナント SaaS 設計 — テナント隔離のパターン (silo/pool/bridge), テナントごとの
  計量と noisy neighbor, `cloud-control-planes.md` の隔離 vs 多重化の応用層
- AI/エージェント基盤 — マネージド推論 (Bedrock 型) の計量・隔離・容量,
  [`../ai/`](../ai/) の製品側からの再訪。エージェントの認可は
  `cloud-iam-authorization.md` の「認可の主語の拡張」の次の段
