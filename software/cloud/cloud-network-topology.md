# クラウドネットワークトポロジー (Cloud Network Topology: Multi-VPC, PrivateLink & Hybrid Connectivity)

## 概要 (Overview)

[`../networking/virtual-networks-overlay.md`](../networking/virtual-networks-overlay.md) は 1 つの VPC の**内部**——カプセル化・マッピングサービス・Nitro オフロード——を扱った。本ドキュメントはその 1 段上、**VPC を単位とした組織規模のトポロジー設計**を扱う：数十〜数百の VPC (アカウント) をどう繋ぎ、オンプレミスとどう接合し、どこまでの到達性を誰に許すか。ここで設計しているのはパケットの配管ではない。**組織の構造とガバナンスの写像**である——アカウント境界は課金・権限・爆発半径の境界 ([`cloud-iam-authorization.md`](cloud-iam-authorization.md), [`cloud-control-planes.md`](cloud-control-planes.md)) であり、ネットワークトポロジーはその境界群を跨ぐ到達性の政策を配線として固定したものだ。

中心テーゼは 3 つ。(1) **到達性は資産ではなく負債である**——繋がっている範囲は攻撃と障害の伝播範囲であり、トポロジー設計の第一原理は「必要な到達性の最小集合を、監査可能な形で張る」こと。VPC ピアリングが推移性を意図的に禁止し、PrivateLink が「ネットワークを繋がずにサービスだけ繋ぐ」方向へ進化したのは、この原理の帰結である。(2) グラフの形は数で決まる——全結合 (ピアリング) は O(n²) で破綻し、ハブ&スポーク (Transit Gateway) が O(n) で救うが、ハブは相関故障点かつポリシー集中点という代金を持つ。(3) **IP アドレス計画は最も安価な先行投資で、最も高価な事後修正である**——CIDR の重複は後から直せず、組織のネットワークの寿命は最初のアドレス設計で大きく決まる。分析深度は Level 3。

## 方法論 (Methodology)

- VPC 内部の機構 (カプセル化・マッピング・セキュリティグループの施行点) は [`../networking/virtual-networks-overlay.md`](../networking/virtual-networks-overlay.md)、BGP と経路制御の一般論は [`../networking/ip-routing-bgp.md`](../networking/ip-routing-bgp.md)、DNS は [`../networking/dns-resolution.md`](../networking/dns-resolution.md) に依拠し、本書は「その上のトポロジーと政策」に集中する。
- 各接続方式を「到達性の範囲・推移性・アドレス要件・故障と課金の集中度」の 4 軸で比較し、規模 (VPC 数 n) の関数として破綻点を示す。
- マルチ VPC 設計はベンダーのアーキテクチャ文書 (whitepaper——**ベンダー提示**のベストプラクティス) が主資料になる領域である。規範との突き合わせとして、プライベートアドレスの原典 (RFC 1918)、ゼロトラストの標準 (NIST SP 800-207) を置く。

## 発見事項 (Findings)

1. **トポロジーは組織の写像である (Conway の法則のネットワーク版)。** マルチアカウント戦略——チーム・環境 (本番/開発)・ドメインごとにアカウントを分ける——は、権限・課金・爆発半径の分離 ([`cloud-control-planes.md`](cloud-control-planes.md) のセル論の組織版) として確立した。ネットワークはこの分離を跨いで業務を成立させる配線であり、「どの VPC とどの VPC が話せるか」の表は、実質的に「どのチームがどのチームに依存してよいか」の組織設計文書である。ゆえにトポロジー変更が遅い組織は、たいてい配線が遅いのではなく、**写像元の組織的合意が遅い**——ネットワーク設計の会議が組織論になるのは偶然ではない。

