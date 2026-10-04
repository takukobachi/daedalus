# Messaging & Streaming

メッセージングとストリーム処理 (messaging & stream processing) の分析。分散ログ
(distributed log) を統一抽象として、ブローカの複製・配信セマンティクス・ストリーム
処理基盤を扱う。`../distributed-systems/consensus.md` の「合意の単位はログである」
という結論を、ログそのものが製品となった側から捉え直す領域であり、
`../databases/storage-engines.md` の WAL・追記構造と地続きになっている。

## 構成

- `distributed-logs-delivery-semantics.md` — 分散ログと配信セマンティクス：
  パーティション化された複製ログ (Kafka)、オフセットとコンシューマグループ、
  ISR 複製と KRaft、at-most/at-least/exactly-once（冪等プロデューサ＋トランザクション）、
  ストリーム処理の基礎（イベント時刻・ウォーターマーク・Flink のスナップショット）、
  バックプレッシャ、メッセージキュー (RabbitMQ/AMQP) との対比

## 今後のテーマ候補

- ストリーム処理エンジンの深掘り (stream processing engines) — Flink/Kafka Streams
  の状態バックエンド、ジョイン、リスケーリング
- スキーマ進化とデータパイプライン (schema evolution & data pipelines) — Avro/Protobuf、
  互換性ルール、CDC (change data capture)
- ブローカ内部の性能分析 — zero-copy、ページキャッシュ、tiered storage
