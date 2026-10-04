# クラウド IAM と認可 (Cloud IAM & Authorization: SigV4, Policy Evaluation & Formal Verification)

## 概要 (Overview)

クラウドでは、インスタンスの起動もオブジェクトの GET もポリシーの変更も、すべてが**認証・認可された API 呼び出し**である。物理的なネットワーク境界（社内 LAN にいるか）ではなく、**リクエスト 1 本ごとの署名とポリシー評価**が唯一の境界になる。クラウドの IAM (Identity and Access Management) は、ゼロトラストと呼ばれるものの最古参かつ最大規模の実装である。AWS IAM を走る事例に、(1) 署名 (SigV4) による**ステートレスな認証**、(2) 複数種のポリシーを合成する**評価論理**、(3) 長命の秘密を廃していく **STS と一時クレデンシャル**、(4) ポリシーの**形式検証** (Zelkova/Cedar)、(5) 構造的欠陥としての **confused deputy** を追う。

中心テーゼは 2 つ。第一に、IAM は**ホットパスに状態を置かない認可の分散システム**として設計されている。認証はセッション DB の照会ではなく HMAC 導出鍵による署名検証であり、認可はポリシー文書という宣言的データの評価である。第二に、**ポリシー言語の表現力を意図的に制限したことが形式検証を可能にした**。「この S3 バケットは公開されうるか」という問いが SMT ソルバの充足可能性判定に落ちるのは、言語がチューリング完全でないからである。表現力と解析可能性の交換は、[`../../foundations/mathematics/discrete-logic/automata-formal-languages.md`](../../foundations/mathematics/discrete-logic/automata-formal-languages.md) が扱った計算モデルの階層の、セキュリティ工学における最も実利的な応用例と言える。分析深度は Level 3。

## 方法論 (Methodology)

- トークン形式・OAuth 系の一般論は [`../apis/authentication-tokens-specification.md`](../apis/authentication-tokens-specification.md) が扱っており、本書は「クラウド事業者の API 認可」に固有の機構（署名・ポリシー合成・ロール・メタデータサービス）に集中する。
- 評価論理・SigV4 の規範記述は AWS 公式文書に依るほかない（**ベンダー一次資料**。自社仕様の記述としては最高位だが、設計意図の説明は宣伝を含みうる）。設計原理の側は査読付き論文 (Zelkova FMCAD '18, Zanzibar ATC '19, Cedar) と実事故の独立分析 (Capital One 事件の事例研究) で相互検証する。
- 比較対象として Google Zanzibar（関係ベース認可）を置き、「何を主語にするか」「どの一貫性を保証するか」の設計空間の広がりを示す。

## 発見事項 (Findings)

1. **認証はステートレスな署名検証として設計されている。ホットパスにセッションストアがない。** SigV4 のクライアントは、秘密鍵そのものではなく `kSigning = HMAC(HMAC(HMAC(HMAC("AWS4"+secret, 日付), リージョン), サービス), "aws4_request")` と導出した鍵でリクエストの正規形を署名する。サーバ側は同じ導出を再現して照合するだけで、セッション状態の照会が要らない。導出チェーンの各段はそのまま**鍵の爆発半径の限定**である：導出鍵が漏れても、その日付・そのリージョン・そのサービスにしか使えず、元の秘密は逆算できない (HMAC の一方向性)。タイムスタンプ（許容ずれ 15 分）とペイロードハッシュの署名内包により、リプレイと改竄も同じ機構で防ぐ。認可以前に、**毎秒膨大な API 呼び出しを裁く認証をどう水平スケールさせるか**という分散システムの問いに対する答えがこの設計である。

