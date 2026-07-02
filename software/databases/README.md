# Databases

データベースとストレージエンジン (databases & storage engines) の分析。
永続化 (durability) と効率的な検索のためにデータをディスク/SSD 上にどう配置し、
障害をまたいで整合性をどう保つかを扱う。`hardware/hw-sw-interface/storage-nvm-interface.md`
（ブロックデバイス・NVM の物理特性）の真上の層にあたる。

## 構成

- `storage-engines.md` — ストレージエンジンの心臓部（B+tree vs LSM-tree、
  WAL、バッファプール、ページレイアウト、圧縮・コンパクション）
- `transactions-isolation.md` — トランザクションと分離レベル：異常カタログ (dirty read〜write skew)、
  ANSI 定義の批判、2PL/MVCC/SI/SSI、実 DB の挙動差 (PostgreSQL/MySQL/Oracle)、Hermitage 検証

## 今後のテーマ候補

- クエリ実行 (query execution) — プランナ, join アルゴリズム, ベクトル化実行
- インデックス設計 (indexing) — 複合インデックス, カバリング, 全文・空間索引
- 分散データベース (distributed databases) — sharding, 分散トランザクション,
  分散コミット（`../distributed-systems/consistency-models.md` の応用）
