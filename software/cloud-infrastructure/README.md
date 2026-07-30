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

## 今後のテーマ候補

- ブロックストレージ (EBS 型) — チェーン複製, ボリューム仮想化, スナップショットの増分構造。
  `physalia` (設定ストア) は `cloud-control-planes.md` が扱済み
- クラウドデータベースの設計系譜 — DynamoDB (ATC '22) と Aurora (SIGMOD '17) を
  「計算と記憶の分離」「ログこそデータベース」の対として読む。
  [`../databases/`](../databases/) のストレージエンジン論の応用層
- コンテナオーケストレーション — Kubernetes の宣言的状態と reconciliation loop, etcd,
  スケジューラ。`cloud-control-planes.md` の望ましい状態 (desired state) 論の一般化
- マルチリージョンとグローバルサービス — リージョン隔離の設計原則, グローバルテーブル,
  コンフリクト解決。[`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) の実地応用
- スポット市場と容量経済 — 統計多重化の価格設計, interruption notice, 容量予約
