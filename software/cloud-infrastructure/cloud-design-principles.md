# クラウド設計原則の総括 (A Synthesis of Cloud Design Principles)

## 概要 (Overview)

本ドキュメントは `cloud-infrastructure/` の 14 本を貫く**設計原則の抽出と統合**である。個々のドキュメントは AWS の特定の層——サーバーレス、コントロールプレーン、IAM、データベース、ストレージ、オーケストレーション、ネットワーク——を分析したが、読み終えて残るべきものはサービスの知識ではない。**同じ少数の原則が、層を変えて何度も再演される**という構造の認識である。Firecracker と S3 と DynamoDB と IAM ポリシー言語は、まったく別の製品でありながら「機能の削除が能力を買う」という同一の判断を含む。Aurora と outbox パターンと CDC は「ログこそデータベース」の三つの顔である。

この反復は偶然ではない。クラウドの全サービスは同じ制約系——マルチテナントの相互不信、従量課金の経済、光速と故障の物理、組織のガバナンス——の下で設計されており、同じ制約は同じ解を強制する。ゆえに原則を先に掴めば、初見のサービスも「どの制約に、どの原則で答えたか」として読める——**サービス名を暗記する学習から、設計判断を導出する学習へ**の転換が、本ドキュメントの目的である。方法は帰納である：各原則について、独立な文脈で 3 回以上出現した証拠を本領域の分析から引き、外部の古典 (Lampson の設計ヒント、Well-Architected Framework) と突き合わせて位置づける。分析深度は Level 3 (ただし対象は個別システムではなく、本領域自身の分析結果である)。

## 方法論 (Methodology)

- **三度現れたらパターン**: 原則として採用する基準は、本領域の異なるドキュメント・異なる技術層で 3 回以上、互いに独立に出現していること。単一システムの工夫は原則ではない。
- 各原則を「どの制約への応答か」で根拠づける——原則は美学ではなく、制約からの演繹であることを示す。
- 外部参照との突き合わせ: システム設計の古典 (Lampson 1983) と、ベンダー自身の体系化 (AWS Well-Architected Framework——**ベンダー提示**の規範であり、本領域の帰納結果との一致・不一致を検証する対象として扱う)。
- 本ドキュメントは新しい事実を導入しない。すべての主張の証拠は、リンク先の各ドキュメントが一次資料に基づいて確立済みである。

## 発見事項 (Findings) — 七つの原則

1. **相関の破壊 (decorrelation)。** 大規模障害の実体は「多数が同時に・同じ理由で」壊れることであり、対策は空間・時間・因果の各軸で相関を切ること。空間: セル・シャッフルシャーディング・AZ/リージョン ([`cloud-control-planes.md`](cloud-control-planes.md), [`multi-region-global-architecture.md`](multi-region-global-architecture.md))。時間: ジッタ付きバックオフ・一定作業。経済: スポットのプール分散 ([`spot-capacity-economics.md`](spot-capacity-economics.md))、FaaS の配置の相関回避 ([`serverless-faas-internals.md`](serverless-faas-internals.md))。**制約**: 統計多重化 (共有による経済) は相関故障を構造的に生むため、共有したまま相関だけを割る技術が必要になる。攻撃に対しても同じ算術が働く (シャッフルシャーディングの DDoS 吸収)——障害と攻撃は「相関した負荷」として同型である。

2. **変更する系と動き続ける系の分離 (control/data plane separation) と静的安定性。** データプレーンはコントロールプレーンの停止時に最後の既知状態で走り続け、障害のさなかに新しいことをしない。出現: EBS のチェーンと Physalia ([`cloud-block-storage.md`](cloud-block-storage.md))、kubelet の自立動作 ([`container-orchestration-kubernetes.md`](container-orchestration-kubernetes.md))、Route 53 のデータプレーンと ARC ([`multi-region-global-architecture.md`](multi-region-global-architecture.md))、レイクハウスのカタログとファイル ([`data-lakehouse-analytics.md`](data-lakehouse-analytics.md))、IAM の結果整合な評価ホットパス ([`cloud-iam-authorization.md`](cloud-iam-authorization.md))。**制約**: 変更系は複雑で低頻度、実行系は単純で高頻度——可用性要求が桁で違う 2 つを混ぜると、低い方に揃う。2021-12 us-east-1 障害は、データプレーンの静的安定性が世界規模で機能した実証であり、同時に「コントロールプレーンの隠れ結合」が残された宿題であることも示した。

