# 認証トークン仕様書 (Authentication Token Specification) — JWT・OAuth 2.0・OIDC

**Version**: 3.0
**Last Updated**: 2026-10-04
**Author**: Claude @ Daedalus

---

## 概要 (Overview)

本ドキュメントは、JWT (JSON Web Token)、OAuth 2.0、OpenID Connect (OIDC) を中心とする現代の認証 (authentication)・認可 (authorization) メカニズムを、IETF RFC と OpenID Foundation の仕様を一次資料として分析する。これらは Web アプリケーション・モバイルアプリ・API・マイクロサービスにおけるアイデンティティ管理の基盤である。

3 つの関係を先に一言でまとめると、**JWT はトークンの「形式」、OAuth 2.0 はアクセス委任の「プロトコル」、OIDC は OAuth 2.0 の上に認証を載せた「アイデンティティ層」** である。2012 年の OAuth 2.0 (RFC 6749) 以降、この体系は「選択肢を増やす」方向ではなく「危険な選択肢を削り、トークンを送信者に束縛する」方向へ進化してきた。その到達点が OAuth 2.0 Security BCP (RFC 9700, 2025 年 1 月) と、策定中の OAuth 2.1 である。

本書の範囲と前提は以下のとおり。

- **範囲**: トークン形式 (JWS/JWT, RFC 9068 のアクセストークンプロファイル)、認可フロー (認可コード + PKCE、クライアントクレデンシャル、トークン交換)、認可要求の強化 (PAR, RAR)、送信者制約 (DPoP, mTLS)、OIDC の ID トークン、エラー処理、標準の成熟度。
- **下位層**: トランスポートの機密性・サーバー認証は TLS に依存する。TLS の詳細は [`../networking/tls-internals.md`](../networking/tls-internals.md) を参照。
- **隣接領域**: ブラウザ側の脅威 (XSS, CSRF, Cookie 属性) は [`../browsers/web-security-model.md`](../browsers/web-security-model.md)、クラウド IAM での認可評価は [`../cloud/cloud-iam-authorization.md`](../cloud/cloud-iam-authorization.md)、署名アルゴリズムの数学的基礎は [`../../foundations/mathematics/number-theory-algebra/public-key-cryptography.md`](../../foundations/mathematics/number-theory-algebra/public-key-cryptography.md) が扱う。

### 主要用語 (Key Terminology)

| 用語 | 英語 | 説明 |
|------|------|------|
| 認証 | Authentication | ユーザーやシステムの身元を検証すること |
| 認可 | Authorization | リソースへのアクセス権限を付与すること |
| トークン | Token | 認証・認可の状態を表現するデータ |
| クレーム | Claim | エンティティ（通常はユーザー）に関する記述 |
| スコープ | Scope | トークンが許可するアクセスの範囲 |
| ベアラートークン | Bearer Token | 所持しているだけで使えるトークン（RFC 6750） |
| 送信者制約トークン | Sender-Constrained Token | 特定の鍵・証明書の所持証明がないと使えないトークン（DPoP, mTLS） |
| 公開クライアント | Public Client | クライアントシークレットを安全に保持できないクライアント（SPA, ネイティブアプリ） |
| 機密クライアント | Confidential Client | サーバー側でクレデンシャルを保持できるクライアント |

---

## 方法論 (Methodology)

- **一次資料の優先**: プロトコルの記述はすべて IETF RFC・Internet-Draft、OpenID Foundation 仕様を根拠とする。各 RFC の発行年月・ステータスは RFC Editor のメタデータで、ドラフトの版数・状態は IETF Datatracker で 2026-10-04 に確認した。
- **成熟度の明示**: 「RFC (Proposed Standard / BCP)」「Internet-Draft（策定中）」「OpenID Final Specification」を区別して表記する。ドラフトの内容は変わりうるため、引用時は版数と確認日を併記する。
- **脅威駆動の整理**: 各機構を「どの攻撃を防ぐために導入されたか」で位置づける（例: PKCE は認可コード横取り、DPoP はトークン窃取後のリプレイ）。RFC 9700 の脅威モデルを整理の軸に使う。
- **二次資料の扱い**: OWASP・PortSwigger 等の解説は攻撃手法の具体例を確認するためにのみ用い、規範的な主張は RFC に帰属させる。
- **深度**: Level 3（全フロー、エッジケース、失敗モード、セキュリティ、最適化）。

---

## 発見事項 (Findings)

1. **OAuth 2.0 の「推奨構成」は RFC 9700 で事実上固定された。** 2025 年 1 月発行の RFC 9700 (BCP 240) は、インプリシットグラントを使うべきでないとし、リソースオーナーパスワードクレデンシャル (ROPC) を「MUST NOT」とした。すべてのクライアントに PKCE を求め、リダイレクト URI の完全一致を義務づけ、送信者制約トークン (DPoP / mTLS) の使用を推奨する[^rfc9700]。
2. **OAuth 2.1 は 2026-10-04 時点でまだ RFC ではない。** 最新は draft-ietf-oauth-v2-1-16 (2026-09-03) で、WG のマイルストーンは「2026 年 12 月に IESG へ提出」である[^oauth21]。内容は RFC 6749/6750 と RFC 9700 の統合であり、新機能の追加ではなく危険な選択肢の削除である。
3. **ベアラートークンから送信者制約トークンへの移行が標準化済み。** DPoP (RFC 9449, 2023 年 9 月) はアプリケーション層の公開鍵署名で、mTLS (RFC 8705, 2020 年 2 月) は TLS クライアント証明書で、トークンを鍵に束縛する[^rfc9449][^rfc8705]。窃取されたトークン単体では使えなくなる。
4. **JWT アクセストークンには専用プロファイルがある。** RFC 9068 (2021 年 10 月) は `typ: at+jwt` と必須クレーム (`iss`, `exp`, `aud`, `sub`, `client_id`, `iat`, `jti`) を定め、ID トークンとの取り違えを型で防ぐ[^rfc9068]。
5. **JWT BCP は改訂中。** RFC 8725 (2020) の後継 draft-ietf-oauth-rfc8725bis は 2026-08-21 の -10 版で IESG 承認後の RFC Editor Queue にあり、RFC 8725 を廃止 (obsolete) し RFC 7519 を更新する予定である[^rfc8725bis]。現時点で規範として引用できるのは RFC 8725 である。
6. **認可要求自体の保護と表現力が拡張された。** PAR (RFC 9126) は認可要求をフロントチャネルから外し、RAR (RFC 9396) はスコープ文字列では表せない細粒度の権限を構造化 JSON で表す[^rfc9126][^rfc9396]。トークン交換 (RFC 8693) は委任 (delegation) と成りすまし (impersonation) を STS (Security Token Service) として標準化した[^rfc8693]。
7. **OIDC は ISO 標準にもなった。** OpenID Connect Core 1.0 (errata set 2) は 2024 年 10 月に ISO/IEC 26131:2024 として発行された[^iso26131]。高保証用途向けには FAPI 2.0 Security Profile が 2025 年 2 月に OpenID Final Specification となった[^fapi2]。
8. **JWT の構造的弱点は変わっていない。** 自己完結トークンは即時失効が難しい。主要な脆弱性（`alg` 混同、`none`、`kid` 悪用）はいずれも「トークン内のヘッダーを信頼する」実装ミスに起因する。

