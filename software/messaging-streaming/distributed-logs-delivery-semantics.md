# 分散ログと配信セマンティクス (Distributed Logs, Delivery Semantics & Stream Processing)

## 概要 (Overview)

[`../distributed-systems/consensus.md`](../distributed-systems/consensus.md) は「合意の単位は値ではなくログである」と結論した。本ドキュメントはそのログを**それ自体が製品になった**姿——**分散ログ (distributed log)**——として扱う。Kafka に代表される分散ログは、パーティション化・複製された**追記専用 (append-only)** のレコード列を第一級の抽象として公開し、その上にメッセージング・ストリーム処理・システム間データ連携のすべてを載せる。

中心となる問いは4つ。(1) なぜ「キュー」ではなく「ログ」が統一抽象になったのか、(2) ブローカの複製 (ISR) は合意プロトコルとどう同じでどう違うか、(3) 「exactly-once 配信」はなぜ一般には不可能で、それでも「exactly-once **処理**」はなぜ達成できるのか、(4) ストリーム処理系 (Flink) の状態・時刻・スナップショットとバックプレッシャはどう成立しているか。対比軸として、従来型メッセージキュー (RabbitMQ/AMQP) を「同じ問題への別の分岐」として置く。分析深度は Level 3。

## 方法論 (Methodology)

- **抽象（ログ）と実装（Kafka）を分離**する。offsets・リテンション・コンシューマグループはログという抽象の帰結であり、Kafka 固有の癖と区別して論じる。
- 配信セマンティクスを「輸送の性質」ではなく**送信側リトライ × 受信側 ack タイミング × 状態更新の原子性**の合成として分解する。「exactly-once」を魔法ではなく既知の部品（冪等性＋原子的コミット）に還元する。
- 複製は [`../distributed-systems/consensus.md`](../distributed-systems/consensus.md) のクォーラム交差を基準系として、ISR 方式が**何を緩め、引き換えに何を得たか**を明示する。
- バックプレッシャは [`../networking/tcp-transport-congestion.md`](../networking/tcp-transport-congestion.md) のフロー制御と [`../../foundations/mathematics/probability-statistics/markov-chains-queueing.md`](../../foundations/mathematics/probability-statistics/markov-chains-queueing.md) の待ち行列理論に接続し、待ち行列の視点で定量的に読む。
- 一次資料（Kreps らの Kafka 論文、Chandy–Lamport、Flink/Dataflow 論文）に基づく。

## 発見事項 (Findings)

1. **ログが統一抽象である。** データベースの WAL（[`../databases/storage-engines.md`](../databases/storage-engines.md)）、合意プロトコルの複製ログ、メッセージブローカのトピック——いずれも「追記専用・順序付き・再読可能なレコード列」という同一の構造である。分散ログはこれを独立したサービスとして切り出したものであり、「メッセージング」は用途の一つにすぎない。ログを公開すれば、購読・再生 (replay)・複製・変更データキャプチャが**同じ read パス**になる。
2. **キューは消費で壊れ、ログは消費で壊れない。** 従来型キューは配達済みメッセージを削除するため、消費行為が状態を破壊する——複数の独立した読者・過去の再読・順序の保証がすべて難しくなる。ログは読み手の位置を**オフセット (offset)**——読者側が持つ単なる整数——に externalize し、ブローカを読者の数に対して無知にする。ファンアウトと再生はオフセットの帰結として無料で従う。
3. **ISR 複製は「クォーラムをリースで置き換えた」合意の親戚である。** Kafka の複製はメッセージごとの多数決ではなく、コントローラが管理する **ISR (in-sync replicas)** 集合への全複製で耐久性を定義する。多数決 (2f+1 で f 耐性) より少ないレプリカ (f+1 で f 耐性) で済む代わりに、ISR の管理そのもの——誰がリーダか、誰が同期内か——は本物の合意 (KRaft = Raft) に外部化されている。**データ平面は合意を薄め、制御平面に合意を凝縮する**という階層化である。
4. **exactly-once 配信は一般に不可能だが、exactly-once 処理は工学的に達成できる。** ack の喪失と受信者のクラッシュを区別できない以上（両将軍問題の変奏）、「ちょうど1回届ける」ことは輸送層では保証できない。達成できるのは「少なくとも1回届け、**2回目以降を副作用ゼロにする**」こと——すなわち**冪等性 (idempotence)** ＋ **消費位置と処理結果の原子的コミット**である。exactly-once とは配信の性質ではなく、端点の状態管理の性質である。
5. **ストリーム処理の難所は計算ではなく「時刻と状態」である。** イベント時刻と処理時刻の乖離（遅延・乱れ）は分散系の非同期性そのものであり、**ウォーターマーク (watermark)** は「これ以前のイベントはもう来ない」という**反証されうる賭け**である。状態の耐障害性は Chandy–Lamport 由来の整合スナップショットに帰着する——1985年の理論がそのまま Flink のチェックポイントになっている。
6. **バックプレッシャは、システムが容量について正直になる機構である。** 到着率が処理率に迫ると待ち行列は非線形に伸びる（M/M/1 の 1/(1−ρ)）。無制限バッファは遅延と OOM に問題を変換するだけで、解決しない。pull 型消費・クレジット型フロー制御は「受け手が受け入れ可能量を宣言する」ことで過負荷を**上流の送信抑制**へ伝播させる——TCP の受信ウィンドウと同形である。

