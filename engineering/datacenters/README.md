# データセンター (Datacenters)

計算機を「数万台まとめて一つの機械として運転する」ための物理インフラ（給電・冷却・
ネットワークトポロジ・可用性設計）の分析。`../energy/`（系統側）と
計算機システム各領域（`../../hardware/accelerators/`・`../../hardware/networking/` ほか）の合流点である。
AI 学習クラスタの電力・冷却密度は、この層の設計を根本から書き換えつつある。
その現状を「ワットと熱の会計」として読む。

## ドキュメント

- [`datacenter-fundamentals.md`](datacenter-fundamentals.md) — データセンター基礎：
  「倉庫規模の計算機 (warehouse-scale computer)」という見方、可用性クラス（Tier）と
  冗長化の語彙、PUE/WUE という会計指標、Clos/リーフスパイントポロジと
  オーバーサブスクリプション、AI がもたらした密度の転換（2026 年時点）
- [`datacenter-power.md`](datacenter-power.md) — 給電：系統受電から CPU の 1 V まで
  の変換チェーン、UPS の方式と電池、冗長トポロジ (N+1/2N)、ラック給電の 48 V 化 (OCP)、
  電力キャッピングとオーバーサブスクリプション、AI 負荷の変動が系統に見せる新しい顔
- [`datacenter-cooling.md`](datacenter-cooling.md) — 冷却：全ワットは熱になるという
  出発点、空冷の完成形（封じ込め・エコノマイザ）とその限界、液冷への移行
  (リアドア → コールドプレート → 液浸)、CDU と冷却分配、水の会計 (WUE)、排熱利用

## 今後のテーマ候補

- データセンターネットワークの物理配線 — ファブリックの経路制御と Jupiter・Orion の
  集中制御は `../../software/networking/dc-fabric-routing.md`、OCS の運用は
  `../../hardware/networking/optical-interconnect.md`、テナントのオーバーレイは
  `../../software/networking/virtual-networks-overlay.md` が扱った。残るのは建屋側の
  物理層で、構造化ケーブリングと光ファイバのトランク、棟間・キャンパス間の接続 (DCI)、
  配線の増設と保守の運用である
- サイト選定と電力調達 — 立地の物理（送電容量・水・気候・レイテンシ）、PPA と
  自家発電、原子力 SMR 併設の動向。`../energy/electricity-markets-dispatch.md` の応用
- 物理セキュリティと運用 — 入退管理、コミッショニング、変更管理の失敗学