---

## アーキテクチャ (Architecture)

### 役割と信頼境界 (Roles and Trust Boundaries)

OAuth 2.0 は 4 つの役割を定義する[^rfc6749]。実運用ではこれに、署名検証鍵を配布する JWKS エンドポイントと、メタデータ公開 (RFC 8414) が加わる[^rfc8414]。

```mermaid
flowchart LR
    RO["リソースオーナー<br/>(Resource Owner)<br/>エンドユーザー"]
    C["クライアント<br/>(Client)<br/>アプリケーション"]
    subgraph ASB ["認可サーバー (Authorization Server)"]
        AZ["/authorize<br/>(フロントチャネル)"]
        TK["/token<br/>(バックチャネル)"]
        PAR["/par<br/>(RFC 9126)"]
        JW["JWKS / メタデータ<br/>(RFC 8414)"]
        IN["/introspect<br/>(RFC 7662)"]
    end
    RS["リソースサーバー<br/>(Resource Server)"]

    RO -- "1. 認証・同意" --> AZ
    C -- "2. 認可要求の事前登録" --> PAR
    C -- "3. コード → トークン交換" --> TK
    C -- "4. トークン提示" --> RS
    RS -- "5a. 署名検証鍵の取得" --> JW
    RS -- "5b. 不透明トークンの照会" --> IN
```

| 役割 | 説明 | 信頼の前提 |
|------|------|------------|
| リソースオーナー (Resource Owner) | アクセスを許可できるエンティティ（通常はエンドユーザー） | ブラウザ/端末は攻撃者の影響下にありうる |
| クライアント (Client) | アクセスを要求するアプリケーション | 公開クライアントはシークレットを保持できない |
| 認可サーバー (Authorization Server) | 認証後にトークンを発行する | 信頼の起点。鍵の保護が全体の安全性を決める |
| リソースサーバー (Resource Server) | 保護されたリソースをホストする | トークンの `aud`・署名・有効期間・束縛を検証する |

**フロントチャネル**（ブラウザのリダイレクトを経由する経路）は改竄・漏洩しやすく、**バックチャネル**（クライアントから AS への直接 HTTPS）は TLS で保護される。現代の OAuth のセキュリティ機構の多くは「フロントチャネルに載る値を減らし、載る値は一回性・束縛付きにする」という設計原理で説明できる。

### 仕様の層構造 (Specification Layering)

```mermaid
flowchart TB
    OIDC["OpenID Connect Core 1.0<br/>(認証・ID トークン)"]
    FAPI["FAPI 2.0 Security Profile<br/>(高保証プロファイル)"]
    OAUTH["OAuth 2.0 / 2.1 draft<br/>(RFC 6749, 6750, 9700)"]
    EXT["拡張: PKCE 7636 / PAR 9126 / RAR 9396<br/>DPoP 9449 / mTLS 8705 / Token Exchange 8693"]
    JOSE["JOSE: JWS 7515 / JWE 7516 / JWK 7517 / JWA 7518<br/>JWT 7519, JWT BCP 8725, JWT AT Profile 9068"]
    TLS["TLS 1.2 / 1.3"]
    FAPI --> OIDC
    FAPI --> OAUTH
    OIDC --> OAUTH
    OAUTH --> EXT
    OAUTH --> JOSE
    OIDC --> JOSE
    EXT --> TLS
    OAUTH --> TLS
```

OAuth はトークンの形式を規定しない。JWT はその「形式」の有力な選択肢であり、OIDC の ID トークンは仕様上 JWT と決まっている。FAPI 2.0 は個別の RFC を「どれを必須にするか」を束ねたプロファイルであり、新しいプロトコルではない。

### アクセストークンの 2 つの検証モデル (Token Validation Models)

| モデル | トークン形式 | RS の検証方法 | 失効の即時性 | AS への負荷 |
|--------|--------------|---------------|--------------|-------------|
| 自己完結型 (self-contained) | JWT (RFC 9068) | JWKS の公開鍵でローカルに署名検証 | 低い（`exp` まで有効） | 低い |
| 参照型 (reference / opaque) | ランダム文字列 | イントロスペクション (RFC 7662) で AS に照会 | 高い | 高い（キャッシュで緩和） |

両者は排他的ではない。短命 JWT で通常経路を高速化し、高リスク操作だけイントロスペクションで最新状態を確認する、という混成が一般的である[^rfc7662]。

---

## 技術詳細 (Technical Details)

### JWT (JSON Web Token)

JWT は、二者間で転送されるクレームを表現するための、コンパクトで URL セーフなトークン形式である[^rfc7519]。署名付き JWT は JWS (JSON Web Signature) のコンパクト直列化であり[^rfc7515]、完全性 (integrity) を保証する。

#### 構造 (Structure)

JWT はピリオド (`.`) で区切られた 3 部分から構成される。

```text
xxxxx.yyyyy.zzzzz
[Header].[Payload].[Signature]
```

**ヘッダー (Header)** はトークンのタイプと署名アルゴリズムを指定する。

```json
{
  "alg": "ES256",
  "typ": "at+jwt",
  "kid": "2026-09-signing-key"
}
```

`typ` は用途を明示する型付け (explicit typing) に使う。RFC 8725 §3.11 は、異なる種類の JWT を取り違える攻撃を防ぐために明示的な型付けを推奨している[^rfc8725]。

**ペイロード (Payload)** はクレームを格納する。RFC 7519 が定義する登録済みクレーム (registered claims) は以下のとおり。

| クレーム | 名称 | 説明 |
|----------|------|------|
| `iss` | Issuer | トークン発行者 |
| `sub` | Subject | トークンの対象者 |
| `aud` | Audience | 想定される受信者 |
| `exp` | Expiration Time | 有効期限（UNIX タイムスタンプ） |
| `nbf` | Not Before | この時刻より前は無効 |
| `iat` | Issued At | トークン発行時刻 |
| `jti` | JWT ID | 一意識別子（リプレイ検出などに利用） |

**署名 (Signature)** は、エンコード済みヘッダーとペイロードを `.` で連結した文字列（署名入力）に対して、ヘッダー指定のアルゴリズムで計算する。

```text
signing_input = base64url(header) + "." + base64url(payload)
signature     = Sign(key, signing_input)        # e.g. ECDSA P-256 / SHA-256 for ES256
```

ヘッダーとペイロードは Base64URL エンコードされているだけで、**暗号化されていない**。署名が保証するのは完全性であって機密性 (confidentiality) ではない。秘匿が必要なら JWE (JSON Web Encryption, RFC 7516) を使うか、ペイロードに機密情報を入れない設計にする。

