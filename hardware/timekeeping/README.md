# 時刻同期とクロック (Timekeeping & Clocks)

「いま何時か」を装置間・大陸間で一致させる技術の分析。時刻同期は「物理時計側」の土台であり、`../../software/distributed-systems/` が扱う論理時計・一貫性モデルの議論と対をなす層である。`../hw-sw-interface/clock-power-reset.md` が扱うチップ内のクロック生成・分配を、発振器の物理から GNSS・ネットワーク時刻プロトコルを経て分散システムの API に至るスケールへ拡張する。ここでの「時計の不確かさの桁」の理解が、分散データベースや TSN の設計判断を読み解く鍵になる。

## ドキュメント

- [`clocks-time-synchronization.md`](clocks-time-synchronization.md) — 時刻同期とクロック (timekeeping & clock synchronization)：水晶発振器とアラン偏差、原子時計、TAI/UTC/うるう秒、GNSS 時刻配信、NTP/PTP/White Rabbit、TrueTime と分散システムへの橋渡し

## 今後のテーマ候補

- チップスケール原子時計と保持時計 (CSAC & holdover clocks) — GNSS 遮断時の自律計時、
  電力・通信インフラの耐障害設計。`../power-generation/` の系統保護と交差
- 金融・放送のタイムスタンプ規制 (regulatory timestamping) — MiFID II の 100 µs 要件、
  SMPTE 2110/PTP による放送設備同期。「精度が法規制になる」ドメインの分析
- 光格子時計と秒の再定義 (optical lattice clocks & SI second redefinition) — 10⁻¹⁸ 級の
  周波数安定度、秒の定義改定ロードマップ、相対論的測地への応用