2. **ピアリングの非推移性は欠陥ではなく安全装置である。** VPC ピアリング (2 VPC 間の直結) は推移しない——A-B、B-C を繋いでも A-C は通らない。不便に見えるが、これは「**B を経由した暗黙の到達性**」を構造的に禁止する設計である：推移が許されると、1 本のピアリング追加が既存の全接続先への到達性を静かに変え、到達性の表が誰にも読めなくなる。非推移性ゆえに、全対全が要るなら n(n−1)/2 本を明示的に張るしかない——10 VPC で 45 本、50 VPC で 1,225 本。この O(n²) の破綻が、次のハブ&スポークへの移行圧力になる。**明示性はスケールしない、しかし暗黙性は監査できない**——この緊張がトポロジー設計の恒常的なテーマである。

3. **Transit Gateway はクラウド版のハブ&スポークであり、その実体は「マネージドなルータ + 経路表というポリシー集中点」である。** TGW は数千 VPC・VPN・専用線接続を 1 つのハブに収容し、接続数を O(n) にする。本質は**経路表 (route table) の分離**にある——TGW は複数の経路表を持ち、「どのアタッチメントがどの経路表を使い、どの経路を学習するか」で到達性のドメイン (本番同士は通す・開発から本番は通さない・全員が共有サービスには届く) を宣言的に切る。これは [`../networking/ip-routing-bgp.md`](../networking/ip-routing-bgp.md) の VRF (仮想ルーティング) の製品化であり、到達性の政策が**一箇所の表**として読める・監査できる・IaC ([`infrastructure-as-code.md`](infrastructure-as-code.md)) でレビューできる、という管理面の転換が接続数の削減より重要である。代金は集中そのもの——ハブは帯域課金の集中点・変更ミスの爆発半径・AZ 単位の相関故障点になる (ベンダー仕様の冗長化はあるが、設定ミスは冗長化で救えない)。

4. **PrivateLink は「ネットワークを繋がずに、サービスだけを繋ぐ」——到達性の最小化の到達点である。** ピアリング/TGW は**ネットワーク同士**を繋ぐ——繋いだ瞬間、相手の CIDR 全体が経路上に現れ、アドレス重複は許されず、到達性はサブネット・SG で絞り込む「引き算」になる。PrivateLink は逆で、提供側がサービス (NLB の裏の特定エンドポイント) だけを公開し、消費側の VPC には**エンドポイント ENI という 1 つの点**が生える——経路交換なし・CIDR 重複可・方向は消費→提供の単方向のみ。「足し算の到達性」であり、[`../networking/virtual-networks-overlay.md`](../networking/virtual-networks-overlay.md) が「到達性そのものが商品」と呼んだものの完成形である。SaaS 提供・アカウント間の API 公開・「相手のネットワークを一切信頼したくない」統合では、これが既定解になる——ネットワークの結合をサービスの結合まで縮退させることは、マイクロサービスの依存最小化 ([`../networking/service-mesh.md`](../networking/service-mesh.md)) のネットワーク層版である。

5. **ハイブリッド接続は「2 つの自律システムの結婚」であり、BGP が国境の言語になる。** オンプレミスとの接合 (Direct Connect 専用線 / Site-to-Site VPN) の技術的本体は、物理配線ではなく**経路情報の交換の設計**である：どの経路を広告し (集約して最小限に)、どれを受け取り (フィルタして無条件に信じない)、障害時にどちらの経路へ倒れるか (BGP 属性による優先度、[`../networking/ip-routing-bgp.md`](../networking/ip-routing-bgp.md))。事故の型は決まっている——過剰広告 (オンプレの全経路がクラウドへ漏れる・逆も)、非対称経路 (行きは専用線・帰りは VPN でステートフル機器が破綻)、MTU 不一致 (カプセル化の分だけ縮む)。DNS も国境問題になる——「オンプレの名前はオンプレへ、クラウドの名前はクラウドへ」という**条件付き転送の分割解決** (Route 53 Resolver エンドポイント) が、名前解決のトポロジーをパケットのトポロジーと別に設計させる。