3. **ログこそデータベース (the log is the database)。** 変更の事実 (ログ) を一級の記録とし、状態はその投影として導出する。出現: Aurora の redo ログ複製 ([`cloud-databases-aurora-dynamodb.md`](cloud-databases-aurora-dynamodb.md))、CDC → レイクハウスの増分コミット ([`data-lakehouse-analytics.md`](data-lakehouse-analytics.md))、outbox パターンとイベントソーシング ([`event-driven-architecture.md`](event-driven-architecture.md))、テレメトリの「構造化イベントから安い集約を導出する」([`cloud-observability.md`](cloud-observability.md))。**制約**: 分散した複製・キャッシュ・派生物を整合させる最も単純な方法は、「事実の全順序列を一つ作り、全員がそれを追う」ことである——WAL ([`../databases/storage-engines.md`](../databases/storage-engines.md)) の理論的地位が、アーキテクチャの全層で物理化された。

4. **計算と記憶の分離 (compute-storage disaggregation)。** 状態を実行体から外し、実行体を使い捨てにする。出現: EBS (インスタンスと独立に生きるボリューム, [`cloud-block-storage.md`](cloud-block-storage.md))、Aurora (共有ストレージと使い捨てレプリカ)、レイクハウス (S3 と交換可能なエンジン)、FaaS の状態外部化 ([`serverless-faas-internals.md`](serverless-faas-internals.md))、スポット適合の第一条件 ([`spot-capacity-economics.md`](spot-capacity-economics.md))。**制約**: 故障・回収・スケールは実行体の側で起きる——状態が実行体に癒着していると、故障が喪失に、回収が不可能に、スケールがコピーになる。分離の主目的は層で異なる (弾力・可搬性・運用モデル) が、機構は同じである。イミュータブルインフラ・「実行していない時間は存在しない」という FaaS の課金も、この分離の帰結である。

5. **機能の削除が能力を買う (constraint as a feature)。** 表現力・自由度を意図的に捨てることで、スケール・安全・予測可能性・検証可能性を得る。出現: S3 の貧しい API ([`../object-storage/object-storage-architecture.md`](../object-storage/object-storage-architecture.md))、Firecracker の最小デバイスモデル ([`serverless-faas-internals.md`](serverless-faas-internals.md))、DynamoDB の有界リクエスト ([`cloud-databases-aurora-dynamodb.md`](cloud-databases-aurora-dynamodb.md))、IAM ポリシー言語の非チューリング完全性が SMT 検証を可能にする ([`cloud-iam-authorization.md`](cloud-iam-authorization.md))、Nitro Enclaves の無機能性 ([`cloud-kms-confidential-computing.md`](cloud-kms-confidential-computing.md))、ワンショット限定のトランザクション。**制約**: 保証 (性能・隔離・解析) は入力空間の全体に対して与えねばならず、入力空間を縮めることが保証を強くする唯一の一般解である——「何が書けるか」ではなく「何が保証できるか」でシステムを測る、という価値転換がこの原則の核心である。

6. **暗黙の文脈の明示化と機械検証 (explicit context, machine verification)。** 事故は「暗黙の前提」——誰の代理か、どこまで許すつもりだったか、何に束縛された暗号文か——が破られる場所で起き、対策は前提を偽造不能な明示的文脈に変え、人間ではなく機械に検証させること。出現: IMDSv2・ExternalId・SourceArn ([`cloud-iam-authorization.md`](cloud-iam-authorization.md))、encryption context と attestation ([`cloud-kms-confidential-computing.md`](cloud-kms-confidential-computing.md))、Zelkova/Access Analyzer・policy as code ([`infrastructure-as-code.md`](infrastructure-as-code.md))・到達可能性解析 ([`cloud-network-topology.md`](cloud-network-topology.md))、イベントの由来検証 ([`event-driven-architecture.md`](event-driven-architecture.md))。**制約**: 規模は人間のレビューをスケールさせない——設定の組合せ空間は指数的に増え、誤設定が最大の侵害原因であり続ける。ガードレール (SCP・boundary・公開ブロック) は「誤っても越えられない床」としてこの原則の実行時版になる。

