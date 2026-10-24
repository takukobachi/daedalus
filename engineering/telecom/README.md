# 通信インフラ (Telecom)

国や大陸をまたぐ通信網の物理的な設備を題材にした工学分析。ケーブル・中継器・給電・敷設船・陸揚げ局といった「通信を運ぶ物」を、故障・修理・所有・規制まで含めた一つのシステムとして扱う。

データセンターの中の光部品は [`../../hardware/networking/`](../../hardware/networking/)、その上を流れる経路制御やプロトコルは [`../../software/networking/`](../../software/networking/) が扱う。本ディレクトリは、それらが前提にしている「建屋の外、国境の外の伝送路」を担当する。

## ドキュメント

- [`submarine-cables.md`](submarine-cables.md) — 海底ケーブル (submarine cables)。外装の深浅による違い、EDFA 中継器と定電流給電 (PFE)、電力の予算から生まれた SDM への転換、ブランチングユニット、故障統計 (年 150〜200 件、大半が漁業と錨)、修理船の不足と修理日数、コンソーシアムからハイパースケーラへの所有の移行、トンガ・紅海・バルト海の事例、Team Telecom と FCC の陸揚げ規制、衛星との補完

## 今後のテーマ候補

- 陸上の長距離光伝送網 (terrestrial long-haul optical networks) — ROADM、C+L 帯、ファイバの敷設経路と鉄道・道路の共用
- 携帯電話網の無線アクセス (radio access network) — 基地局、フロントホール、Open RAN
- 衛星通信の地上局とゲートウェイ (satellite ground segment) — 海底ケーブルとの経路の分担