6. **IP アドレス計画は「無料に見える資源」の政治である。** RFC 1918 のプライベート空間は有限であり、無計画な `10.0.0.0/16` の乱用は、後の統合 (企業買収・VPC 統合・オンプレ接続) で CIDR 重複として爆発する——重複したネットワーク同士は素直には繋げず、NAT による読み替えという恒久的な複雑さを飲むことになる。対策は退屈だが決定的である：組織全体のアドレス台帳 (IPAM)、リージョン・環境ごとの階層的な割当 (集約可能性 = 経路表の単純さに直結)、そして将来の接続相手 (買収先は `10.0.0.0/16` を使っている、経験則として必ず) を見込んだ余白。IPv6 は枯渇を根本解決するが、移行期は二重運用 (dual stack) の複雑さを払う——「アドレスは余っている方が安い」という一点だけは、どちらの世界でも変わらない。

7. **ネットワーク境界は死なず、「信頼の根拠」から「爆発半径の制御層」へ降格した。** ゼロトラスト (NIST SP 800-207) の主張は「ネットワーク上の位置を信頼の根拠にしない」——境界の内側だから信頼する、をやめ、全アクセスをアイデンティティと文脈で認可する ([`cloud-iam-authorization.md`](cloud-iam-authorization.md) の毎リクエスト認可はその実装である)。しかしこれは「ネットワーク制御の廃止」ではない。プライベートサブネット・SG・到達性の最小化は、**認可の誤り (誤設定・漏えいした資格情報) が悪用される前提での被害範囲の制御**として残る——Capital One 型の連鎖 ([`cloud-iam-authorization.md`](cloud-iam-authorization.md)) は、IAM の誤りとネットワークの過剰な到達性の**積**で成立した。実務の答えは二重の柵である：認可 (誰が何をできるか) と到達性 (そもそも届くか) を独立に最小化し、どちらか一方の失敗が単独では事故にならないようにする。

## アーキテクチャ (Architecture)

### 接続方式の全体像 — ハブ&スポークと点の公開

```
                          オンプレミス DC
                          (BGP で経路交換・集約広告)
                             │ Direct Connect / VPN
 ┌─ Transit Gateway (ハブ) ──┴──────────────────────────┐
 │  経路表A (本番): prod-1, prod-2, shared が相互到達      │
 │  経路表B (開発): dev-1, dev-2 → shared のみ到達        │ ← 到達性の政策が
 └──┬───────┬───────┬───────────┬───────────────────────┘   表として読める
    │       │       │           │
 ┌──▼──┐ ┌──▼──┐ ┌──▼──┐    ┌──▼────────┐
 │prod-1│ │prod-2│ │dev-1 │    │shared VPC  │ (共有サービス・egress 集約)
 └──────┘ └─────┘ └──────┘    └───────────┘

 PrivateLink (トポロジーの外側で「点」だけを繋ぐ):
 ┌─ provider VPC ─────────┐         ┌─ consumer VPC (CIDR 重複可) ─┐
 │  service ── NLB ───────┼──────▶──┼── endpoint ENI (1 つの点)     │
 └────────────────────────┘ 単方向   └──────────────────────────────┘
   経路交換なし・相手のネットワークは見えない・繋がるのはこのサービスだけ
```

### 接続方式の比較 — 4 軸

| 軸 | ピアリング | Transit Gateway | PrivateLink | VPN/Direct Connect |
|---|---|---|---|---|
| 到達性の単位 | VPC 対 VPC (全 CIDR) | 経路表が定める集合 | 単一サービス (単方向) | オンプレ ↔ クラウド |
| 推移性 | なし (意図的) | あり (経路表の範囲で) | なし (点のみ) | BGP 設計次第 |
| アドレス要件 | 重複不可 | 重複不可 | **重複可** | 重複不可 (NAT で回避可) |
| 集中度 (故障・課金) | 分散 (n² 本の管理) | ハブに集中 | 提供側 NLB に集中 | 回線・GW に集中 |
| スケールの破綻点 | 本数 O(n²) | 経路表の複雑さ | サービス数の管理 | 帯域と経路数 |