#### 署名アルゴリズム (Signing Algorithms)

署名アルゴリズムは JWA で定義される[^rfc7518]。

| アルゴリズム | 種別 | 説明 | 備考 |
|--------------|------|------|------|
| HS256 / HS384 / HS512 | 対称鍵 (symmetric) | HMAC + SHA-2（共有シークレット） | 検証者全員が署名能力も持つ |
| RS256 / RS384 / RS512 | 非対称鍵 (asymmetric) | RSASSA-PKCS1-v1_5 + SHA-2 | 広く互換。鍵長 2048 ビット以上 |
| PS256 / PS384 / PS512 | 非対称鍵 | RSASSA-PSS + SHA-2 | FAPI 2.0 が許可するアルゴリズムの 1 つ |
| ES256 / ES384 / ES512 | 非対称鍵 | ECDSA（P-256/P-384/P-521）+ SHA-2 | 署名が短い。乱数品質に依存 |
| `none` | なし | 署名なし JWT | **本番では必ず拒否する** |

対称鍵 (HS*) では発行者と検証者が同一シークレットを共有するため、検証者が多数いるマイクロサービス構成では鍵配布の問題が生じ、どの検証者も偽造できてしまう。検証者が多い構成では、公開鍵で検証できる非対称鍵 (RS*/PS*/ES*) を選ぶ。RSA・楕円曲線署名の安全性の根拠は [`../../foundations/mathematics/number-theory-algebra/public-key-cryptography.md`](../../foundations/mathematics/number-theory-algebra/public-key-cryptography.md) を参照。署名鍵を HSM / KMS に置く設計は [`../cloud/cloud-kms-confidential-computing.md`](../cloud/cloud-kms-confidential-computing.md) が扱う。

#### JWT アクセストークンプロファイル (RFC 9068)

OAuth 2.0 はアクセストークンの形式を定めないため、ベンダーごとに JWT のクレーム構成がばらばらだった。RFC 9068 はこれを統一する[^rfc9068]。

| 要件 | 内容 |
|------|------|
| ヘッダー `typ` | `at+jwt`（または `application/at+jwt`）とする（SHOULD） |
| 必須クレーム | `iss`, `exp`, `aud`, `sub`, `client_id`, `iat`, `jti` |
| 任意クレーム | `auth_time`, `acr`, `amr`, `scope`, `groups`, `roles`, `entitlements` など |
| RS の検証 | `typ` が `at+jwt` 以外なら拒否（MUST）。`alg: none` は拒否（MUST）。署名・`iss`・`aud`・`exp` を検証 |

`typ` の検証によって、ID トークン（`typ` が異なる）をアクセストークンとして API に流用する攻撃を防げる。

### OAuth 2.0 の認可フロー (Grant Types)

#### 認可コード + PKCE (Authorization Code with PKCE)

認可コードグラントは、一時的な認可コードをバックチャネルでトークンと交換する。トークンがブラウザを経由しない点が要である[^rfc6749]。PKCE (Proof Key for Code Exchange) はこれを拡張し、認可コードの横取り (interception) を防ぐ[^rfc7636]。RFC 9700 はすべてのクライアントに PKCE を求め、AS に PKCE のサポートと強制を求めている[^rfc9700]。OAuth 2.1 ドラフトでは認可コードフローで PKCE が必須になる[^oauth21]。

```mermaid
sequenceDiagram
    autonumber
    participant U as ユーザー/ブラウザ
    participant C as クライアント
    participant AS as 認可サーバー
    participant RS as リソースサーバー

    C->>C: code_verifier を乱数生成し code_challenge = BASE64URL-SHA256 を計算
    C->>U: /authorize へリダイレクト (code_challenge, S256, state, redirect_uri)
    U->>AS: 認可要求
    AS->>U: ログイン・同意画面
    U->>AS: 認証して許可
    AS->>U: redirect_uri へリダイレクト (code, state, iss)
    U->>C: code と state を受け渡し
    C->>C: state と iss を検証
    C->>AS: POST /token (code, code_verifier, クライアント認証)
    AS->>AS: SHA256(code_verifier) と code_challenge を照合
    AS->>C: access_token (+ refresh_token, id_token)
    C->>RS: Authorization: Bearer または DPoP でリクエスト
    RS->>RS: 署名・aud・exp・束縛を検証
    RS->>C: 保護されたリソース
```

PKCE の計算は次のとおりである。

```text
# 1. Generate a high-entropy random code_verifier (43-128 chars, RFC 7636 §4.1)
code_verifier  = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

# 2. Derive the code_challenge with S256 (plain MUST NOT be used when S256 is possible)
code_challenge = BASE64URL(SHA256(ASCII(code_verifier)))

# 3. Authorization request
GET /authorize?response_type=code&client_id=...&redirect_uri=...
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256&state=...

# 4. Token request
POST /token
  grant_type=authorization_code&code=...&redirect_uri=...&code_verifier=dBjftJeZ4CVP-...
```

認可コードを盗んでも `code_verifier` を知らなければトークンに交換できない。PKCE は「認可要求を出した主体とトークン要求を出した主体が同一である」ことの一回性の証明である。

認可レスポンスに含まれる `iss` パラメータ (RFC 9207) は、複数の AS を扱うクライアントでのミックスアップ攻撃 (mix-up attack) を防ぐ[^rfc9207]。RFC 9700 はミックスアップ対策として `iss` の利用（または AS ごとに異なるリダイレクト URI）を求めている。

#### プッシュ型認可要求 (PAR, RFC 9126)

通常の認可要求では、パラメータがすべてブラウザの URL に載る。これは改竄・漏洩・URL 長制限の原因になる。PAR では、クライアントが認可要求の中身をバックチャネルで AS に事前送信 (push) し、返された `request_uri` だけをブラウザに渡す[^rfc9126]。

```text
POST /par HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic █████████

response_type=code&client_id=s6BhdRkqt3&redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
&code_challenge=E9Melhoa...&code_challenge_method=S256&scope=accounts

HTTP/1.1 201 Created
{"request_uri": "urn:ietf:params:oauth:request_uri:6esc_11ACC5bwc014ltc14eY22c", "expires_in": 60}

GET /authorize?client_id=s6BhdRkqt3&request_uri=urn%3Aietf%3Aparams%3Aoauth%3Arequest_uri%3A6esc_11ACC5bwc014ltc14eY22c
```

利点は 3 つある。(1) 認可要求の完全性と機密性が TLS で守られる。(2) AS がユーザー対話の前にクライアントを認証できる。(3) RAR の大きな JSON も URL 長を気にせず送れる。FAPI 2.0 Security Profile は PAR を必須としている[^fapi2]。

#### リッチ認可要求 (RAR, RFC 9396)

