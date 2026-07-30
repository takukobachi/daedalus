# Cloud Infrastructure

クラウド基盤 (cloud infrastructure) の分析。仮想化・分散システム・ネットワーキング・
ストレージの一般機構が、**マルチテナント (multi-tenancy) と従量課金 (pay-per-use)**
という制約の下でどう再設計され、製品化されるかを読む。Android / iOS / ブラウザに続く
「第四のプラットフォーム」——データセンターというコンピュータ——の内部分析であり、
AWS を主要な走る事例 (running example) として、標準・論文・オープン実装と相互検証する。

既存スタックとの関係:

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

## 今後のテーマ候補

- コンテナオーケストレーション — Kubernetes の宣言的状態と reconciliation loop, etcd,
  スケジューラ。`cloud-control-planes.md` の望ましい状態 (desired state) 論の一般化
- マルチリージョンとグローバルサービス — リージョン隔離の設計原則, グローバルテーブル,
  コンフリクト解決。[`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) の実地応用
- スポット市場と容量経済 — 統計多重化の価格設計, interruption notice, 容量予約