7. **難しさは消えず、移動する (conservation of difficulty)。** アーキテクチャの選択とは、複雑さ・不確実さ・調整コストを消すことではなく、**引き受ける場所を選ぶ**ことである。出現: Dynamo (アプリが競合解決) → DynamoDB (スキーマ設計が負担) の移転、Aurora (互換性の裏でストレージ層が難しさを吸収)、saga の中間状態 (原子性の喪失を語彙で引き受ける, [`event-driven-architecture.md`](event-driven-architecture.md))、競合解決の 4 択 (どれも痛く、痛む場所が違う, [`multi-region-global-architecture.md`](multi-region-global-architecture.md))、スポットの中断設計 (割引の対価は利用者の設計, [`spot-capacity-economics.md`](spot-capacity-economics.md))、抽象度スペクトル EC2→Lambda ([`container-orchestration-kubernetes.md`](container-orchestration-kubernetes.md))。**制約**: 分散・並行・故障は問題の本質的複雑さであり、抽象化は複雑さを隠蔽するのではなく再配置する。「どこが痛くなるか」を選択の時点で言語化することが、アーキテクチャレビューの実体である。

## アーキテクチャ (Architecture)

### 原則 × ドキュメントの出現マップ

| 原則 \ 文書 | serverless | control-planes | IAM | databases | block-storage | K8s | multi-region | KMS/機密 | spot | IaC | lakehouse | event-driven | network | observability |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 相関の破壊 | ● | ●● | | | ● | | ●● | | ●● | | | ● | ● | ● |
| 2 制御/実行分離・静的安定 | ● | ●● | ● | ● | ●● | ●● | ●● | | | ● | ● | | | ● |
| 3 ログこそ DB | | | | ●● | | | ● | | | | ●● | ●● | | ● |
| 4 計算と記憶の分離 | ● | | | ●● | ●● | ● | | | ● | | ●● | | | |
| 5 機能の削除 | ●● | | ●● | ●● | | | | ●● | | | | ● | ● | |
| 6 明示化と機械検証 | | ● | ●● | | | ● | | ●● | | ●● | | ● | ●● | ● |
| 7 難しさの移動 | ● | ● | | ●● | ● | ●● | ●● | | ●● | ● | ● | ●● | | |

(●● = 中心テーゼ、● = 主要な出現。空欄は「無関係」ではなく「主題でない」)

### 学習経路 — AWS を原則で読む順序

```
 ① 土台 (機構):        virtualization → object-storage → distributed-systems
                        └ 既存スタックが与える語彙 (VM/コンテナ, 耐久性, 合意/一貫性)
 ② 経済と制御 (原則):   serverless-faas → cloud-control-planes → spot-capacity
                        └ 原則 1,2,4 と「経済が設計を規定する」視点
 ③ 権限と信頼:          cloud-iam → cloud-kms-confidential → cloud-network-topology
                        └ 原則 5,6 と「二重の柵」(認可 × 到達性)
 ④ データ:             cloud-databases → cloud-block-storage → data-lakehouse
                        └ 原則 3,4,7 と「契約からの演繹」
 ⑤ 組み立て:           container-orchestration → event-driven → IaC → multi-region
                        └ 調停ループの連続体と「難しさの置き場所」の選択
 ⑥ 閉ループ:           cloud-observability → (本書)
```

## 技術詳細 (Technical Details) — 原則の使い方

### 1. 設計レビューの質問群への変換