スコープは空白区切りの文字列であり、「口座 X から Y へ 100 ユーロ送金」のような取引単位の権限を表せない。RAR は `authorization_details` パラメータで、`type` を持つ JSON オブジェクトの配列として権限を表す[^rfc9396]。

```json
[
  {
    "type": "payment_initiation",
    "actions": ["initiate"],
    "locations": ["https://api.bank.example/payments"],
    "instructedAmount": { "currency": "EUR", "amount": "100.00" },
    "creditorAccount": { "iban": "DE02100100109307118603" }
  }
]
```

AS は同意画面でこの内容を提示し、発行するトークン（またはイントロスペクション応答）に `authorization_details` を含める。RS はスコープ文字列ではなく構造化された権限を評価する。これはクラウド IAM のポリシー評価に近い発想である（[`../cloud/cloud-iam-authorization.md`](../cloud/cloud-iam-authorization.md)）。

#### クライアントクレデンシャルグラント (Client Credentials Grant)

ユーザーが関与しないマシン間 (M2M) 通信に使用する。

```text
POST /token HTTP/1.1
Host: authorization-server.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic █████████

grant_type=client_credentials&scope=read:data
```

クライアント認証は共有シークレット (`client_secret_basic`) より、秘密鍵による署名付きアサーション (`private_key_jwt`) や mTLS (RFC 8705) の方が強い。RFC 9700 は非対称暗号ベースのクライアント認証を推奨している[^rfc9700]。

#### トークン交換 (Token Exchange, RFC 8693)

マイクロサービス連鎖では、フロントの API が受け取ったトークンを下流にそのまま転送すると、`aud` が広すぎるトークンが内部を流通する。トークン交換は、AS を STS として使い、受け取ったトークンを下流向けの別トークンに交換する[^rfc8693]。

```text
POST /token
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=eyJhbGciOi...           # token representing the end user
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&actor_token=eyJhbGciOi...             # token representing the calling service (delegation)
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
&audience=https://orders.internal.example
```

`actor_token` を伴う場合は**委任 (delegation)** となり、発行トークンの `act` クレームに「誰が誰の代理で行動しているか」が記録される。`actor_token` がない場合は**成りすまし (impersonation)** となり、下流からは主体と区別できない。監査性の観点からは委任が望ましい。

#### 非推奨のグラント (Deprecated Grants)

| グラント | 問題 | RFC 9700 | OAuth 2.1 draft |
|----------|------|----------|-----------------|
| インプリシット (Implicit) | アクセストークンが URL フラグメントに露出し、漏洩・リプレイに弱い | 使うべきでない (SHOULD NOT) | 削除 |
| リソースオーナーパスワード (ROPC) | 資格情報をクライアントに渡す。MFA と両立しない | 使ってはならない (MUST NOT) | 削除 |

どちらも認可コード + PKCE で置き換える。ROPC はレガシーシステムの移行期間に限る。

### 送信者制約トークン (Sender-Constrained Tokens)

ベアラートークンは「持っている者が使える」ため、ログ・プロキシ・XSS・悪意ある RS から漏れた時点で悪用できる。送信者制約は、トークンをクライアントの鍵に束縛し、リクエストごとに鍵の所持証明を求める。

#### DPoP (RFC 9449)

DPoP (Demonstrating Proof of Possession) は、クライアントが生成した公開鍵ペアで、リクエストごとに `DPoP` ヘッダーの証明 JWT に署名する[^rfc9449]。TLS 層に手を入れないため、ブラウザの SPA でも使える。

```mermaid
sequenceDiagram
    autonumber
    participant C as クライアント
    participant AS as 認可サーバー
    participant RS as リソースサーバー

    C->>C: 鍵ペア生成 (秘密鍵は非抽出可能な形で保持)
    C->>AS: POST /token + DPoP ヘッダー (htm=POST, htu=/token, jwk=公開鍵)
    AS->>AS: 証明 JWT の署名・htm・htu・iat・jti を検証
    AS->>C: access_token (cnf.jkt = 公開鍵サムプリント), token_type=DPoP
    C->>RS: Authorization: DPoP トークン + DPoP ヘッダー (htm, htu, ath=トークンのハッシュ)
    RS->>RS: 証明の署名を jwk で検証し jkt がトークンの cnf と一致するか確認
    alt サーバーがノンスを要求
        RS->>C: 401 error=use_dpop_nonce + DPoP-Nonce
        C->>RS: nonce を含めた新しい証明で再送
    end
    RS->>C: 保護されたリソース
```

DPoP 証明 JWT の要素は次のとおりである。

| 要素 | 位置 | 意味 |
|------|------|------|
| `typ: dpop+jwt` | ヘッダー | 明示的型付け |
| `alg` | ヘッダー | 非対称署名アルゴリズム（`none`・対称鍵は不可） |
| `jwk` | ヘッダー | クライアントの公開鍵 |
| `jti` | クレーム | 証明ごとの一意 ID（リプレイ検出） |
| `htm` / `htu` | クレーム | HTTP メソッドとクエリを除いた URI（別リクエストへの流用防止） |
| `iat` | クレーム | 生成時刻（鮮度の検証） |
| `ath` | クレーム | アクセストークンの SHA-256 ハッシュ（RS 宛て時に必須） |
| `nonce` | クレーム | サーバー提供ノンス（事前生成された証明の悪用防止） |

アクセストークン側では `cnf.jkt`（JWK SHA-256 サムプリント）で鍵に束縛される。エラーには `invalid_dpop_proof` と `use_dpop_nonce` が定義されている。

#### mTLS 証明書束縛トークン (RFC 8705)

RFC 8705 は、TLS 相互認証 (mutual TLS) を (1) クライアント認証と (2) トークンの証明書束縛の 2 用途に使う[^rfc8705]。AS は発行時のクライアント証明書の SHA-256 サムプリントを `cnf` の `x5t#S256` に記録し、RS は TLS 接続で提示された証明書のサムプリントと照合する。

| 観点 | DPoP (RFC 9449) | mTLS (RFC 8705) |
|------|-----------------|-----------------|
| 束縛の層 | アプリケーション層（JWT 署名） | トランスポート層（TLS クライアント証明書） |
| ブラウザ SPA | 使える（WebCrypto の非抽出鍵） | 実質的に困難 |
| TLS 終端プロキシ | 影響を受けない | 証明書情報の転送設計が必要 |
| リクエストごとのコスト | 署名生成・検証 | ハンドシェイク時のみ |
| 主な用途 | SPA・モバイル・一般 API | 金融・B2B・サービスメッシュ |

TLS 終端とクライアント証明書の扱いは [`../networking/tls-internals.md`](../networking/tls-internals.md) を参照。

### トークンのライフサイクル (Token Lifecycle)