## 技術詳細 (Technical Details)

### 1. 経路表というポリシー言語

TGW の経路表設計は、実質的に小さなポリシー言語である：アタッチメントの**関連付け** (このVPC はどの表で転送されるか) と**伝播** (この VPC の経路はどの表に載るか) の 2 操作で、到達性の有向グラフを宣言する。定石のパターンは 3 つ——(a) フラット (全員が 1 表・全到達: 最も単純、最も広い爆発半径)、(b) 分離 (環境ごとの表 + 共有サービスへの片方向伝播: 本番/開発の相互不可視)、(c) 検査挿入 (東西トラフィックを検査 VPC 経由に強制: 経路表で「全経路の次ホップ = 検査アプライアンス」と書く)。重要なのは、これが [`cloud-iam-authorization.md`](cloud-iam-authorization.md) のポリシー評価と同じく**宣言的で・一箇所で・機械検査可能**なことである——Network Access Analyzer 型のツールが「意図しない到達経路の存在」を到達可能性解析として検証できるのは、到達性が表に集約されているからであり、ここでも「人間が読むのではなく機械が検証する」路線が貫かれる。

### 2. PrivateLink の内部と制約

エンドポイント ENI は消費側サブネットに生えた通常の ENI であり、宛先はその ENI の私設 IP——つまり消費側から見れば「自分の VPC 内のローカルなIP」への通信である。提供側では NLB ([`../networking/load-balancing-architecture.md`](../networking/load-balancing-architecture.md)) が背後の実体へ分配する。含意：(a) 提供側は消費側の送信元 IP を(既定では)見ない——アイデンティティは接続メタデータやアプリ層で運ぶ、(b) 到達できるのは NLB が指すものだけ——「サービスの公開」の粒度が NLB の構成粒度に縛られる、(c) クロスゾーンの帯域・接続数は NLB がボトルネックの集中点になる。AWS 自身のサービス群への「VPC エンドポイント」も同じ機構であり、「S3 へのアクセスをインターネット経由にしない」という要件は、ゲートウェイ型/インターフェース型エンドポイントで実現される——**クラウド API への到達経路さえも、トポロジー設計の対象**である ([`cloud-iam-authorization.md`](cloud-iam-authorization.md) の `aws:SourceVpce` 条件と組で、経路と認可の二重柵になる)。

### 3. ハイブリッド DNS の分割解決

名前解決の実務は「どのリゾルバに聞くか」のトポロジーである。VPC 内の既定リゾルバはプライベートホストゾーンを解決するが、オンプレのリゾルバはそれを知らず、逆も然り。Route 53 Resolver のインバウンド/アウトバウンドエンドポイントは、両世界のリゾルバ間に**条件付き転送** (このドメインはあちらへ) の橋を架ける——DNS のトポロジーがパケットのトポロジーと独立に設計され、しかも両者が整合しなければならない (名前は引けるが経路がない・逆も然り、はどちらも障害である) 点が、ハイブリッド運用の定番の落とし穴になる。[`../networking/dns-resolution.md`](../networking/dns-resolution.md) の一般論に対し、ここで足すべき認識は一つ——**プライベート DNS は到達性政策の一部**であり、「名前を見せない」ことは弱いながらも層の一つ (見えない名前は打ち間違いや偵察のコストを上げる) として扱われる。

### 4. Egress 設計 — 出口の集約と最小化

