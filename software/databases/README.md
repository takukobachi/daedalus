# Databases

データベースとストレージエンジン (databases & storage engines) の分析。
永続化 (durability) と効率的な検索のためにデータをディスク/SSD 上にどう配置し、
障害をまたいで整合性をどう保つかを扱う。`hardware/hw-sw-interface/storage-nvm-interface.md`
（ブロックデバイス・NVM の物理特性）の真上の層にあたる。

## 構成

- `storage-engines.md` — ストレージエンジンの心臓部（B+tree vs LSM-tree、
  WAL、バッファプール、ページレイアウト、圧縮・コンパクション）

## 今後のテーマ候補

- トランザクションと分離レベル (transactions & isolation) — ACID, MVCC,
  snapshot isolation, 2PL, serializability（`../concurrency/` と `../distributed-systems/` の交点）
- クエリ実行 (query execution) — プランナ, join アルゴリズム, ベクトル化実行
- インデックス設計 (indexing) — 複合インデックス, カバリング, 全文・空間索引
- 分散データベース (distributed databases) — sharding, 分散トランザクション,
  分散コミット（`../distributed-systems/consistency-models.md` の応用）