原則は暗記ではなく質問として使う。新しい設計・初見のサービスに対して：(1) **相関**: この構成で「全部が同時に壊れる」経路はどこか (共有依存・同期した再試行・単一プール)。(2) **分離**: 制御系が止まったとき、実行系は何分動き続けるか。回復経路に制御系依存はないか。(3) **ログ**: 状態の二重書き込みはないか——事実の一次記録はどこか。(4) **状態**: この実行体は今すぐ回収されてよいか。だめなら、その状態はなぜ外部化できないか。(5) **制約**: この API/言語は何を書けなくしているか——それが保証しているものは何か。(6) **検証**: この設定の正しさは誰が確認するか——人間のレビューなら、機械検査に落とせないか。(7) **移動**: この選択で難しさはどこへ行ったか——そこにいるチームはそれを知っているか。7 問に答えられない箇所が、設計の未決定部分である。

### 2. Well-Architected との対応と差分

AWS Well-Architected Framework (ベンダー提示) の 6 本の柱 (運用・セキュリティ・信頼性・性能効率・コスト最適化・持続可能性) は、本書の原則群と大部分が対応する——信頼性の柱は原則 1,2 の、セキュリティの柱は原則 6 の、コスト最適化は [`spot-capacity-economics.md`](spot-capacity-economics.md) の経済観の規範化である。差分が 2 つある。(a) Well-Architected は**チェックリスト (何をすべきか)** であり、本書は**導出 (なぜそうなるか)** を目的にする——前者は監査に、後者は初見の問題への転移に向く。(b) 本書の原則 7 (難しさの移動) に対応する柱は存在しない——ベンダーの規範は「正しい選択」を提示する形式を取り、「どの選択も何かを引き受ける」というトレードオフの保存則は、利用者側が持ち込むべき視点として残る。両者は代替ではなく、導出 (本書) で理解し、チェックリスト (Well-Architected) で網羅を確認する補完関係が実用的である。

### 3. 原則の衝突と調停

原則は互いに衝突し、調停こそが設計である。代表例：**静的安定性 (動き続けろ) vs fail-closed (検証できなければ拒否せよ)** ——IAM は「評価データをホットパスに複製し、失効は期限で保証する」という妥協で両立させた ([`cloud-iam-authorization.md`](cloud-iam-authorization.md))。**不変性 (履歴を保て) vs 消去義務 (忘れろ)** ——レイクハウスのスナップショット失効設計 ([`data-lakehouse-analytics.md`](data-lakehouse-analytics.md))。**隔離 (分けろ) vs 統計多重化 (混ぜて稼げ)** ——この緊張は Physalia のセル、DynamoDB の GAC、スポットの存在自体を生んだ、本領域で最も生産的な衝突である。衝突に気づいたとき、「どちらが正しいか」ではなく「この文脈ではどの妥協点か、その妥協は明示されているか」を問うのが原則の正しい使い方である。

### 4. 原則が予測すること — 転移の練習

原則の価値は未知への転移で測られる。例題として「初見のマネージド サービス X を評価する」手順：X の課金単位を見る (経済が設計を規定する——課金単位は内部の資源の単位を映す)。X の API の「できないこと」を列挙する (機能の削除——そこに X の保証の源泉がある)。X のコントロールプレーン操作とデータプレーン操作を分類する (分離——障害時に何が動き続けるかの予測)。X のマルチテナント境界を推定する (隔離の等級——microVM か、アカウントか、論理か)。この手順は本領域の各ドキュメントが実際に辿った分析手順の一般化であり、新サービスの発表資料を「マーケティングの層」と「制約と原則の層」に分けて読む訓練でもある。

## セキュリティ考察 (Security Considerations)

セキュリティ面の総括は、三つの反復に集約される。

