# Infrastructure as Code (IaC: CloudFormation, Terraform & 宣言的状態管理)

## 概要 (Overview)

**Infrastructure as Code (IaC)** は「インフラをコードで書く」道具として説明されるが、テンプレート記法は本質ではない。本質は、[`container-orchestration-kubernetes.md`](container-orchestration-kubernetes.md) で見た**宣言的状態と調停 (reconciliation)** をクラウドリソース全般へ適用しようとする試みである。ただし決定的な違いが一つある。Kubernetes は常時調停 (現在状態との差分を検出し次第、自動で埋める) だが、CloudFormation / Terraform は**計画実行型** (plan/apply)：差分を実行計画として人間に提示し、人間が引き金を引いて初めて収束する。

この違いは怠慢ではなく必然である。理由は 2 つ。(a) 対象であるクラウド API は非同期・非トランザクショナルで部分失敗し、リソース作成には分単位の時間がかかる。そのため、高速な調停ループの前提が成り立たない。(b) **誤収束のコストが桁違いに高い**。Pod の再作成は無害だが、「データベースの再作成」はデータ消失である。自動で埋めてよい差分と、人間の確認が要る差分の区別を機械は知らない。ゆえに IaC の中心問題は、**意図 (コード)・記録 (state)・現実 (API 上の実リソース) という三つの状態の突合**になり、本ドキュメントが扱う drift、実行計画、置換 (replacement)、部分失敗、依存グラフは、すべてこの三体問題からの演繹である。CloudFormation と Terraform を対照させながら機構を追い、GitOps による常時調停への回帰、policy as code によるガバナンス執行点化までを扱う。分析深度は Level 3。

## 方法論 (Methodology)

- IaC を「意図・記録・現実の三状態と、それを突合する計画コンパイラ」というモデルに還元し、各ツールの機能はモデル上の設計選択として比較する。
- 調停ループの一般論は [`container-orchestration-kubernetes.md`](container-orchestration-kubernetes.md)、収束型構成管理の原点は CFEngine の論文 (Burgess 1995) に置き、IaC を「その系譜がクラウド API に出会って何を変えたか」として読む。
- グラフの語彙 (DAG・トポロジカル順序) は [`../../foundations/mathematics/discrete-logic/graphs-trees-algorithms.md`](../../foundations/mathematics/discrete-logic/graphs-trees-algorithms.md) に依拠する。
- ツール仕様は公式文書 (HashiCorp / AWS の**ベンダー一次資料**) に、体系的な整理は教科書 (Morris, *Infrastructure as Code*) に依る。

## 発見事項 (Findings)