| トークン | 目的 | 典型的な有効期間 |
|----------|------|------------------|
| アクセストークン (access token) | 保護されたリソースへのアクセス | 短命（分〜1 時間程度） |
| リフレッシュトークン (refresh token) | 新しいアクセストークンの取得 | 長命（日〜月単位） |
| ID トークン (ID token) | 認証イベントの証明（クライアント宛て） | 短命。API 認可には使わない |

有効期間の値は仕様が固定するものではなく、運用上の目安である。RFC 9700 は、公開クライアントのリフレッシュトークンについて、送信者制約か**ローテーション**（使用ごとに新しいリフレッシュトークンを発行し、旧トークンの再使用を検知したら系列全体を失効）を求めている[^rfc9700]。

失効は失効エンドポイント (RFC 7009) で行う[^rfc7009]。JWT アクセストークンは失効しても `exp` までは RS 側でローカル検証に通ってしまう。そのため、短い `exp` との組み合わせ、または高リスク操作でのイントロスペクション (RFC 7662) が必要になる。

### エラー処理 (Error Handling)

トークンエンドポイントのエラーは HTTP 400（`invalid_client` は 401 になりうる）で、JSON ボディの `error` フィールドで通知される（RFC 6749 §5.2）[^rfc6749]。

| `error` 値 | 意味 |
|------------|------|
| `invalid_request` | パラメータの不足・不正 |
| `invalid_client` | クライアント認証の失敗 |
| `invalid_grant` | 認可コード・リフレッシュトークンが無効/期限切れ、または PKCE 検証失敗 |
| `unauthorized_client` | このグラントタイプの使用が許可されていない |
| `unsupported_grant_type` | サーバーが対応しないグラントタイプ |
| `invalid_scope` | 要求スコープが無効・範囲超過 |
| `invalid_dpop_proof` | DPoP 証明の検証失敗（RFC 9449） |
| `use_dpop_nonce` | DPoP ノンスの使用を要求（RFC 9449） |

リソースサーバー側のエラーは RFC 6750 の `WWW-Authenticate` ヘッダーで返す[^rfc6750]。

| `error` 値 | HTTP ステータス | 意味 |
|------------|-----------------|------|
| `invalid_request` | 400 | リクエスト形式の不正 |
| `invalid_token` | 401 | トークンが無効・期限切れ → クライアントは再取得を試みる |
| `insufficient_scope` | 403 | トークンのスコープ不足 |

クライアント実装は `invalid_token` (401) を「リフレッシュ → リトライ」の契機として扱い、`insufficient_scope` (403) はリトライしても無駄である、と区別するのが要点である。リフレッシュ自体が `invalid_grant` で失敗した場合は、ユーザーの再認証へ戻す。

### OpenID Connect (OIDC)

OIDC は OAuth 2.0 の上に構築されたアイデンティティ層である[^oidc]。OAuth 2.0 のアクセストークンだけで「ログイン」を実装すると、トークンの発行先や発行契機を検証できず脆弱になる（アクセストークンは RS 宛てであり、クライアント宛てではない）。OIDC は、クライアント宛ての検証可能なクレームを持つ ID トークンでこの穴を塞いだものと理解できる。

#### ID トークン (ID Token)

ID トークンは、認証イベントと認証されたユーザーに関するクレームを含む JWT である。ここで 3 仕様が合流する。OIDC のプロトコル（OAuth 2.0 ベース）が JWT 形式のトークンを運ぶ。

| クレーム | 必須性 | 説明 |
|----------|--------|------|
| `iss` | 必須 | 発行者識別子 |
| `sub` | 必須 | サブジェクト識別子（発行者内で一意なユーザー ID） |
| `aud` | 必須 | オーディエンス（クライアント ID を含む） |
| `exp` | 必須 | 有効期限 |
| `iat` | 必須 | 発行時刻 |
| `auth_time` | 条件付き | 認証が実際に行われた時刻（`max_age` 要求時など） |
| `nonce` | 条件付き | 認可要求の `nonce` と照合（リプレイ防止） |
| `acr` | 任意 | 認証コンテキストクラス |
| `amr` | 任意 | 認証方法（パスワード、OTP 等） |
| `azp` | 任意 | 認可された当事者 |

クライアントは `iss` の一致、`aud` に自分のクライアント ID が含まれること、署名、`exp`、`nonce`（送った場合）を検証する。ユーザーの同一性は `email` ではなく `iss` + `sub` の組で判定する。`email` は変更や再割り当てがありうるためである。

#### OIDC スコープ (Scopes)

| スコープ | 返されるクレーム |
|----------|------------------|
| `openid` | `sub`（OIDC リクエストであることの宣言。必須） |
| `profile` | `name`, `family_name`, `given_name`, `nickname`, `picture` など |
| `email` | `email`, `email_verified` |
| `address` | `address` |
| `phone` | `phone_number`, `phone_number_verified` |

### JWT と OAuth 2.0 の比較 (Comparison)

そもそも比較の土俵が違う。JWT は**トークン形式**、OAuth 2.0 は**認可プロトコル**である。「JWT vs OAuth」という問いの多くは、「OAuth のアクセストークンとして JWT を使うか、不透明 (opaque) トークンを使うか」という問いに還元される。

| 観点 | JWT | OAuth 2.0 |
|------|-----|-----------|
| 種別 | トークン形式 | 認可プロトコル |
| 目的 | クレームのエンコードと転送 | アクセスの委任 |
| 状態管理 | ステートレス | 認可サーバーが状態を持つ |
| 失効 | 困難（組み込み機構なし。期限切れを待つか失効リストが必要） | 失効エンドポイント (RFC 7009)、イントロスペクション (RFC 7662) |

- **JWT（形式として）を使う場面**: ステートレス認証、マイクロサービス間の API 認証、サーバー間通信。検証側が発行側に問い合わせずに済むことが利点である。
- **OAuth 2.0 を使う場面**: サードパーティへのアクセス委任、ソーシャルログイン、データアクセスへのユーザー同意、外部アプリへの API 開放。
- **組み合わせる場面（実際には最も多い）**: OAuth の認可サーバーがアクセストークンを JWT 形式 (RFC 9068) で発行する。OIDC の ID トークンは仕様上 JWT である。エンタープライズ認証基盤はほぼこの複合形をとる。

### 標準の成熟度と互換性 (Standards Status and Compatibility)

2026-10-04 時点の状態を以下にまとめる。

