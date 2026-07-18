# Object Storage

オブジェクトストレージ (object storage) の分析。フラットなキー空間と不変オブジェクトの
全体 PUT/GET という貧しい抽象が、いかにして惑星規模のスケールと「11 ナイン」の
耐久性を買ったかを扱う。メタデータ層（索引）は `../distributed-systems/consensus.md` の
合意と `../databases/storage-engines.md` の LSM/WAL がそのまま再利用される場であり、
データ層（倉庫）はイレイジャーコーディングと背景修復による「ドルあたり耐久性」の
経済工学である。一貫性の語彙は `../distributed-systems/consistency-models.md`、
耐久性の数学は `../../foundations/mathematics/probability-statistics/reliability-theory.md`
と地続きになっている。

## 構成

- `object-storage-architecture.md` — オブジェクトストレージのアーキテクチャ：
  オブジェクトモデル（POSIX の放棄、マルチパートアップロード、バージョニング）、
  メタデータ層とデータ層の分離（S3 型パーティション索引 vs Ceph/CRUSH 計算配置）、
  耐久性の工学（複製 vs Reed–Solomon EC、修復増幅、故障ドメイン、ナインズの算術、
  スクラビング）、一貫性（S3 の 2020 年強一貫性化、条件付き書き込み）、
  スケールアウト機構とコールドティア、セキュリティ（署名付き URL・公開バケット
  誤設定・オブジェクトロック）

## 今後のテーマ候補

- データレイクとテーブル形式 (data lakes & table formats) — Parquet/Iceberg/Delta が
  オブジェクトストレージの上にトランザクションとスキーマ進化を組み立てる機構
- CDN とエッジキャッシング (CDN & edge caching) — オリジンとしてのオブジェクト
  ストレージ、キャッシュ無効化、署名付き配信
- 修復スケジューリングの深掘り — 修復帯域と耐久性モデルの結合、優先度付け、
  相関故障下の挙動