2. **ポリシー評価は「和で広げ、積で削り、拒否が全てに勝つ」。** 既定は暗黙の拒否 (implicit deny)。アイデンティティベースポリシーとリソースベースポリシーは**和集合**として許可を広げる（同一アカウント内はどちらか一方の allow で足りる）。SCP (組織の上限)・permissions boundary・セッションポリシーは**積集合**として上限を課す。これらに allow を与える力はなく、削る力しかない。そして **explicit deny はあらゆる allow に優先する**。この構造は「権限を付与する系」と「権限を制限する系」を代数的に分離しており、組織のガバナンス（積の側）を個々のチームの権限設計（和の側）から独立に運用できる。クロスアカウントでは両アカウントの allow が必要になり、和の規則が積へ切り替わる。信頼境界を跨ぐところで代数が変わる点は、事故の多発地帯でもある。

3. **一時クレデンシャルへの移行は「秘密の配布問題」を「トークン発行問題」に変換した。** 長命のアクセスキーは、配布・保管・ローテーション・失効のすべてが人間の運用に依存する。STS の AssumeRole は、検証可能な既存アイデンティティ（IAM ユーザ、EC2 インスタンス、OIDC でフェデレートされた外部 ID。Kubernetes の ServiceAccount や GitHub Actions まで含む）を**短命の資格情報に交換する**発行局であり、[`../apis/authentication-tokens-specification.md`](../apis/authentication-tokens-specification.md) が整理した「セッションの資格情報化」と同じ構図のインフラ版である。ロールは「誰が引き受けられるか (trust policy)」と「引き受けたら何ができるか (permission policy)」を分離した権限の**間接層**であり、EC2 のインスタンスプロファイルはこの発行局をメタデータサービス (IMDS) 経由で機械に配管したものである。

4. **IAM は結果整合な分散システムであり、本当の失効保証は期限である。** ポリシー変更・キー無効化は全リージョンの評価点へ非同期に伝播する（ベンダー一次資料も結果整合を明言する）。認可判定のホットパスは複製されたキャッシュであり、[`cloud-control-planes.md`](cloud-control-planes.md) の静的安定性（認可系が沈黙しても API は裁ける）とセキュリティの即時失効要求のトレードオフを、**短い有効期限**で解いている。失効を「無効化の伝播」でなく「期限切れ」で保証するのは、トークン設計 ([`../apis/authentication-tokens-specification.md`](../apis/authentication-tokens-specification.md)) と同一の妥協である。