| 仕様 | 番号 / 識別子 | 発行 | ステータス |
|------|---------------|------|------------|
| OAuth 2.0 Authorization Framework | RFC 6749 | 2012-10 | Proposed Standard |
| Bearer Token Usage | RFC 6750 | 2012-10 | Proposed Standard |
| PKCE | RFC 7636 | 2015-09 | Proposed Standard |
| JWT | RFC 7519 | 2015-05 | Proposed Standard |
| Token Introspection | RFC 7662 | 2015-10 | Proposed Standard |
| AS Metadata | RFC 8414 | 2018-06 | Proposed Standard |
| Token Exchange | RFC 8693 | 2020-01 | Proposed Standard |
| mTLS / Certificate-Bound Tokens | RFC 8705 | 2020-02 | Proposed Standard |
| JWT BCP | RFC 8725 | 2020-02 | BCP（改訂版 draft-ietf-oauth-rfc8725bis-10 が RFC Editor Queue） |
| JWT Access Token Profile | RFC 9068 | 2021-10 | Proposed Standard |
| PAR | RFC 9126 | 2021-09 | Proposed Standard |
| AS Issuer Identification | RFC 9207 | 2022-03 | Proposed Standard |
| RAR | RFC 9396 | 2023-05 | Proposed Standard |
| DPoP | RFC 9449 | 2023-09 | Proposed Standard |
| OAuth 2.0 Security BCP | RFC 9700 (BCP 240) | 2025-01 | Best Current Practice |
| OAuth 2.1 | draft-ietf-oauth-v2-1-16 | 2026-09-03 | **Internet-Draft**（IESG 提出目標 2026-12） |
| OpenID Connect Core 1.0 | ISO/IEC 26131:2024 | 2024-10（ISO 版） | OpenID Final / ISO 標準 |
| FAPI 2.0 Security Profile | — | 2025-02 | OpenID Final Specification |

互換性に関する要点は次のとおりである。

- **OAuth 2.0 → 2.1**: OAuth 2.1 は RFC 6749/6750 を置き換え・廃止する予定のドラフトである[^oauth21]。主な変更は、(1) 認可コードフローでの PKCE 必須化、(2) インプリシット・パスワードグラントの削除、(3) URL クエリ文字列でのベアラートークン送信の禁止、(4) リダイレクト URI の完全一致、(5) 公開クライアントのリフレッシュトークンの送信者制約または一回限り使用、である。既存の 2.0 実装は、RFC 9700 のプラクティスへ先行して移行しておけば 2.1 と実質的に互換になる。
- **JWT**: RFC 7519 自体は安定しているが、運用指針は RFC 8725 で更新され、さらに改訂版が発行待ちである[^rfc8725][^rfc8725bis]。改訂版が RFC になった時点で参照先を切り替える。
- **OIDC**: OpenID Connect Core 1.0 は errata 更新のみで後方互換が維持されている[^iso26131]。

---

## セキュリティ考察 (Security Considerations)

### JWT の脆弱性 (JWT Vulnerabilities)

#### 1. アルゴリズム混同攻撃 (Algorithm Confusion Attack)

`alg` ヘッダーを検証せずに受け入れるライブラリを悪用する。RS256 のトークンを HS256 に書き換え、**公開鍵を HMAC のシークレットとして**署名すると、検証側が公開鍵を HMAC 鍵として使ってしまい検証が通る[^portswigger]。

```text
# Attack scenario
Original (RS256): signed with private key, verified with public key
Attack   (HS256): attacker signs with the public key (now used as HMAC secret)
```

**対策**: 許可するアルゴリズムをサーバー側で固定（許可リスト化）し、トークンの `alg` ヘッダーを信頼しない。鍵とアルゴリズムを対で管理する（RFC 8725 §3.1）[^rfc8725]。

#### 2. none アルゴリズム攻撃 (None Algorithm Attack)

`alg: none` は「署名なし JWT」を意味する。これを受け入れるライブラリでは署名検証が丸ごとスキップされる。

**対策**: 本番環境では `alg: none` を必ず拒否する。RFC 9068 は JWT アクセストークンについてこれを MUST としている[^rfc9068]。

#### 3. キー ID (`kid`)・`jku`・`x5u` の悪用

ヘッダーの `kid` を鍵の検索にそのまま使う実装は、SQL インジェクションやパストラバーサルに悪用されうる。`jku`（JWK Set URL）や `x5u` を検証なしに辿る実装では、攻撃者が用意した鍵で署名を検証させられる。

**対策**: `kid` は事前登録された鍵集合の中から選ぶ識別子としてのみ扱い、サニタイズする。`jku`/`x5u` は無視するか、許可リストのホストに限定する。鍵の取得元は AS のメタデータ (RFC 8414) の `jwks_uri` に固定する。

#### 4. 型の取り違え (Cross-JWT Confusion)

ある用途で発行された JWT（例: ID トークン）を別の用途（例: API のアクセストークン）として受け入れさせる攻撃である。

**対策**: `typ` による明示的型付け（RFC 9068 の `at+jwt`、DPoP の `dpop+jwt`）と、`aud` の厳格な検証を行う[^rfc8725]。

#### 5. トークン窃取 (Token Theft)

XSS で読める場所（`localStorage` など）に保存されたトークンは窃取される。

**対策**: ブラウザでは、BFF (Backend for Frontend) パターンでトークンをサーバー側に置き、ブラウザには `HttpOnly`・`Secure`・`SameSite` 属性付きのセッション Cookie だけを渡す。SPA 単体で保持する場合は、短い有効期限と DPoP の非抽出鍵で窃取の価値を下げる。Cookie 属性と同一オリジンポリシーの詳細は [`../browsers/web-security-model.md`](../browsers/web-security-model.md) を参照。

### OAuth 2.0 のセキュリティ脅威 (OAuth Threats)

RFC 9700 の脅威モデルに沿って、主な脅威と対策を整理する[^rfc9700]。

| 脅威 | 説明 | 対策 |
|------|------|------|
| CSRF | 攻撃者の認可レスポンスを被害者のセッションに注入 | PKCE（RFC 9700 は PKCE を CSRF 対策として認める）、または一回性の `state` |
| 認可コード横取り・注入 | リダイレクト経路でのコード窃取、窃取コードの別セッションへの注入 | PKCE（S256） |
| オープンリダイレクタ・リダイレクト URI 改竄 | 悪意あるリダイレクト先への誘導 | リダイレクト URI の完全文字列一致（ネイティブアプリの localhost ポートのみ例外） |
| ミックスアップ攻撃 | 複数 AS を使うクライアントを騙し、別 AS のコードを送らせる | 認可レスポンスの `iss` 検証 (RFC 9207) |
| アクセストークン漏洩・リプレイ | ログ・URL・悪意ある RS からの漏洩 | URL に載せない、`aud` 制限、送信者制約 (DPoP / mTLS) |
| リフレッシュトークン窃取 | 長命トークンの悪用 | ローテーションと再使用検知、または送信者制約 |
| 過大な権限 | 広すぎるスコープ・`aud` | 最小権限のスコープ、RAR、RS ごとの `aud` 制限 |
| クリックジャッキング | 同意画面を透明フレームで覆う | `X-Frame-Options` / CSP `frame-ancestors` |
| 認可要求の改竄 | フロントチャネルでのパラメータ書き換え | PAR (RFC 9126) |

### 実装チェックリスト (Implementation Checklist)

旧版のベストプラクティス節を、上記の脅威に対応づけて整理し直したものである。

**トークン形式と検証 (JWT)**