インターネットへの出口 (egress) は、コスト (NAT ゲートウェイの処理課金)・監査 (どこへ出て行ったか)・統制 (どこへ出てよいか) の三重の関心事であり、定石は**集約**である：各 VPC に NAT を置かず、egress 専用 VPC に集約して TGW 経由で通す——課金の一元化・出口での検査 (ドメインフィルタ・プロキシ) の挿入点・「出口を持たない VPC」の実現、が同時に得られる。さらに進んだ姿勢は「そもそも出口が要るか」の棚卸しである——依存 (パッケージレジストリ・外部 API) をエンドポイント化・ミラー化していくと、多くのワークロード VPC は**インターネット到達性ゼロ**で成立する。[`../security/software-supply-chain-security.md`](../security/software-supply-chain-security.md) の文脈では、出口の最小化はデータ持ち出し (exfiltration) 経路の最小化でもある——到達性は負債、の最も直接的な適用である。

### 5. トポロジーの検証と変更管理

到達性の意図と実際の乖離は、IaC の drift ([`infrastructure-as-code.md`](infrastructure-as-code.md)) と同じ構造の問題である。対策も同型で、(a) トポロジー (TGW 経路表・SG・NACL・エンドポイント) を IaC の管理下に置き、変更をレビューに通す、(b) 到達可能性解析 (「開発 VPC から本番 DB へ届く経路が存在するか」を形式的に検査するツール群) を CI に組み込む——[`cloud-iam-authorization.md`](cloud-iam-authorization.md) の Zelkova がポリシーに対して行うことを、経路に対して行う、(c) フローログを「実際に流れたもの」の記録として、意図 (経路表) との突合に使う。到達性は「作った時に正しい」では足りず、「正しくあり続けることが検証され続ける」必要がある——調停と検証の一般原則が、配線の層にも適用される。

## セキュリティ考察 (Security Considerations)

- **到達性の最小化は、認可の誤りに対する保険である。** IAM・アプリ認可・ネットワークのどれか一つの失敗が事故に直結しない構造 (独立な柵の積) が設計目標であり、「IAM があるからネットワークは全通でよい」「境界があるから認可は緩くてよい」はどちらも単一障害点の設計である。SSRF→IMDS→S3 の連鎖 ([`cloud-iam-authorization.md`](cloud-iam-authorization.md)) は、各層の柵が独立に立っていれば途中で切れていた——多層の柵は冗長ではなく、失敗モードの独立性である ([`../../foundations/mathematics/probability-statistics/reliability-theory.md`](../../foundations/mathematics/probability-statistics/reliability-theory.md) の並列合成)。
- **共有ネットワークサービスは共有の攻撃面である。** TGW・共有 VPC・中央 egress・共有 DNS は、効率と統制の集中点であると同時に、侵害・誤設定の影響が全接続者へ及ぶ相関点である——ハブの管理権限は [`cloud-control-planes.md`](cloud-control-planes.md) の「コントロールプレーンは最高権限の攻撃対象」に準じて扱い、変更の多者承認・最小の管理者集合・完全な監査を課す。
- **「プライベート」は「安全」ではない。** プライベートサブネット・PrivateLink・エンドポイントは経路の性質であって認証ではない——プライベート経路の先のサービスにも、公開サービスと同じ認証・認可 ([`cloud-iam-authorization.md`](cloud-iam-authorization.md))・暗号化 (TLS, [`../networking/tls-internals.md`](../networking/tls-internals.md)) を要求するのがゼロトラストの実務的意味である。「内部だから平文・無認証」の系は、境界の一点突破で全損する——境界は被害の減速材であり、防弾壁ではない。
- **偵察と横移動の経済を悪化させる。** 攻撃者の内部活動は「どこに何があり、どこへ届くか」の探索から始まる——フラットなネットワークは探索を安くし、細分化された到達性 (セグメンテーション)・見えない名前・出口のない VPC は探索と持ち出しを高くする。防御の目的を「侵入の完全阻止」から「侵入後の経済の悪化」へ広げたとき、トポロジー設計はセキュリティ工学の主要な道具になる——爆発半径の議論 ([`cloud-control-planes.md`](cloud-control-planes.md)) が、障害から攻撃へそのまま拡張される。