1. **IaC の本体はテンプレートではなく差分計画のコンパイラである。** apply の実行前に起きているのは、(i) 現実の読み取り (refresh: API から実リソースの現状を取得)、(ii) 意図との差分計算、(iii) 差分を Create/Update/Delete/**Replace** の操作列へ変換、(iv) 依存 DAG による順序付けである。つまり**宣言 (何であるべきか) から命令 (何をするか) へのコンパイル**であり、plan / change set はその中間表現である。この視点に立つと、IaC のレビューとは「テンプレートのレビュー」ではなく「**コンパイル結果 (計画) のレビュー**」だと分かる。同じテンプレート変更でも、現実の状態次第で in-place 更新にも全置換にもコンパイルされるからである。

2. **Terraform と CloudFormation の対立軸は「記録 (state) を誰が持つか」である。** Terraform は state file という記録を**顧客の管理物** (S3 バケット等) として明示的に持たせる。見える・触れる・壊せる。CloudFormation は記録をスタックとして**サービス側**に持つ。見えない・触れない・管理不要。Terraform の運用問題群 (state の破損・ロック競合・秘密の混入・state surgery) と、CloudFormation の問題群 (サービスの知らない現実との乖離、記録の外からの修復の困難) は、**同じ三体問題のどちら側に複雑さを置いたか**の差である。どちらにも「記録と現実の乖離」は残る。記録の置き場所は乖離を消さない。

3. **変更の意味論で最重要な区別は in-place 更新と置換 (replacement) である。** 各リソースの属性には「作成後に変更可能」と「変更不能 (immutable)」があり、不能な属性の変更は「**削除して作り直す**」操作へコンパイルされる。RDS のインスタンス識別子の変更、サブネットの CIDR 変更のように、テンプレート上は 1 行の差分が、実行では**データを持つリソースの破壊**になりうる。IaC 最悪の事故類型はほぼすべてこの機構であり、防御は (a) plan における置換の検出を review の第一項目にする、(b) 削除保護・作成先行 (create-before-destroy) の指定、(c) ステートフルなリソースをステートレスな層と別スタックに分離し、変更頻度と爆発半径を揃えることである。これは [`cloud-control-planes.md`](cloud-control-planes.md) のセル分割と同じ「変更のブラストラディウス管理」である。

4. **Drift は「常時調停しないことを選んだ系」の宿命である。** apply の時点でしか調停しない以上、コンソールでの手作業・他の自動化・障害対応の緊急変更は、次の apply まで記録されない乖離 (drift) として蓄積する。drift 検出機能は診断であって治療ではない。治療の選択肢は「現実をコードに合わせて上書きする」「コードを現実に合わせて更新する (import/採録)」「乖離の発生源を権限で塞ぐ」の 3 つで、根治は 3 番目 (人間の書き込み権限を読み取り専用へ落とし、変更経路を IaC パイプラインに一本化する) しかない。GitOps (Flux/Argo 型) は Git を意図の唯一の置き場にして**常時調停へ回帰する**中間解だが、それが主に Kubernetes 上で成功しているのは、K8s のリソースが「誤収束が安い」(発見事項の冒頭の条件を満たす) からである。常時調停の適用範囲は、対象の再作成コストが決める。

5. **クラウド API はトランザクションを提供しないので、IaC は補償か停止かを選ぶ。** 10 リソースの変更中に 7 個目で失敗したとき、原子性は存在しない。CloudFormation は**自動ロールバック** (成功済み操作の逆操作を実行する saga 型補償) を選び、Terraform は**停止** (部分適用のまま止まり、人間が再 apply で前進回復) を選んだ。補償には「補償不能な操作」という破綻がある。削除してしまったものは戻せず、ロールバック自体が失敗する状態 (UPDATE_ROLLBACK_FAILED) は CloudFormation 運用の名物である。ここにあるのは分散トランザクションの古典的教訓 ([`../databases/transactions-isolation.md`](../databases/transactions-isolation.md) の原子性がいかに高価か) であり、実務の答えは「スタックを小さく分割し、1 回の apply のブラストラディウスを縛る」ことであり、ここでもまたセル分割である。

6. **依存グラフが並列度と削除順序の両方を規定する。** リソース間の参照 (サブネット ID を EC2 が使う等) は暗黙の依存辺になり、実行計画は DAG のトポロジカル順序で走る。独立な部分木は並列に、依存するものは直列に実行される。削除は**逆順**であり、「依存されているものは先に消せない」。循環参照はコンパイルエラーである (現実には作れない構造だから正しい)。この DAG は [`../../foundations/mathematics/discrete-logic/graphs-trees-algorithms.md`](../../foundations/mathematics/discrete-logic/graphs-trees-algorithms.md) の教科書的応用であると同時に、設計の写像でもある。グラフが密で深いスタックは、変更の影響範囲が読めない設計の症状として現れる。

7. **IaC の最大の副産物はガバナンスの執行点である。** インフラ変更が「コードの差分」になった時点で、ソフトウェア開発の全装備がインフラに輸入される。レビュー、差分の監査証跡、ロールバック (revert)、CI での検査である。**Policy as code** (OPA/Rego, CloudFormation Guard) は plan 段階で「公開 S3 バケットを作る計画は拒否」「暗号化なしのボリュームは拒否」を機械検査する。これは [`cloud-iam-authorization.md`](cloud-iam-authorization.md) の Zelkova と同じ「人間が読むのではなく機械が検証する」路線の、デプロイ前 (shift-left) 版である。実行時ガードレール (SCP・AWS Config) との関係は多層防御であり、「plan で拒否・実行時にも拒否」の二層が、どちらか一方の抜け (パイプライン外の変更、検査の穴) を補う。

## アーキテクチャ (Architecture)

### 三体問題 — 意図・記録・現実

```
   意図 (desired)              記録 (recorded state)          現実 (actual)
  ┌──────────────┐   diff    ┌──────────────────┐  refresh  ┌──────────────┐
  │ コード (Git)   │◀────────▶│ TF: state file    │◀────────▶│ クラウド API   │
  │ HCL/テンプレート│          │ CFn: スタック状態  │           │ 上の実リソース │
  └──────┬───────┘          └──────────────────┘           └──────▲───────┘
         │      ① refresh: 現実を記録に反映                        │
         │      ② diff:    意図 − 記録 → 差分                     │
         │      ③ compile: 差分 → Create/Update/Delete/Replace    │
         │      ④ order:   依存 DAG のトポロジカル順序で計画化      │
         └────── ⑤ apply:  人間の承認後に実行 ─────────────────────┘

  drift = ①〜⑤ の合間に「現実」だけが動くこと (手作業・他の自動化)
  常時調停 (K8s/GitOps) は ①〜⑤ を高頻度自動化した特殊解 — 誤収束が安い対象に限る
```

### 設計空間の比較

| 軸 | Terraform | CloudFormation | Kubernetes/GitOps |
|---|---|---|---|
| 記録の所有 | 顧客 (state file) | サービス (スタック) | サービス (etcd) |
| 調停の引き金 | 人間 (plan→apply) | 人間 (change set→実行) | 常時 (自動) |
| 部分失敗の意味論 | 停止・前進回復 | 自動ロールバック (saga) | 再試行し続ける |
| 対象 | 任意プロバイダ | AWS (+カスタムリソース) | クラスタ内 + operator の及ぶ範囲 |
| 誤収束コストの想定 | 高い (確認必須) | 高い (確認必須) | 低い (再作成が安い前提) |

## 技術詳細 (Technical Details)

### 1. 計画は約束ではない — refresh と TOCTOU

plan は「その時点の記録と現実」に基づくコンパイル結果であり、apply までの間に現実が動けば結果は変わる。つまり、check-then-act の古典的な TOCTOU (time-of-check to time-of-use) 競合が、インフラ規模で存在する。緩和は (a) plan の成果物 (実行計画ファイル) を apply に渡し、乖離があれば apply を失敗させる、(b) state のロック (Terraform はロック機構にクラウド側のロックテーブル等を使う。IaC 自身が [`cloud-databases-aurora-dynamodb.md`](cloud-databases-aurora-dynamodb.md) の条件付き書き込みの利用者になる)、(c) 変更経路の一本化 (並行 apply の禁止)。「plan で見たものが実行される」は保証ではなく、パイプライン設計で**作る**性質である。

### 2. State file の力学と秘密

Terraform の state には、意図と現実の対応 (リソースのアドレス ↔ 実 ID) に加えて、**API が返した属性値がそのまま**入る。DB の初期パスワード、生成された秘密鍵など、秘密が平文で記録されうる。ゆえに state は「設定ファイル」ではなく**機密データベース**として扱う：保存時暗号化・アクセス制御・バージョニングされたリモートバックエンド、そして「state を読める者はインフラの全構成と一部の秘密を読める」という権限設計上の認識である。state surgery (import / moved / removed) は記録と現実の対応の手術であり、「現実を変えずに記録だけ直す」操作群が第一級の機能として存在すること自体が、三体問題が本質であることの証左である。

### 3. CloudFormation の実行モデルと拡張

change set は plan に相当する差し替え計画で、スタックポリシーが「このリソースは更新禁止」という追加の柵を与える。カスタムリソースは「API が存在しない対象」を Lambda フック ([`serverless-faas-internals.md`](serverless-faas-internals.md)) で CRUD 化する拡張点であり、IaC のカバレッジがプロバイダ API の網羅性で決まるという制約への脱出口である (Terraform では provider プラグインが同じ役割)。StackSets は同一テンプレートのマルチアカウント・マルチリージョン展開である。[`multi-region-global-architecture.md`](multi-region-global-architecture.md) の文脈では、「リージョン独立性を保ったまま構成だけを揃える」ための道具であり、展開順序・失敗許容度 (何リージョン失敗したら止めるか) という**デプロイのブラストラディウス制御**がパラメータ化されている。

### 4. 高級言語化 — CDK/Pulumi と表現力の振り子

CDK / Pulumi は汎用言語 (TypeScript/Python) で意図を**生成**する。ループ・抽象化・型検査・パッケージ共有という言語の力を得て、出力 (合成されたテンプレート) を従来のエンジンに渡す。ここで起きているのは [`cloud-iam-authorization.md`](cloud-iam-authorization.md) の Cedar と**逆向きの振り子**である：認可言語は解析可能性のために表現力を削ったが、IaC は表現力のためにチューリング完全へ動いた。代償は解析可能性である。「このコードが何を作るか」は実行 (synth) するまで分からず、policy as code の検査は生成後のテンプレート/計画に対して行うしかない。実務の均衡点は「生成は自由に、検査は生成物に対して機械的に」という二段構えであり、**生成の表現力と検査対象の単純さを両立させる**ためにこそ、中間表現 (テンプレート・計画) の存在が意味を持っている。

### 5. パイプラインとテストの階層

インフラのテストが難しい根本理由は「外部世界がモックできない」ことにある。IAM の評価も VPC の経路も、本物の API の挙動が仕様だからである。実務の階層は：(i) **静的検査** (構文・型・policy as code で、速く・網羅的・浅い)、(ii) **plan アサーション** (「この変更で置換が発生しない」「作られるリソースは N 個」など、実 API なしで意味論の一部を検査)、(iii) **エフェメラル環境** (使い捨てアカウント/スタックに実際に apply する統合検証で、遅く・高価・深い)。この階層は通常のテストピラミッドと同型だが、(iii) のコストが高いぶん (ii) の充実が投資対効果の要になる。デプロイ自体は [`cloud-control-planes.md`](cloud-control-planes.md) の原則に従う。つまり、環境の段階的展開 (dev→staging→本番の wave)、自動ロールバックの条件、変更のブラストラディウスを一度に広げないこと、である。

## セキュリティ考察 (Security Considerations)

- **IaC パイプラインはインフラ全体の管理権限の集中点である。** apply を実行する主体は定義上、対象環境の作成・削除権限を持つ。そのため、パイプラインの侵害は全環境の掌握である。防御は [`cloud-iam-authorization.md`](cloud-iam-authorization.md) の道具立てそのもの：長命キーを排した OIDC フェデレーション (CI がリポジトリ・ブランチ束縛の短命トークンで AssumeRole)、環境ごとのロール分離、permissions boundary による「パイプラインが作れる権限の上限」。そして [`../security/software-supply-chain-security.md`](../security/software-supply-chain-security.md) の主戦場でもある。サードパーティのモジュール・プロバイダ・アクションは、apply の権限で任意コードを実行できる依存である。
- **State と計画成果物は機密である。** state file の秘密 (技術詳細 2) に加え、plan の出力にも属性値が現れる。CI のログに plan を素朴に出すことは、秘密のログ流出と同じ類型になる。
- **「手作業の禁止」は権限で執行して初めて意味を持つ。** ドリフト対策としての運用規律 (コンソール変更禁止) は、書き込み権限が残っている限り破られる。読み取り専用の人間 + 書き込みはパイプラインのロールのみ、緊急時の break-glass 手順 (時間制限付き昇格 + 事後監査) という権限設計が、規律を機構に変える。副産物として、**drift 検出が侵害検知になる**。正規の変更経路が一本なら、経路外の変更 (drift) は誤操作か攻撃のどちらかであり、[`cloud-control-planes.md`](cloud-control-planes.md) の「調停ループは改竄検知器」がここでも成り立つ。
- **Policy as code は「作る前に拒否する」層である。** 実行時ガードレール (SCP・Config ルール) だけでは「作られてから消すまでの窓」が残る。plan 段階の検査は窓そのものを消す。ただしパイプライン外の変更には無力なので、両層は代替ではなく直列である。検査ルール自体もコードとしてレビュー・バージョン管理される。「ガバナンスのガバナンス」も Git に載る、というのが IaC の到達点である。

## 結論 (Conclusions)

- IaC は調停ループの**人間律速版**である。その主要な難点である state、drift、置換事故、部分失敗は、すべて「常時調停しない」という一つの設計選択から演繹され、その選択自体は「クラウド API の性質」と「誤収束の高コスト」という正当な理由を持つ。K8s/GitOps の常時調停と IaC の計画実行は優劣ではなく、**対象の再作成コストで使い分ける連続体**である。
- 実務スキルの核心は記法ではなく計画の読解であり、とくに置換 (replacement) の検出である。そしてアーキテクチャスキルの核心はスタック分割である。変更頻度と爆発半径を揃え、1 回の apply が壊せる範囲を縛る。[`cloud-control-planes.md`](cloud-control-planes.md) のセル論は、インフラ定義の構造にまで適用される。
- IaC の歴史的意義はガバナンスの執行点を作ったことにある。レビュー・監査・policy as code・OIDC 化されたパイプラインにより、「インフラ変更」という最高権限の操作が、ソフトウェア工学の検証装置一式の下に入った。本領域の通奏低音 (機械検証・ガードレール・文脈の明示化) が、インフラの変更管理そのものに適用された姿である。
- 表現力 (CDK) と解析可能性 (policy 検査) の緊張は、中間表現 (テンプレート・計画) を挟むことで両立されている。「自由に生成し、生成物を機械検査する」という形である。この構図は認可言語 (Cedar) が「最初から書ける範囲を絞る」のと対照的であり、失敗のコスト (誤設定の実害が出るまでの距離) の違いが、どちらの均衡点を選ぶかを決めている。

### 次の接続先 (Related Topics)

- [`container-orchestration-kubernetes.md`](container-orchestration-kubernetes.md) — 常時調停側の実装。spec/status・informer・楽観並行制御。
- [`cloud-control-planes.md`](cloud-control-planes.md) — 変更のブラストラディウス・段階的展開・調停ループの一般論。
- [`cloud-iam-authorization.md`](cloud-iam-authorization.md) — パイプラインの OIDC 化・permissions boundary・機械検証の路線。
- [`../security/software-supply-chain-security.md`](../security/software-supply-chain-security.md) — モジュール・プロバイダ・CI という依存の信頼問題。
- [`../../foundations/mathematics/discrete-logic/graphs-trees-algorithms.md`](../../foundations/mathematics/discrete-logic/graphs-trees-algorithms.md) — 依存 DAG とトポロジカル順序の数理。

## 参考文献 (References)

**一次資料 (Primary — 論文・原典)**

- Burgess, M. (1995). Cfengine: a site configuration engine. *USENIX Computing Systems*, 8(2). — 収束型構成管理 (convergent configuration) の原点。「望ましい状態へ繰り返し近づける」思想の原典。
- Burns, B., Grant, B., Oppenheimer, D., Brewer, E., & Wilkes, J. (2016). Borg, Omega, and Kubernetes. *Communications of the ACM*, 59(5). DOI: 10.1145/2890784 — 宣言的 API と調停への収束の系譜 (詳細は [`container-orchestration-kubernetes.md`](container-orchestration-kubernetes.md))。

**ベンダー一次資料 (Vendor — ツール仕様の規範記述)**

- HashiCorp. *Terraform Documentation*: State; Plan and Apply; Resource Behavior (replacement). [developer.hashicorp.com/terraform/docs](https://developer.hashicorp.com/terraform/docs) — Accessed: 2026-07-30。
- AWS. *AWS CloudFormation User Guide*: Change sets; Stack failure options and rollback; StackSets; Custom resources. [docs.aws.amazon.com/cloudformation/](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) — Accessed: 2026-07-30。
- Open Policy Agent 文書 / AWS CloudFormation Guard. [openpolicyagent.org/docs/](https://www.openpolicyagent.org/docs/latest/) — Accessed: 2026-07-30。policy as code の実装。

**二次資料 (Secondary)**

- Morris, K. (2020). *Infrastructure as Code: Dynamic Systems for the Cloud Age* (2nd ed.). O'Reilly. — スタック分割・パイプライン・テスト階層の体系的整理。