## アーキテクチャ (Architecture)

### パーティション化された複製ログ

```
 Producers                    Topic "orders" (3 partitions)             Consumer Group A
 ┌────┐  key=K1 ─┐   ┌─────────────────────────────────────┐   ┌─▶ ┌──────────┐
 │ P1 │──────────┼──▶│ partition 0: [0][1][2][3][4][5] ──▶ 追記 │───┘   │ consumer1 │ offset=4
 └────┘          │   ├─────────────────────────────────────┤       └──────────┘
 ┌────┐  key=K2 ─┤   │ partition 1: [0][1][2][3] ────────▶ 追記 │──────▶ ┌──────────┐
 │ P2 │──────────┘   ├─────────────────────────────────────┤       │ consumer2 │ offset=2
 └────┘              │ partition 2: [0][1][2][3][4] ──────▶ 追記 │───┐   └──────────┘
                     └─────────────────────────────────────┘   └─▶ (consumer2 が兼任)
                       各パーティション = 独立した順序付きログ          Group B は独立した
                       同一キーは同一パーティションへ (順序保証の単位)     オフセットで同じログを再読
```

- **順序はパーティション内のみ**。トピック全体の全順序は放棄し、その代わりパーティション数だけ並列度が得られる——[`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) の「全順序は高くつく」の実践形である。
- **コンシューマグループ (consumer group)**: グループ内でパーティションを排他的に分担（キュー的な負荷分散）し、グループ間では同じログを独立に読む（pub/sub 的なファンアウト）。キューと pub/sub の二択が、ログ＋オフセットの上では**同じ機構の設定違い**になる。
- 各パーティションはリーダブローカ＋フォロワ群に複製される（技術詳細 §2）。

### 分岐点 — メッセージキューとの対比

| 軸 | 従来型キュー (RabbitMQ/AMQP) | 分散ログ (Kafka) |
|---|---|---|
| 単位 | メッセージ（配達で削除） | レコード（追記、期限まで保持） |
| 消費位置 | ブローカが個別 ack を管理 | 読者がオフセットを管理 |
| 再配達 | 未 ack を再キュー（順序は崩れうる） | オフセット巻き戻し（パーティション内順序維持） |
| 得意領域 | タスク分配・個別再試行・複雑ルーティング (exchange) | 高スループット・再生・複数購読者・ストリーム処理基盤 |
| 遅い消費者 | キュー成長がブローカを圧迫 | ラグ（オフセット差）として読者側に局所化 |

per-message ack は個々のジョブの完了追跡には強いが、ブローカがメッセージごとの状態機械を持つ分だけスループットと再生可能性を失う。**どちらが優れているかではなく、状態を誰が持つかの分岐**である。

## 技術詳細 (Technical Details)

### 1. ログの構造 — オフセット・リテンション・コンパクション

- パーティションはセグメントファイルの列で、追記とシーケンシャル読みだけを行う——[`../databases/storage-engines.md`](../databases/storage-engines.md) の「ランダムを避ける契約」をそのまま実装した形であり、ページキャッシュと sendfile による zero-copy が高スループットの源泉になる。
- **リテンション (retention)**: 時間/サイズで古いセグメントを丸ごと破棄する。読者の消費と削除が無関係なのが本質。
- **ログコンパクション (log compaction)**: キーごとに最新値だけを残す圧縮。ログが「イベントの列」から「テーブルの変更履歴＝最新状態の複製手段」へ変わる。テーブルとストリームの双対性 (stream–table duality) の機構的根拠であり、LSM-tree の compaction と同じ操作の別用途である。

### 2. 複製と耐久性 — ISR・acks・unclean election

- リーダが書き込みを受け、フォロワが**pull で追随**する。**ISR** は「リーダに十分追随しているレプリカの集合」で、**high watermark（ISR 全員が持つ最小オフセット）** までがコミット済み——コンシューマに見えるのはここまで。
- **acks 設定**が耐久性の契約を決める: `acks=0`（投げっぱなし）/ `acks=1`（リーダのみ——リーダ障害で喪失しうる）/ `acks=all` ＋ `min.insync.replicas`（ISR 全員への複製を待つ）。合意プロトコルが固定で払うコストを、Kafka はトピックごとのダイヤルにした。
- **unclean leader election**: ISR が空になったとき、同期外レプリカをリーダに昇格させるか。昇格させれば可用性と引き換えに**コミット済みデータの喪失**（safety 違反）を許す。合意プロトコルが決して売らないものを、明示的なオプションとして売っている点が思想の違いを最もよく示す。
- **リーダエポック (leader epoch)**: リーダ交代の世代番号。フォロワは high watermark の巻き戻しではなくエポック境界で自ログを切り詰めることで、交代の連鎖によるログ食い違い（合意プロトコルの Log Matching が防ぐのと同種の問題）を修復する——ISR 方式も結局、リーダ中心複製の古典的難所を一つずつ踏み直している。
- **KRaft**: かつて ZooKeeper に置いていたメタデータ（ブローカ生死・パーティション割当・ISR）を、Kafka 自身が持つ Raft 実装のメタデータログに置き換えた。制御平面は正真正銘の合意で守り、データ平面は ISR で軽くする——発見事項3の階層化の完成形である。

### 3. 配信セマンティクス — 冪等性＋トランザクション

失敗時の振る舞いは3値に分類される:

| セマンティクス | 送信側 | 受信側 (ack と処理の順) | 失敗時 |
|---|---|---|---|
| at-most-once | リトライしない | 先に ack（オフセットコミット）、後で処理 | 欠落しうる |
| at-least-once | ack が来るまでリトライ | 先に処理、後で ack | 重複しうる |
| exactly-once **処理** | リトライ＋冪等化 | 処理と ack を**原子的に**コミット | どちらも起きない |

- **冪等プロデューサ**: プロデューサ ID とシーケンス番号をレコードに付け、ブローカが重複追記を弾く。TCP のシーケンス番号による重複除去と同じ発想を、セッションを跨いで永続化したもの。ブローカ内の「書き込みの exactly-once」を作る。
- **トランザクション**: 複数パーティションへの書き込みと**消費オフセットのコミット**（オフセット自体が内部トピックへの書き込みである点が鍵）を、トランザクションコーディネータ経由の 2 相コミットで原子化する。読者は `read_committed` でコミット済みのみを読む。これで「読む→処理する→書く→オフセットを進める」が全体として原子になり、consume-transform-produce 連鎖の exactly-once 処理が成立する。分離性の語彙は [`../databases/transactions-isolation.md`](../databases/transactions-isolation.md) のそれがそのまま通用する。
- 外部システムへの副作用（メール送信、決済 API）はトランザクション境界の外であり、そこは**受け手側の冪等キー**で守るしかない——exactly-once の保証域は常に「原子的コミットが届く範囲まで」である。

### 4. ストリーム処理 — 時刻・ウィンドウ・スナップショット

- **イベント時刻 (event time) vs 処理時刻 (processing time)**: 事象が起きた時刻と系がそれを観測した時刻は、遅延・リトライ・オフライン端末により恣意的に乖離する。処理時刻でのウィンドウ集計は再実行のたびに答えが変わる。正しさを求めるならイベント時刻で計算するしかない——が、すると「ウィンドウはいつ閉じてよいか」という完備性の問題が生じる。
- **ウォーターマーク (watermark)**: 「イベント時刻 t 以前はもう到着しない」という系内を流れる表明。本質的に**ヒューリスティックな賭け**であり、遅刻データ (late data) は必ず存在しうる。Dataflow モデルはこれを「ウィンドウ＋トリガ＋遅刻データの訂正発火」として体系化し、正確性・遅延・コストの三者トレードオフを API に昇格させた。
- **ウィンドウ**: tumbling / sliding / session。集計はウィンドウごとの**状態 (state)** をキー別に保持することであり、ストリーム処理系の実体は「状態を持つ演算子の分散 DAG」である。状態バックエンドには RocksDB のような組込み LSM エンジンが使われる——[`../databases/storage-engines.md`](../databases/storage-engines.md) の構造がここにも現れる。
- **チェックポイント (Flink)**: 演算子状態の耐障害性は、Chandy–Lamport の分散スナップショットの変奏である**非同期バリアスナップショット**で守る。ソースが**バリア (barrier)** をログのオフセットとともに流し、各演算子はバリア到達時（複数入力ではバリア整列後）に自状態を退避する。障害時は最新スナップショットへ巻き戻し、記録したオフセットからログを再生する——**ログの再生可能性（§1）がなければこの回復モデル自体が成立しない**。スナップショット＋トランザクショナルシンクで、パイプライン端から端までの exactly-once 処理になる。

### 5. バックプレッシャ — フロー制御と待ち行列

- Kafka の消費は **pull 型**であり、遅い消費者は単に読むのが遅れるだけ——過負荷は「コンシューマラグ」という観測可能な整数に変換され、ブローカは守られる。push 型のキューでは同じ状況がブローカのメモリ圧として現れる。
- Flink のタスク間は **credit-based flow control**: 受信側が自分の空きバッファ数をクレジットとして通告し、送信側はクレジット分しか送らない。TCP の受信ウィンドウ（[`../networking/tcp-transport-congestion.md`](../networking/tcp-transport-congestion.md)）と同じ機構をアプリ層で再演したものであり、輻輳がチェーンを遡ってソースの消費速度（＝ログを読む速度）まで自然に減速する。
- 定量的には [`../../foundations/mathematics/probability-statistics/markov-chains-queueing.md`](../../foundations/mathematics/probability-statistics/markov-chains-queueing.md) の領分: 利用率 ρ→1 で待ち時間は 1/(1−ρ) で発散するから、**定常運転は容量に余裕がある状態でしか成立しない**。バックプレッシャは障害ではなく、この数学をシステムが認めた正直な信号である。ラグ・スループット・p99 遅延の観測は [`../observability/observability-performance-engineering.md`](../observability/observability-performance-engineering.md) の手法に接続する。

## セキュリティ考察 (Security Considerations)

- **認証と認可**: ブローカは組織の全イベントが通る要衝であり、無認証クラスタは全データの読み書き自由を意味する。実務は mTLS または SASL (SCRAM/Kerberos/OAUTHBEARER) による認証＋トピック/グループ単位の ACL による認可が最低線。KRaft メタデータログ（＝クラスタの制御平面）への到達は特に厳格に絞る。
- **マルチテナント分離とクォータ**: 同一クラスタの共有では、1テナントの暴走プロデューサ/コンシューマが他を飢えさせうる。produce/fetch のクォータ（帯域・リクエストレート）はセキュリティ機構として扱うべきで、パーティション数の暴発（メタデータ圧迫）も含めて上限を課す。
- **ポイズンメッセージと DoS**: 解析不能・処理不能なレコードは at-least-once の下で無限再試行ループを作り、パーティション全体の消費を止める（ラグの単調増大として現れる）。dead letter queue への退避と再試行上限は可用性防御である。
- **at-least-once の重複は正確性の危険である**: 「再配達で二重課金」は輸送の仕様であって異常ではない。冪等キーなしで金銭・在庫系の副作用を書くことは、仕様どおりの動作で事故る設計である。重複除去は受け手の責務として明示的に実装・監査する。
- **監査ログとしての不変性**: 追記専用のログは改竄検出可能な監査証跡の良い土台だが、リテンション設定・コンパクション・管理者権限による削除はその前提を破りうる。監査用途では削除系 API の権限分離と、保持期間の規制要件（と GDPR 型の削除権との緊張）を設計時に固定する。

## 結論 (Conclusions)

- 分散ログは WAL・合意ログ・メッセージングを貫く同一抽象の製品化であり、オフセットの外部化がファンアウト・再生・キュー的分配を単一機構に統合する。
- 複製の設計は「データ平面は ISR で軽く、制御平面は Raft で固く」という階層化であり、acks や unclean election は合意プロトコルが固定していたトレードオフを利用者のダイヤルへ開いたものである。
- exactly-once は配信の魔法ではなく、**冪等性＋消費位置と結果の原子的コミット**という端点の規律である。保証域は原子的コミットの届く範囲で終わり、外部副作用は冪等キーで守る。
- ストリーム処理の核心は時刻（ウォーターマークという賭け）と状態（Chandy–Lamport 由来のスナップショット）であり、その回復モデルはログの再生可能性の上にしか立たない。
- バックプレッシャは待ち行列理論の帰結を上流へ伝える正直な信号であり、pull 消費・クレジット制御は TCP フロー制御の再演である。

### 次の接続先 (Related Topics)

- [`../distributed-systems/consensus.md`](../distributed-systems/consensus.md) — ISR/KRaft の基準系となる合意・複製状態機械・リーダ交代。
- [`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) — パーティション内順序のみという選択は一貫性ダイヤルの位置決めである。
- [`../databases/storage-engines.md`](../databases/storage-engines.md) — セグメント追記・fsync・コンパクションの物理的土台 (WAL/LSM)。
- [`../databases/transactions-isolation.md`](../databases/transactions-isolation.md) — read_committed・2相コミット・冪等性の語彙の供給元。
- [`../networking/tcp-transport-congestion.md`](../networking/tcp-transport-congestion.md) — クレジット型フロー制御の原型としての受信ウィンドウと輻輳制御。
- [`../../foundations/mathematics/probability-statistics/markov-chains-queueing.md`](../../foundations/mathematics/probability-statistics/markov-chains-queueing.md) — ラグと遅延の定量モデル (M/M/1, 1/(1−ρ))。

