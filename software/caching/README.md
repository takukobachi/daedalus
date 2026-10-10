# Caching

キャッシュ (caching) の分析。データベースの前に置く分散キャッシュ (memcached, Redis) を、
データベースの値の写しを、データベースとは別の一貫性と障害の単位で持つ層として扱う。
一貫性ハッシュとリバランスの一般論は [`../distributed-systems/`](../distributed-systems/README.md)、
エッジのキャッシュ階層は [`../networking/cdn-edge-delivery.md`](../networking/cdn-edge-delivery.md)、
CPU のキャッシュは [`../../hardware/cpu/cache-memory-hierarchy.md`](../../hardware/cpu/cache-memory-hierarchy.md)
が扱う。

## 構成

- [`distributed-cache-architecture.md`](distributed-cache-architecture.md) — 分散キャッシュの
  アーキテクチャ：利用パターン（キャッシュ・アサイド、リードスルー、ライトスルー、
  ライトバック）、無効化と古い値の書き戻し（リース、cas）、スタンピードとホットキー
  （XFetch、L1 キャッシュ）、配置とリバランス（一貫性ハッシュ、Redis Cluster の
  ハッシュスロット）、置換方式（近似 LRU/LFU、W-TinyLFU、スラブ）、ヒット率と
  キャッシュ喪失によるメタステーブル障害、Redis と memcached の設計比較とライセンスの
  経緯、セキュリティ（UDP 増幅、認証のない Redis、テナント間の漏洩）

## 今後のテーマ候補

- アプリケーション内キャッシュとメモ化 — Caffeine/Guava、HTTP キャッシュヘッダとの
  関係、計算結果のキャッシュと無効化
- 永続キャッシュとティアリング — SSD を使う大容量キャッシュ (CacheLib 型)、
  [`../../hardware/storage/`](../../hardware/storage/README.md) の NAND 特性との相互作用