- 許可するアルゴリズムを許可リスト化する。検証者が複数いる構成では ES256 / PS256 / RS256（非対称鍵）を使い、`none` を拒否する。
- 受信側で `iss` の一致、`aud`（自分宛てであること）、`exp`・`nbf`（有効期間内であること）、`typ`（期待する型）を必ず検証する。時計のずれは数十秒程度の許容幅に限る。
- 署名鍵は十分な強度（HMAC なら 256 ビット以上のランダム鍵、RSA なら 2048 ビット以上）とし、定期的にローテーションする。`kid` で新旧鍵を併存させ、JWKS で公開する。秘密鍵は KMS / HSM で管理し、環境ごとに分離する。

**フローとクライアント (OAuth)**

- すべてのクライアントで認可コード + PKCE (`S256`) を使う。`code_verifier` は暗号論的に安全な乱数で生成する。インプリシットフロー・パスワードグラントは使わない。
- リダイレクト URI は完全一致で検証し、HTTPS を必須にする。
- 複数 AS を扱うクライアントは認可レスポンスの `iss` を検証する。
- 機密クライアントの認証には `private_key_jwt` または mTLS を優先する。
- 高保証用途では PAR を使い、FAPI 2.0 Security Profile への準拠を検討する。

**トークンの扱い**

- アクセストークンは短命にし、RS ごとに `aud` を絞る。
- リフレッシュトークンはローテーションし、再使用を検知したら系列全体を失効させる。
- 可能なら DPoP / mTLS で送信者制約をかける。
- ブラウザアプリでは BFF パターンを第一候補にする。`localStorage` にトークンを保存しない。
- OIDC では `nonce` で ID トークンのリプレイを防ぎ、ユーザー識別には `iss` + `sub` を使う。

### 推奨ライブラリ (Recommended Libraries)

トークン処理は自作せず、実績のあるライブラリを使う。以下は広く使われている例であり、採用時にはメンテナンス状況と、DPoP・RFC 9068 など必要な機能への対応を確認する。

| 言語 | JWT / JOSE | OAuth/OIDC |
|------|------------|------------|
| Node.js | `jose` | `openid-client` |
| Python | `PyJWT`, `joserfc` | `Authlib` |
| Java | `nimbus-jose-jwt` | Spring Security OAuth2 |
| Go | `golang-jwt/jwt`, `go-jose` | `golang.org/x/oauth2`, `coreos/go-oidc` |
| Rust | `jsonwebtoken` | `oauth2`, `openidconnect` |

依存ライブラリ自体の改竄リスクと来歴検証については [`../security/software-supply-chain-security.md`](../security/software-supply-chain-security.md) を参照。

### インシデント対応 (Incident Response)

トークン漏洩時の対応手順は次のとおりである。

1. 漏洩したトークンを直ちに失効させる（RFC 7009 の失効エンドポイント。リフレッシュトークンを失効させると関連アクセストークンも失効する実装が多い）。
2. 署名用秘密鍵が漏洩した場合は署名鍵をローテーションし、JWKS から旧鍵を削除する。JWT では鍵の失効が全トークンの失効を意味する点に注意する。
3. 影響を受けたユーザーに再認証を強制する。
4. アクセスログ（`jti`・`client_id`・`sub`）で不正アクセスの有無を確認する。
5. 送信者制約が未導入なら導入し、再発時の被害を抑える。

---

## 結論 (Conclusions)

JWT・OAuth 2.0・OIDC の 10 年余りの進化は、一貫して**攻撃面の削減**の歴史として読める。インプリシットと ROPC は削除され、PKCE は全クライアントに広がった。リダイレクト URI は完全一致となり、認可要求は PAR でフロントチャネルから外れつつある。RFC 9700 がこの方向を BCP として固定し、OAuth 2.1 ドラフトがそれを本体仕様に取り込もうとしている。2026-10-04 時点で OAuth 2.1 はまだ Internet-Draft だが、準拠すべき実務上の基準は RFC 9700 としてすでに確定している。

残る最大の構造的リスクは「ベアラー性」である。ベアラートークンは持ち主を区別しないため、どこか 1 か所から漏れれば十分に悪用できる。DPoP と mTLS はこの性質を「鍵の所持証明」に置き換えるもので、短命トークン・`aud` 制限・トークン交換と組み合わせることで、漏洩時の被害範囲 (blast radius) を時間・宛先・鍵の 3 軸で絞れる。

JWT 固有の脆弱性は、ほぼすべてが「トークン自身が名乗る情報（`alg`, `kid`, `jku`, `typ`）を信頼する」ことに起因する。検証側が期待値（アルゴリズム・鍵・型・宛先）を先に決め、トークンをそれに照合するという原則を守れば、既知の攻撃の大半は構造的に防げる。設計判断の要点をまとめると、(1) RFC 9700 を基準線とする、(2) 新規実装は OAuth 2.1 ドラフトの制約を先取りする、(3) 高価値 API では送信者制約を必須にする、(4) 標準の成熟度（RFC / Draft / Final）を区別して追跡する、の 4 点である。

---

## 付録 A: トークン例 (Appendix A: Token Examples)

### JWT の例

```text
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.POstGetfAytaZS82wHcjoTyoqhMyxXiWdR7Nn7A29DNSl0EiXLdwJ6xC6AfgZWF1bOsS_TuYI3OG85AmiExREkrS6tDfTQ2B3WXlrr-wp5AokiRbz3_oB4OxG-W9KcEEbDRcZc0nH3L7LzYptiy1PtAylQGxHTWZXtGz4ht0bAecBgmpdgXMguEIcoqPJ1n3pIWk_dUZegpqx0Lka21H6XxUTxiy8OcaarA8zdnPUnV6AmNP3ecFawIFYdvJB_cm-GvpCSbr8G8y_Mllj8f4x9nBH8pQux89_6gUY618iYv7tuPWBFfEbLxtF2pZS6YC1aSfLQxeNe8djT9YjpvRZA
```

デコードされたヘッダーとペイロードは次のとおりである（説明用の公開サンプルであり、`exp`・`aud` を欠くため本番の検証には通らないのが正しい挙動である）。