5. **ポリシー言語の制限された表現力が形式検証を可能にした (Zelkova)。** IAM ポリシーはループも再帰も持たない：条件はワイルドカード文字列・CIDR・数値比較の有限の組合せである。Zelkova (FMCAD '18) はポリシーを SMT の論理式（文字列制約・ビットベクトル）に翻訳し、「あるリクエストが存在して、アカウント外の主体に許可されるか」を**充足可能性判定**として解く。これはテストケースの列挙ではなく、無限のリクエスト空間に対する全称的な証明である。S3 の「パブリックアクセス」警告や IAM Access Analyzer はこの技術の製品化であり、後継の認可言語 **Cedar** は「解析可能性を言語設計の先行条件にする」ことを明示して設計された。ここにあるのは「表現力を捨てるほど強い保証が買える」という、計算理論の階層のセキュリティへの直接応用である。

6. **Confused deputy はクラウドの構造的問題であり、対策は「呼び出し文脈の明示化」に収束する。** サービスがユーザの代理で別リソースへアクセスする構図（deputy = 代理人）は、クラウドではあらゆる場所に現れる。古典例が IMDS 経由の資格情報窃取である：EC2 上のアプリに SSRF があると、攻撃者は `169.254.169.254` へのリクエストを踏み台にロールの一時クレデンシャルを引き出せる。2019 年の Capital One 事件（WAF の SSRF → IMDS → ロール資格情報 → S3 から約 1 億件）はこの連鎖の実証となった。IMDSv2 が要求する「PUT でセッショントークンを取得 + ヘッダ必須 + ホップ制限 1」は、単純 GET の SSRF や NAT 越しの転送では資格情報に届かないよう「**メタデータへの問い合わせであることの明示**」を強制する。クロスアカウントの `sts:ExternalId`、サービス間の `aws:SourceArn` 条件も同型で、「誰の代理としての呼び出しか」を偽造不能な文脈として要求する。これは Hardy が 1988 年に示した capability の教訓の、30 年越しの再演である。

7. **AWS（ポリシー文書）と Google Zanzibar（関係タプル）は認可の双対解である。** AWS は「主体は何をできるか」をポリシー文書（主体・組織側に置く）で表し、Zanzibar (ATC '19) は「このオブジェクトに誰が関係するか」を (object, relation, user) タプルの巨大なグラフ（オブジェクト側に置く）で表す。前者はインフラ権限（種類は少なく、粒度は粗く、変更はまれ）に適し、結果整合で足りる。後者は Google Docs の共有のような**オブジェクト単位・秒単位で変わる**認可に適し、「ACL を古く読んだせいで、削除したはずの相手に新文書が見える」という **new enemy problem** を防ぐため外部一貫性 (Spanner の TrueTime) まで要求する。認可系の設計とは、権限の置き場所と一貫性の強度を、対象ドメインの変更頻度・粒度に合わせて選ぶことだと分かる。

## アーキテクチャ (Architecture)

### リクエスト 1 本の認可経路

```
 Client (SDK)                         AWS API エンドポイント
 ──────────────                       ─────────────────────────────────────
 1. 資格情報の取得                     4. 署名検証 (同じ導出を再現・時刻窓 15 分)
    静的キー / STS / IMDS                 └ 認証: 「この主体からの無改竄の要求である」
 2. 正規リクエスト構築                 5. リクエスト文脈の組立て
    (メソッド・パス・ヘッダ・             principal / action / resource / condition keys
     ペイロードハッシュ)                  (source IP, TLS, タグ, VPC エンドポイント…)
 3. SigV4 署名                        6. ポリシー評価 (下図) → Allow / Deny
    kSigning で HMAC                  7. 実行 + 監査ログ (CloudTrail) へ記録
```

### 評価論理 — 和・積・拒否

```
                 ┌ 明示的 Deny がどこかにあるか? ── yes → DENY (絶対優先)
                 │
 リクエスト文脈 ──┤ 積の柵 (すべて通過が必要):
                 │   SCP は許可するか? ∧ permissions boundary は? ∧ session policy は?
                 │        └ いずれかが不許可 → DENY (柵は allow を生まない)
                 │
                 └ 和の許可 (どれかが必要):
                     identity-based ∨ resource-based のいずれかに Allow があるか?
                       ├ yes → ALLOW
                       └ no  → DENY (暗黙の拒否)

 クロスアカウント: 呼び出し側アカウントの allow ∧ リソース側アカウントの allow (和→積へ転換)
```

## 技術詳細 (Technical Details)

### 1. SigV4 の構造

署名対象は**正規リクエスト (canonical request)**（メソッド、正規化パス、ソート済みクエリ、署名対象ヘッダ、ペイロードの SHA-256）のハッシュを含む文字列である。正規化が厳密に定義されているのは、プロキシや中間層がヘッダ順序・エンコードを揺らしても署名検証が安定するためであり、裏返せば正規化の実装差異はそのまま署名回避の攻撃面になる。導出鍵チェーン（日付→リージョン→サービス→"aws4_request"）は、(a) 秘密そのものをメモリ上で使い回さない、(b) 導出鍵のキャッシュ・委譲を安全にする（署名だけを前段に任せるアーキテクチャが組める）、(c) 漏えい時の被害をスコープで限定する、という 3 役を兼ねる。マルチリージョン対応の SigV4A は導出を ECDSA 署名に置き換えることで「リージョンを事前に固定できない」要求（グローバルアクセラレータ等）へ拡張した。対称鍵の導出スコープと非対称署名の可搬性という、暗号プリミティブの選択がそのままシステム設計の自由度を決める例である（[`../../foundations/mathematics/number-theory-algebra/public-key-cryptography.md`](../../foundations/mathematics/number-theory-algebra/public-key-cryptography.md)）。

### 2. ポリシーの型と役割分担

| ポリシー型 | 付く場所 | 代数上の役割 | 典型用途 |
|---|---|---|---|
| Identity-based | ユーザ/グループ/ロール | 和（allow を与える） | チームの権限設計 |
| Resource-based | リソース (S3, KMS, Lambda…) | 和 + クロスアカウントの受け側 | 共有・公開の制御 |
| SCP (Service Control Policy) | 組織/OU/アカウント | 積（上限） | ガバナンス（リージョン制限等） |
| Permissions boundary | ユーザ/ロール | 積（上限） | 権限委譲の封じ込め |
| Session policy | STS セッション | 積（上限） | 一時的な絞り込み |
| Trust policy | ロール | 「誰が assume できるか」 | ロールの入口制御 |

- 積の柵の実務的意味は**権限管理の委譲**にある：permissions boundary を付けた上でなら、チーム自身に `iam:CreateRole` を許しても、作られるロールの権限は boundary を超えられない。中央チームがすべての権限変更を審査するボトルネックを、代数的な上限で置き換える。
- 条件キー (condition keys) はリクエスト文脈への述語であり、ABAC（タグベース認可）・ネットワーク条件 (`aws:SourceVpce`)・呼び出し元条件 (`aws:SourceArn`) をポリシー本文から参照する。表現力の追加はすべて「SMT に翻訳可能な述語」の範囲で行われている点（発見事項 5）が言語設計の規律である。

### 3. STS・ロール・IMDS の配管

- **AssumeRole**: 呼び出し元の現行資格情報で `sts:AssumeRole` を呼び、trust policy が許せば短命の (AccessKeyId, SecretAccessKey, SessionToken) を得る。権限は role の permission policy ∩ session policy。監査上は「誰がどのロールをいつ引き受けたか」が CloudTrail に残り、権限の**使用**と**引き受け**が分離して記録される。
- **インスタンスプロファイル**: EC2 上のプロセスは IMDS (`169.254.169.254`) からロールの一時資格情報を取得する。資格情報は自動ローテーションされ、ディスクに秘密を置かずに機械へ権限を与える標準経路になった。IMDSv2 では、まず PUT でセッショントークン（TTL 最大 6 時間）を取得し、以後のメタデータ照会にヘッダとして添付する。既定のホップ制限 1 により、コンテナ内からの素朴な転送・NAT 越えの窃取も遮断される。
- **OIDC フェデレーション**: 外部の ID 基盤（Kubernetes, GitHub Actions, 各種 IdP）が発行する署名付きトークンを trust policy で検証して AssumeRole させる。CI からデプロイする際の「リポジトリに長命キーを置く」慣行を、`sub` クレーム（どのリポジトリ・ブランチか）まで条件に落とせる短命トークンで置き換えた。長命秘密の追放は IAM 全体を貫く設計方針である。

### 4. Zelkova — ポリシーの SMT 翻訳

Zelkova はポリシー言語の各要素を SMT-LIB の理論に落とす：Action/Resource のワイルドカード文字列は文字列制約（正規表現相当）、`aws:SourceIp` の CIDR は 32/128 ビットのビットベクトル区間、数値条件は整数制約。2 つのポリシー P, Q に対し「P が許すのに Q が拒むリクエストは存在するか」(P ∧ ¬Q の充足可能性) を問えるため、**ポリシー間の含意・等価・公開性**が判定できる。「バケットポリシーがアカウント外の principal に許可を与えうるか」は unsat なら安全の**証明**であり、sat なら反例リクエストが得られる。健全性のための近似（判定不能な述語は保守側へ倒す）を許容しつつ、CI でポリシー変更のたびに検証を回せる速度で解けるのは、言語に量化子の入れ子や再帰がないからである。Cedar はこの教訓を出発点に、ポリシーの型検査・自動解析・証明支援系による検証済みリファレンス実装 (formal model) を備えた言語として設計された。

### 5. Zanzibar との対比

| 軸 | AWS IAM | Google Zanzibar |
|---|---|---|
| 権限の置き場所 | 主体・組織側のポリシー文書 | オブジェクト側の関係タプル (object#relation@user) |
| 得意な粒度 | インフラ資源（粗く・変更まれ） | エンドユーザのオブジェクト共有（細かく・高頻度変更） |
| 表現の核 | 述語付き allow/deny 文書 | 関係の合成 (userset rewrite: owner ⊆ editor ⊆ viewer) |
| 一貫性 | 結果整合 + 期限で失効保証 | 外部一貫性 (zookie/TrueTime) で new enemy problem を排除 |
| 検証 | SMT による全称的解析 (Zelkova) | グラフ到達可能性の評価がそのまま意味論 |

「認可」という同じ語の下に、要求される一貫性が結果整合から外部一貫性まで幅を持つ点が重要である。IAM の失効遅延が許されるのはインフラ権限の変更頻度が低いからであり、ドキュメント共有に同じ緩さを持ち込むと new enemy problem（古い ACL + 新しいコンテンツの組合せで情報が漏れる）が現実の漏えいになる。[`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) の一貫性の階段は、認可系の設計選択にそのまま再登場する。

## セキュリティ考察 (Security Considerations)

- **IAM 内部の権限昇格経路**: 最小権限の敵は、IAM を操作する権限そのものである。`iam:PassRole`（自分より強いロールをサービスに渡してそのサービス経由で実行）、`iam:CreatePolicyVersion` / `iam:AttachUserPolicy`（自分のポリシーを書き換える）、`sts:AssumeRole` の広すぎる trust policy などがそれにあたる。「IAM への書き込み権限は、実質的にそのアカウントの全権限」として扱い、boundary・SCP の積の柵で封じるのが定石である。ポリシー静的解析（Zelkova 系ツール）はこうした昇格経路の網羅探索にも使われる。
- **資格情報の窃取と検知**: 一時クレデンシャルの窃取（IMDS 経由・環境変数・CI ログ）は、期限が短くとも被害には十分である。防御は多層になる：IMDSv2 の強制、資格情報の**使用場所**の異常検知（発行元インスタンス外からの使用を検出する仕組み）、`aws:SourceVpce` 等の条件によるネットワーク束縛、CloudTrail の監査。Capital One 事件の事例研究が示すのは、単一の SSRF 脆弱性ではなく「広すぎるロール権限 × IMDSv1 × 監視の欠如」という積が被害を決めたことである。
- **評価論理の複雑さ自体がリスクである**: 和と積と拒否の合成、6 種のポリシー型、数百の条件キーという表現力は、誤設定の空間でもある。実クラウド侵害の主因は暗号の破綻ではなく**設定ミス**であり続けており、対策は (a) 人間が読むのではなく機械が検証する (Access Analyzer / Zelkova)、(b) ガードレール（SCP・boundary）で「誤設定しても越えられない床」を敷く、(c) 危険な既定値の廃止（S3 Block Public Access のようなアカウント全体の一括柵）、の 3 点に収束している。
- **fail-closed と可用性**: 認可は fail-closed（判定不能なら拒否）が原則だが、認可系自体の障害が全 API を止めることは許されない。[`cloud-control-planes.md`](cloud-control-planes.md) の静的安定性との調停は「評価に必要なデータをホットパスへ複製し、失効は期限で保証する」というものであり、即時性を犠牲にした結果整合こそが、fail-closed と可用性を両立させる意図的な妥協である。

## 結論 (Conclusions)

- クラウド IAM の設計は 2 本の柱で読める：**ホットパスの無状態化**（署名検証・複製されたポリシー評価・期限による失効）が規模を支え、**代数的なポリシー合成**（和で与え、積で削り、拒否が勝つ）がガバナンスの委譲を支える。どちらも「認可を分散システムとして成立させる」ための選択である。
- 長命の秘密は一貫して追放され続けている：静的キー → ロール → インスタンスプロファイル → OIDC フェデレーション。方向は一つで、「秘密を配る」から「検証可能なアイデンティティを短命の資格情報に交換する」へ。この系譜はトークン一般 ([`../apis/authentication-tokens-specification.md`](../apis/authentication-tokens-specification.md)) と完全に平行である。
- 表現力の制限は IAM 最大の隠れた資産である。ポリシー言語が計算能力を抑制しているからこそ、公開性・含意・昇格経路が SMT で**証明**でき、Cedar のような後継は解析可能性を言語仕様の先行条件に据えるまでになった。セキュリティ機構の価値は「何が書けるか」ではなく「何が保証できるか」で測られる。
- Confused deputy と誤設定という二大事故源は、いずれも「文脈の欠落」に帰着する。欠けているのは、誰の代理か (ExternalId/SourceArn/IMDSv2)、どこまで許すつもりだったか (boundary/SCP/Block Public Access) という文脈である。IAM の進化は、暗黙の文脈を偽造不能な明示的文脈へ置き換える歴史として要約できる。

### 次の接続先 (Related Topics)

- [`../apis/authentication-tokens-specification.md`](../apis/authentication-tokens-specification.md) — トークン形式・OAuth/OIDC・失効と期限の一般論。本書の STS/フェデレーションの前提。
- [`cloud-control-planes.md`](cloud-control-planes.md) — 認可系を含むコントロールプレーンの可用性設計（静的安定性と fail-closed の調停）。
- [`serverless-faas-internals.md`](serverless-faas-internals.md) — 関数単位の実行ロールという最小権限の適用先。
- [`../../foundations/mathematics/discrete-logic/propositional-boolean-logic.md`](../../foundations/mathematics/discrete-logic/propositional-boolean-logic.md) — 充足可能性 (SAT/SMT) の基礎。Zelkova の判定の前提。
- [`../security/software-supply-chain-security.md`](../security/software-supply-chain-security.md) — CI/CD への OIDC フェデレーション適用の背景となる、ビルド基盤の信頼問題。

## 参考文献 (References)

**一次資料 (Primary — 論文・規範仕様)**

- Backes, J., Bolignano, P., Cook, B., et al. (2018). Semantic-based Automated Reasoning for AWS Access Policies using SMT. *FMCAD 2018*, 1–9. DOI: 10.23919/FMCAD.2018.8602994 — Zelkova の一次記述。
- Pang, R., Cáceres, R., Burrows, M., et al. (2019). Zanzibar: Google's Consistent, Global Authorization System. *USENIX ATC '19*. — 関係ベース認可・zookie・new enemy problem の一次記述。
- Cutler, J. W., Disselkoen, C., Eline, A., et al. (2024). Cedar: A New Language for Expressive, Fast, Safe, and Analyzable Authorization. *Proc. ACM Program. Lang. (OOPSLA)*. DOI: 10.1145/3649835 — 解析可能性を先行条件とした認可言語の設計。
- Hardy, N. (1988). The Confused Deputy (or why capabilities might have been invented). *ACM SIGOPS Operating Systems Review*, 22(4), 36–38. DOI: 10.1145/54289.871709 — confused deputy の原典。
- AWS 公式文書（ベンダー一次資料——自社仕様の規範記述）: *Signature Version 4 signing process*; *IAM Policy evaluation logic*; *Instance metadata service v2 (IMDSv2)*. [docs.aws.amazon.com](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html) — Accessed: 2026-07-30。

**二次資料 (Secondary — 独立分析)**

- Novaes Neto, N., Madnick, S., Moraes G. de Paula, A., & Malara Borges, N. (2020). A Case Study of the Capital One Data Breach. *MIT Sloan CISL Working Paper* 2020-07. — SSRF → IMDS → ロール資格情報の連鎖と組織要因の独立分析。
- AWS Security Blog: *Add defense in depth against open firewalls, reverse proxies, and SSRF vulnerabilities with enhancements to the EC2 Instance Metadata Service* (2019). [aws.amazon.com/blogs/security/](https://aws.amazon.com/blogs/security/defense-in-depth-open-firewalls-reverse-proxies-ssrf-vulnerabilities-ec2-instance-metadata-service/) — Accessed: 2026-07-30。IMDSv2 の設計根拠（ベンダー提示）。
