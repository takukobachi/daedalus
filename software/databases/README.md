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
- `query-execution-optimization.md` — クエリ実行と最適化：System R/Volcano/Cascades、
  カーディナリティ推定の失敗モード (JOB)、結合アルゴリズムと spilling (grace/hybrid hash)、
  Volcano iterator vs ベクトル化 (X100/DuckDB) vs コンパイル (HyPer)、morsel 並列、適応実行
- `distributed-databases.md` — 分散データベース：分割方式とホットスポット、2PC の阻塞と Paxos Commit、
  Percolator/Spanner TrueTime/CockroachDB (HLC・parallel commits)/Calvin、分散クエリ、グローバル二次索引、失敗モード

## 今後のテーマ候補

- インデックス設計 (indexing) — 複合インデックス, カバリング, 全文・空間索引