```json
{ "alg": "RS256", "typ": "JWT" }
```

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "iat": 1516239022
}
```

### RFC 9068 準拠アクセストークンのペイロード例

```json
{
  "iss": "https://as.example.com/",
  "sub": "5ba552d67",
  "aud": "https://api.example.com/",
  "client_id": "s6BhdRkqt3",
  "exp": 1790000000,
  "iat": 1789996400,
  "jti": "dbe39bf3a3ba4238a513f51d6e1691c4",
  "scope": "orders:read",
  "cnf": { "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I" }
}
```

### DPoP 束縛トークンレスポンスの例

```json
{
  "access_token": "Kz~8mXK1EalYznwH-LC-1fBAo.4Ljp~zsPE_NeO.gxU",
  "token_type": "DPoP",
  "expires_in": 600,
  "refresh_token": "Q..Zkm29lexi8VnWg2zPW1x-tgGad0Ibc3s3EwM_Ni4-g",
  "scope": "openid profile orders:read"
}
```

---

## 参考文献 (References)

**一次資料 (Primary — RFC・標準・Internet-Draft)**

[^rfc6749]: Hardt, D. (Ed.) (2012). The OAuth 2.0 Authorization Framework. *RFC 6749*. https://www.rfc-editor.org/rfc/rfc6749 — Accessed: 2026-10-04.

[^rfc6750]: Jones, M., & Hardt, D. (2012). The OAuth 2.0 Authorization Framework: Bearer Token Usage. *RFC 6750*. https://www.rfc-editor.org/rfc/rfc6750 — Accessed: 2026-10-04.

[^rfc7009]: Lodderstedt, T. (Ed.), Dronia, S., & Scurtescu, M. (2013). OAuth 2.0 Token Revocation. *RFC 7009*. https://www.rfc-editor.org/rfc/rfc7009 — Accessed: 2026-10-04.

[^rfc7515]: Jones, M., Bradley, J., & Sakimura, N. (2015). JSON Web Signature (JWS). *RFC 7515*. https://www.rfc-editor.org/rfc/rfc7515 — Accessed: 2026-10-04.

[^rfc7518]: Jones, M. (2015). JSON Web Algorithms (JWA). *RFC 7518*. https://www.rfc-editor.org/rfc/rfc7518 — Accessed: 2026-10-04.

[^rfc7519]: Jones, M., Bradley, J., & Sakimura, N. (2015). JSON Web Token (JWT). *RFC 7519*. https://www.rfc-editor.org/rfc/rfc7519 — Accessed: 2026-10-04.

[^rfc7636]: Sakimura, N. (Ed.), Bradley, J., & Agarwal, N. (2015). Proof Key for Code Exchange by OAuth Public Clients. *RFC 7636*. https://www.rfc-editor.org/rfc/rfc7636 — Accessed: 2026-10-04.

[^rfc7662]: Richer, J. (Ed.) (2015). OAuth 2.0 Token Introspection. *RFC 7662*. https://www.rfc-editor.org/rfc/rfc7662 — Accessed: 2026-10-04.

[^rfc8414]: Jones, M., Sakimura, N., & Bradley, J. (2018). OAuth 2.0 Authorization Server Metadata. *RFC 8414*. https://www.rfc-editor.org/rfc/rfc8414 — Accessed: 2026-10-04.

[^rfc8693]: Jones, M., Nadalin, A., Campbell, B. (Ed.), Bradley, J., & Mortimore, C. (2020). OAuth 2.0 Token Exchange. *RFC 8693*. https://www.rfc-editor.org/rfc/rfc8693 — Accessed: 2026-10-04.

[^rfc8705]: Campbell, B., Bradley, J., Sakimura, N., & Lodderstedt, T. (2020). OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens. *RFC 8705*. https://www.rfc-editor.org/rfc/rfc8705 — Accessed: 2026-10-04.

[^rfc8725]: Sheffer, Y., Hardt, D., & Jones, M. (2020). JSON Web Token Best Current Practices. *RFC 8725* (BCP 225). https://www.rfc-editor.org/rfc/rfc8725 — Accessed: 2026-10-04.

[^rfc8725bis]: IETF OAuth WG. *JSON Web Token Best Current Practices* (draft-ietf-oauth-rfc8725bis-10, 2026-08-21). https://datatracker.ietf.org/doc/draft-ietf-oauth-rfc8725bis/ — Accessed: 2026-10-04. — RFC Editor Queue。RFC 8725 を廃止、RFC 7519 を更新予定（ドラフトのため内容は変わりうる）。

[^rfc9068]: Bertocci, V. (2021). JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens. *RFC 9068*. https://www.rfc-editor.org/rfc/rfc9068 — Accessed: 2026-10-04.

[^rfc9126]: Lodderstedt, T., Campbell, B., Sakimura, N., Tonge, D., & Skokan, F. (2021). OAuth 2.0 Pushed Authorization Requests. *RFC 9126*. https://www.rfc-editor.org/rfc/rfc9126 — Accessed: 2026-10-04.

[^rfc9207]: Meyer zu Selhausen, K., & Fett, D. (2022). OAuth 2.0 Authorization Server Issuer Identification. *RFC 9207*. https://www.rfc-editor.org/rfc/rfc9207 — Accessed: 2026-10-04.

[^rfc9396]: Lodderstedt, T., Richer, J., & Campbell, B. (2023). OAuth 2.0 Rich Authorization Requests. *RFC 9396*. https://www.rfc-editor.org/rfc/rfc9396 — Accessed: 2026-10-04.

[^rfc9449]: Fett, D., Campbell, B., Bradley, J., Lodderstedt, T., Jones, M., & Waite, D. (2023). OAuth 2.0 Demonstrating Proof of Possession (DPoP). *RFC 9449*. https://www.rfc-editor.org/rfc/rfc9449 — Accessed: 2026-10-04.

[^rfc9700]: Lodderstedt, T., Bradley, J., Labunets, A., & Fett, D. (2025). Best Current Practice for OAuth 2.0 Security. *RFC 9700* (BCP 240). https://www.rfc-editor.org/rfc/rfc9700 — Accessed: 2026-10-04.

[^oauth21]: Hardt, D., Parecki, A., & Lodderstedt, T. *The OAuth 2.1 Authorization Framework* (draft-ietf-oauth-v2-1-16, 2026-09-03). https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ — Accessed: 2026-10-04. — Active Internet-Draft。WG マイルストーンは 2026-12 に IESG 提出（ドラフトのため内容は変わりうる）。

[^oidc]: Sakimura, N., Bradley, J., Jones, M., de Medeiros, B., & Mortimore, C. *OpenID Connect Core 1.0 incorporating errata set 2*. OpenID Foundation. https://openid.net/specs/openid-connect-core-1_0.html — Accessed: 2026-10-04.

[^iso26131]: ISO/IEC. *ISO/IEC 26131:2024 Information technology — OpenID connect — OpenID connect core 1.0 incorporating errata set 2*. https://www.iso.org/standard/89056.html — Accessed: 2026-10-04.

[^fapi2]: OpenID Foundation. *FAPI 2.0 Security Profile* (Final, 2025-02). https://openid.net/specs/fapi-security-profile-2_0-final.html — Accessed: 2026-10-04. — 承認告知: https://openid.net/fapi-2-security-profile-attacker-model-final-specifications-approved/

**二次資料 (Secondary — 攻撃手法の解説)**

[^portswigger]: PortSwigger Web Security Academy. *JWT attacks* / *Algorithm confusion attacks*. https://portswigger.net/web-security/jwt — Accessed: 2026-10-04. — 攻撃手順の具体例の確認に使用（規範は RFC 8725 に帰属）。