- **独立な柵の積 (defense as independent failures)**: 単一の防御の完璧さではなく、失敗モードが独立な複数の柵——認可と到達性 ([`cloud-network-topology.md`](cloud-network-topology.md))、暗号と権限 ([`cloud-kms-confidential-computing.md`](cloud-kms-confidential-computing.md))、plan 検査と実行時ガードレール ([`infrastructure-as-code.md`](infrastructure-as-code.md))——の積で守る。[`../../foundations/mathematics/probability-statistics/reliability-theory.md`](../../foundations/mathematics/probability-statistics/reliability-theory.md) の並列合成が、そのまま防御設計の数理である。Capital One 型の事故は「柵が独立でなかった」ことの実証として、本領域で繰り返し参照された。
- **爆発半径は障害と侵害の共通言語**: セル・アカウント境界・最小権限・関数単位ロール・鍵の分離は、障害の伝播と攻撃の水平移動の両方を縛る同じ機構である。「この資格情報/このセル/この鍵が失われたら、何が失われるか」という問いは、可用性設計とセキュリティ設計を同じ図面の上に載せる。
- **保証の等級を混同しない**: 数学的保証 (暗号・SMT 検証)、設計的保証 (隔離機構・fail-closed)、組織的保証 (運用統制・監査) は等級が異なり、「TEE だから」「プライベートだから」「検証済みだから」という等級の混同が過信を生む ([`cloud-kms-confidential-computing.md`](cloud-kms-confidential-computing.md) の結論)。何を・誰から・どの等級で守っているかの明示が、セキュリティ主張の最低要件である。

## 結論 (Conclusions)

- クラウドの設計パターンは少数であり、再利用される。本領域 14 本の分析から帰納された七つの原則——相関の破壊、制御/実行の分離、ログこそデータベース、計算と記憶の分離、機能の削除、文脈の明示化と機械検証、難しさの移動——は、それぞれ 3 つ以上の独立な層で確認された。サービスは変わり続けるが、制約 (相互不信・従量課金・光速・故障・ガバナンス) が変わらない限り、原則は残る。
- 学習の単位はサービスではなく原則であるべきである。「S3 とは何か」より「なぜ S3 の API は貧しいのか」を問う学習は、初見のサービス・他社クラウド・次の 10 年の製品にも転移する——本領域の各ドキュメントが「契約からの演繹」という形式を取ったのは、この転移可能性のためである。
- 原則は衝突する。静的安定と fail-closed、不変性と消去、隔離と多重化——設計とは原則の適用ではなく調停であり、良い設計文書とは「どの緊張を、どの妥協点で、なぜ解いたか」が明示されたものである。この基準は、本リポジトリ自身の今後の分析文書にも適用される。
- 残された地平も原則から見える：GPU/AI 基盤は「余剰なき世界の容量経済」(原則の適用条件の変化) を、機密計算は「事業者を信頼から外す」(保証等級の引き上げ) を、エージェント型ワークロードは「認可の主語のさらなる拡張」を試している——次に何を分析すべきかの指針も、原則の延長線上にある。

### 次の接続先 (Related Topics)

- 本領域の全ドキュメント — [`README.md`](README.md) の索引と、上記の学習経路を参照。
- [`../../software/README.md`](../README.md) — software スタック全体における本領域の位置 (第四のプラットフォーム)。
- [`../distributed-systems/`](../distributed-systems/) / [`../virtualization/`](../virtualization/) / [`../object-storage/`](../object-storage/) — 原則群の機構的土台を与えた既存分析。

## 参考文献 (References)

**古典 (Classics — 設計原則の先行定式化)**

- Lampson, B. W. (1983). Hints for Computer System Design. *ACM SOSP '83*, 33–48. DOI: 10.1145/800217.806614 — システム設計原則の帰納という形式の原点。「単純にせよ」「速い経路を作れ」等、本書の原則 2,5 の祖形を含む。
- Saltzer, J. H., & Schroeder, M. D. (1975). The Protection of Information in Computer Systems. *Proceedings of the IEEE*, 63(9), 1278–1308. DOI: 10.1109/PROC.1975.9939 — 最小権限・fail-safe defaults・完全仲介。本書のセキュリティ考察の三反復の理論的先祖。

**ベンダー提示 (Vendor-presented — 規範化されたベストプラクティス)**

- AWS. *Well-Architected Framework*. [docs.aws.amazon.com/wellarchitected/](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) — Accessed: 2026-07-30。本書の帰納結果と突き合わせる対象としてのベンダー規範 (技術詳細 2)。

**一次資料**

- 各原則の証拠となる一次資料 (NSDI/ATC/SIGMOD/VLDB/OSDI/HotOS/FMCAD 等の査読付き論文・標準・公式ポストモーテム) は、リンク先の各ドキュメントの参考文献に記載済みであり、本書はそれらを再引用しない。