## 参考文献 (References)

**一次資料 (Primary — 原論文・一次解説)**

- Kreps, J., Narkhede, N., & Rao, J. (2011). Kafka: a Distributed Messaging System for Log Processing. *NetDB '11*. — Kafka の原典。ログ抽象・pull 消費・オフセット外部化の設計根拠。
- Kreps, J. (2013). The Log: What every software engineer should know about real-time data's unifying abstraction. — [LinkedIn Engineering blog](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) — Accessed: 2026-07-18。発見事項1「ログが統一抽象」の出典。
- Chandy, K. M., & Lamport, L. (1985). Distributed Snapshots: Determining Global States of Distributed Systems. *ACM TOCS*, 3(1), 63–75. DOI: 10.1145/214451.214456 — 整合スナップショットの原典。Flink チェックポイントの理論的祖先。
- Carbone, P., Katsifodimos, A., Ewen, S., Markl, V., Haridi, S., & Tzoumas, K. (2015). Apache Flink: Stream and Batch Processing in a Single Engine. *IEEE Data Engineering Bulletin*, 38(4). — Flink のアーキテクチャと非同期バリアスナップショット。
- Akidau, T., et al. (2015). The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing. *VLDB*, 8(12). — イベント時刻・ウォーターマーク・トリガの体系化。

**二次資料 (Secondary — 解説)**

- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly. — ログ・複製・exactly-once を横断的に整理した標準的解説書。