## 結論 (Conclusions)

- クラウドネットワークトポロジーの第一原理は「到達性は負債である」——必要最小の到達性を、明示的に・監査可能に・機械検証可能に張る。非推移のピアリング、経路表というポリシー集中点、点だけを繋ぐ PrivateLink という進化は、この原理が製品史として現れたものである。
- グラフの形は規模の関数である：少数なら明示的なピアリング、数十からはハブ&スポーク (TGW)、サービス境界の統合は PrivateLink——そして選択の実体は「管理の何を集中させ、何を分散させるか」であり、[`cloud-control-planes.md`](cloud-control-planes.md) のセルとハブの緊張がネットワーク層で再演される。
- ハイブリッドとアドレス計画は「後から直せないもの」の代表である。BGP の広告規律と階層的な CIDR 台帳という退屈な初期投資が、数年後の統合可能性を決める——本領域で繰り返し見た「書き込み時の秩序が読み取り時の自由を買う」の、最もタイムスケールの長い実例と言える。
- ゼロトラスト時代のネットワーク境界の正しい位置づけは「信頼の根拠としては廃止、爆発半径の制御層としては強化」である。認可と到達性という独立な二重の柵——どちらか単独の失敗が事故にならない構造——が、本領域のセキュリティ考察を貫いてきた結論のトポロジー版であり、この設計思想の一貫性こそが、クラウドを学ぶ際に個々のサービス名より先に掴むべきものである。

### 次の接続先 (Related Topics)

- [`../networking/virtual-networks-overlay.md`](../networking/virtual-networks-overlay.md) — 1 つの VPC の内部機構。本書の下の層。
- [`../networking/ip-routing-bgp.md`](../networking/ip-routing-bgp.md) — BGP・経路制御・VRF の一般論。TGW とハイブリッドの理論的土台。
- [`../networking/dns-resolution.md`](../networking/dns-resolution.md) — 分割解決の前提となる DNS の一般機構。
- [`cloud-iam-authorization.md`](cloud-iam-authorization.md) — 到達性と対を成すもう一方の柵 (認可)。`aws:SourceVpce` による両者の結合。
- [`infrastructure-as-code.md`](infrastructure-as-code.md) — トポロジーの変更管理・drift・到達可能性検証の CI 統合。

## 参考文献 (References)

**規範・標準 (Standards)**

- Rekhter, Y., Moskowitz, B., Karrenberg, D., de Groot, G. J., & Lear, E. (1996). *RFC 1918: Address Allocation for Private Internets*. IETF. — プライベートアドレス空間の原典。
- Rose, S., Borchert, O., Mitchell, S., & Connelly, S. (2020). *NIST SP 800-207: Zero Trust Architecture*. DOI: 10.6028/NIST.SP.800-207 — 「ネットワーク位置を信頼の根拠にしない」の標準定式化。

**ベンダー提示 (Vendor-presented)**

- AWS Whitepaper. *Building a Scalable and Secure Multi-VPC AWS Network Infrastructure*. [docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/welcome.html) — Accessed: 2026-07-30。マルチ VPC・TGW・ハイブリッド・DNS 設計の体系的なベンダー文書。
- AWS 文書: *Amazon VPC peering (non-transitivity)*; *AWS Transit Gateway route tables*; *AWS PrivateLink*; *Amazon Route 53 Resolver endpoints*; *AWS Direct Connect*. [docs.aws.amazon.com](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html) — Accessed: 2026-07-30。各接続方式の意味論の規範記述。
- Amazon VPC IP Address Manager (IPAM) 文書. [docs.aws.amazon.com/vpc/latest/ipam/](https://docs.aws.amazon.com/vpc/latest/ipam/what-it-is-ipam.html) — Accessed: 2026-07-30。組織的アドレス台帳の実装。
