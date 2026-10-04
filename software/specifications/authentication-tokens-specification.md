# 認証トークン仕様書 (Authentication Token Specification) — JWT・OAuth 2.0・OIDC

**Version**: 2.0
**Last Updated**: 2026-07-06
**Author**: Claude @ Daedalus

---

## 概要 (Overview)

本ドキュメントは、JWT (JSON Web Token)、OAuth 2.0、および OpenID Connect (OIDC) を中心とした、現代の認証 (authentication)・認可 (authorization) メカニズムの技術仕様を分析する。これらの技術は、Web アプリケーションおよび API におけるアイデンティティ管理の基盤を構成する。

3 つの関係を先に一言でまとめると、**JWT はトークンの「形式」、OAuth 2.0 はアクセス委任の「プロトコル」、OIDC は OAuth 2.0 の上に認証を載せた「アイデンティティ層」** である。本書では各仕様のデータ構造・プロトコルフロー・エラー処理・バージョン互換性を扱い、セキュリティ考察とベストプラクティスで締める。

### 主要用語 (Key Terminology)

| 用語 | 英語 | 説明 |
|------|------|------|
| 認証 | Authentication | ユーザーやシステムの身元を検証すること |
| 認可 | Authorization | リソースへのアクセス権限を付与すること |
| トークン | Token | 認証・認可の状態を表現するデータ |
| クレーム | Claim | エンティティ（通常はユーザー）に関する記述 |
| スコープ | Scope | トークンが許可するアクセスの範囲 |

---

## JWT (JSON Web Token)

JWT は、二者間で転送されるクレームを表現するための、コンパクトで URL セーフなトークン形式である ([RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519))。JSON 形式の情報（クレーム）を自己完結的に含み、デジタル署名によって完全性 (integrity) を保証する。

### 構造 (Structure)

JWT はピリオド (`.`) で区切られた 3 部分から構成される。

```
xxxxx.yyyyy.zzzzz
[Header].[Payload].[Signature]
```

#### 1. ヘッダー (Header)

トークンのタイプ (JWT) と署名アルゴリズムを指定する。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

#### 2. ペイロード (Payload)

クレームを格納する。RFC 7519 が定義する登録済みクレーム (registered claims) は以下の通り。

| クレーム | 名称 | 説明 |
|----------|------|------|
| `iss` | Issuer | トークン発行者 |
| `sub` | Subject | トークンの対象者 |
| `aud` | Audience | 想定される受信者 |
| `exp` | Expiration Time | 有効期限（UNIX タイムスタンプ） |
| `nbf` | Not Before | この時刻より前は無効 |
| `iat` | Issued At | トークン発行時刻 |
| `jti` | JWT ID | 一意識別子（リプレイ検出などに利用） |

ペイロード例を示す。

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622,
  "role": "admin"
}
```

#### 3. 署名 (Signature)

エンコード済みヘッダー・エンコード済みペイロード・鍵から、ヘッダーで指定されたアルゴリズムで生成する。

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

なお、ヘッダーとペイロードは Base64URL エンコードされているだけで**暗号化はされていない**。署名が保証するのは完全性であり、機密性 (confidentiality) ではない。秘匿が必要なら JWE (JSON Web Encryption) か、そもそもペイロードに機密情報を入れない設計が必要である。

### サポートされるアルゴリズム (Supported Algorithms)

署名アルゴリズムは JWA ([RFC 7518](https://datatracker.ietf.org/doc/html/rfc7518)) で定義される。

| アルゴリズム | 種別 | 説明 |
|--------------|------|------|
| HS256 / HS384 / HS512 | 対称鍵 (symmetric) | HMAC + SHA-256/384/512（共有シークレット） |
| RS256 / RS384 / RS512 | 非対称鍵 (asymmetric) | RSA 署名 + SHA-256/384/512 |
| ES256 / ES384 / ES512 | 非対称鍵 | ECDSA（P-256/P-384/P-521） + SHA-256/384/512 |
| `none` | なし | 署名なし JWT（**危険** — 本番では必ず拒否する） |

対称鍵 (HS*) は発行者と検証者が同一シークレットを共有するためマイクロサービス間で鍵配布の問題を生む。検証者が多数いる構成では、公開鍵で検証できる非対称鍵 (RS*/ES*) を選ぶ。

### 認証フロー (Authentication Flow)

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Auth as 認証サーバー
    participant API as リソースサーバー

    Client->>Auth: 1. ログイン (credentials)
    Auth->>Auth: 2. 資格情報を検証
    Auth->>Client: 3. JWT を返却
    Client->>API: 4. リクエスト + JWT (Authorization: Bearer)
    API->>API: 5. JWT 署名を検証
    API->>Client: 6. 保護されたリソースを返却
```

### 特徴 (Characteristics)

| 特徴 | 説明 |
|------|------|
| ステートレス (stateless) | サーバー側のセッションストレージが不要 |
| 自己完結 (self-contained) | 検証に必要な情報をトークン自身が含む |
| コンパクト | サイズが小さく、HTTP ヘッダーや URL に載せやすい |
| 検証可能 | 署名により改竄を検出できる |

裏返せば、自己完結ゆえに**発行後の失効 (revocation) が難しい**のが JWT の構造的弱点である（[比較](#jwt-と-oauth-20-の比較-comparison)参照）。

---

## OAuth 2.0

OAuth 2.0 は、サードパーティアプリケーションが HTTP サービスへの**限定的なアクセス**を取得することを可能にする認可フレームワークである ([RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749))。ユーザーのパスワードをアプリケーションに渡すことなく、スコープを絞ったアクセスを委任できる点が本質である。

### 役割 (Roles)

```mermaid
graph LR
    RO["リソースオーナー<br/>(Resource Owner)<br/>エンドユーザー"]
    C["クライアント<br/>(Client)<br/>アプリケーション"]
    AS["認可サーバー<br/>(Authorization Server)"]
    RS["リソースサーバー<br/>(Resource Server)"]

    RO -- "1. アクセスを認可" --> AS
    C -- "2. トークンを要求" --> AS
    AS -- "3. トークンを発行" --> C
    C -- "4. トークンを提示" --> RS
```

| 役割 | 説明 |
|------|------|
| リソースオーナー (Resource Owner) | アクセスを許可できるエンティティ（通常はエンドユーザー） |
| クライアント (Client) | アクセスを要求するアプリケーション |
| 認可サーバー (Authorization Server) | 認証後にトークンを発行するサーバー |
| リソースサーバー (Resource Server) | 保護されたリソースをホストするサーバー |

### 認可タイプ (Grant Types)

#### 1. 認可コードグラント (Authorization Code Grant)

最も安全なフローで、一時的な認可コード (authorization code) をトークンと交換する。トークンがブラウザを経由しない点が要である。

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Client as クライアントアプリ
    participant Auth as 認可サーバー
    participant Resource as リソースサーバー

    User->>Client: 1. アプリケーションにアクセス
    Client->>Auth: 2. 認可エンドポイントへリダイレクト
    Auth->>User: 3. ログイン・同意画面
    User->>Auth: 4. 許可
    Auth->>Client: 5. 認可コード付きでリダイレクト
    Client->>Auth: 6. コードをトークンと交換
    Auth->>Client: 7. アクセストークン (+ リフレッシュトークン)
    Client->>Resource: 8. アクセストークンでリクエスト
    Resource->>Client: 9. 保護されたリソースを返却
```

#### 2. 認可コード + PKCE (Authorization Code with PKCE)

PKCE (Proof Key for Code Exchange, [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)) は認可コードフローの拡張で、認可コードの横取り (interception) を防ぐ。現在は公開クライアントに限らず**すべてのクライアントに推奨**されており、OAuth 2.1 ドラフトでは必須化される方向である。

手順は次のとおりである。

```
# 1. Generate a random code_verifier
code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r..."

# 2. Derive the code_challenge
code_challenge = BASE64URL(SHA256(code_verifier))

# 3. Send code_challenge with the authorization request
GET /authorize?...&code_challenge=E9Melhoa...&code_challenge_method=S256

# 4. Exchange the code together with the original code_verifier
POST /token
  ...&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r...

# 5. Server verifies: SHA256(code_verifier) == code_challenge
```

認可コードを盗んでも `code_verifier` を知らなければトークンに交換できない、という一回性の証明である。

#### 3. クライアントクレデンシャルグラント (Client Credentials Grant)

ユーザーが関与しないマシン間 (M2M) 通信に使用する。

```
POST /token HTTP/1.1
Host: authorization-server.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=xxxxxxxxxx
&client_secret=yyyyyyyyyy
&scope=read:data
```

#### 4. インプリシットグラント (Implicit Grant) — 非推奨

ブラウザベースアプリ向けに設計されたが、トークンが URL フラグメントに露出する。OAuth 2.0 Security BCP ([RFC 9700](https://datatracker.ietf.org/doc/html/rfc9700)) で非推奨とされ、OAuth 2.1 ドラフトでは**削除**される。認可コード + PKCE を使用すること。

#### 5. リソースオーナーパスワードクレデンシャル (Resource Owner Password Credentials) — 非推奨

ユーザーの資格情報そのものをクライアントに渡すフローで、OAuth が本来解決したかった問題への逆行である。同じく RFC 9700 で非推奨、OAuth 2.1 ドラフトで削除される。レガシーシステムの移行期間に限る。

### トークンタイプ (Token Types)

| トークン | 目的 | 有効期間 |
|----------|------|----------|
| アクセストークン (access token) | 保護されたリソースへのアクセス | 短命（分〜時間単位） |
| リフレッシュトークン (refresh token) | 新しいアクセストークンの取得 | 長命（日〜月単位） |

アクセストークンを短命にして漏洩時の被害を限定し、長いセッションはリフレッシュトークン（+ ローテーション）で維持するのが基本形である。失効は失効エンドポイント ([RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009)) で行う。

### エラー処理 (Error Handling)

トークンエンドポイントのエラーは HTTP 400（`invalid_client` は 401）で、JSON ボディの `error` フィールドにより通知される (RFC 6749 §5.2)。

| `error` 値 | 意味 |
|------------|------|
| `invalid_request` | パラメータの不足・不正 |
| `invalid_client` | クライアント認証の失敗 |
| `invalid_grant` | 認可コード・リフレッシュトークンが無効/期限切れ |
| `unauthorized_client` | このグラントタイプの使用が許可されていない |
| `unsupported_grant_type` | サーバーが対応しないグラントタイプ |
| `invalid_scope` | 要求スコープが無効・範囲超過 |

リソースサーバー側のエラーは Bearer トークン仕様 ([RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750)) の `WWW-Authenticate` ヘッダーで返す。

| `error` 値 | HTTP ステータス | 意味 |
|------------|-----------------|------|
| `invalid_request` | 400 | リクエスト形式の不正 |
| `invalid_token` | 401 | トークンが無効・期限切れ → クライアントは再取得を試みる |
| `insufficient_scope` | 403 | トークンのスコープ不足 |

クライアント実装は `invalid_token` (401) を「リフレッシュ → リトライ」の契機として扱い、`insufficient_scope` (403) はリトライしても無駄である、という区別が要点になる。

---

## OpenID Connect (OIDC)

OpenID Connect は OAuth 2.0 の上に構築された**アイデンティティ層**である ([OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html))。ID トークン (ID Token) を導入することで、認可のフレームワークである OAuth に「ユーザーが誰であるか」の認証機能を追加する。

```mermaid
graph TB
    OIDC["OpenID Connect<br/>認証 (Authentication)"]
    OAuth["OAuth 2.0<br/>認可 (Authorization)"]
    HTTP["HTTP"]
    OIDC --> OAuth --> HTTP
```

OAuth 2.0 のアクセストークンだけで「ログイン」を実装すると、トークンの発行先・発行契機を検証できず脆弱になる。OIDC はこの穴を、検証可能なクレームを持つ ID トークンで塞いだものと理解できる。

### ID トークン (ID Token)

ID トークンは、認証イベントと認証されたユーザーに関するクレームを含む **JWT** である。ここで 3 仕様が合流する: OIDC のプロトコル (OAuth 2.0 ベース) が JWT 形式のトークンを運ぶ。

| クレーム | 必須性 | 説明 |
|----------|--------|------|
| `iss` | 必須 | 発行者識別子 |
| `sub` | 必須 | サブジェクト識別子（発行者内で一意なユーザー ID） |
| `aud` | 必須 | オーディエンス（クライアント ID） |
| `exp` | 必須 | 有効期限 |
| `iat` | 必須 | 発行時刻 |
| `auth_time` | 条件付き | 認証が実際に行われた時刻 |
| `nonce` | 条件付き | セッション関連付け文字列（リプレイ防止） |
| `acr` | 任意 | 認証コンテキストクラス |
| `amr` | 任意 | 認証方法（パスワード、OTP 等） |
| `azp` | 任意 | 認可された当事者 |

### OIDC スコープ (Scopes)

| スコープ | 返されるクレーム |
|----------|------------------|
| `openid` | `sub`（OIDC リクエストであることの宣言。必須） |
| `profile` | `name`, `family_name`, `given_name`, `nickname`, `picture` など |
| `email` | `email`, `email_verified` |
| `address` | `address` |
| `phone` | `phone_number`, `phone_number_verified` |

---

## JWT と OAuth 2.0 の比較 (Comparison)

### 根本的な違い (Fundamental Difference)

そもそも比較の土俵が違う——JWT は**トークン形式**、OAuth 2.0 は**認可プロトコル**である。「JWT vs OAuth」という問いの多くは「OAuth のアクセストークンとして JWT を使うか、不透明 (opaque) トークンを使うか」という問いに還元される。

| 観点 | JWT | OAuth 2.0 |
|------|-----|-----------|
| 種別 | トークン形式 | 認可プロトコル |
| 目的 | クレームのエンコードと転送 | アクセスの委任 |
| 状態管理 | ステートレス | 認可サーバーが状態を持つ |
| 失効 | 困難（組み込み機構なし。有効期限切れを待つか失効リストが必要） | 失効エンドポイントあり (RFC 7009) |

### 使い分け (When to Use)

- **JWT（形式として）を使う場面**: ステートレス認証、マイクロサービス間の API 認証、サーバー間通信。検証側が発行側に問い合わせずに済むことが利点。
- **OAuth 2.0 を使う場面**: サードパーティへのアクセス委任、ソーシャルログイン、データアクセスへのユーザー同意、外部アプリへの API 開放。
- **組み合わせる場面（実際には最も多い）**: OAuth の認可サーバーがアクセストークンを JWT 形式で発行する。OIDC の ID トークンは仕様上 JWT である。エンタープライズ認証基盤はほぼこの複合形をとる。

### バージョン互換性 (Version Compatibility)

- **OAuth 2.0 → 2.1**: OAuth 2.1 は 2026 年時点でドラフト ([draft-ietf-oauth-v2-1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)) だが、方向性は確定している——RFC 6749/6750 と Security BCP (RFC 9700) の統合であり、新機能の追加ではなく**危険な選択肢の削除**である。主な変更: (1) 認可コードフローでの PKCE 必須化、(2) インプリシット・パスワードグラントの削除、(3) URL クエリ文字列でのベアラートークン送信の禁止、(4) リフレッシュトークンの送信者制約または一回限り使用。既存の 2.0 実装は、これらのプラクティスへ先行して移行しておけば 2.1 と互換になる。
- **JWT**: RFC 7519 自体は安定しているが、運用指針は JWT BCP ([RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725)) で更新されており、`alg` 検証の厳格化など本書のベストプラクティス節の内容が標準化されている。
- **OIDC**: OpenID Connect Core 1.0 は errata 更新のみで後方互換が維持されている。

---

## セキュリティ考察 (Security Considerations)

### JWT の脆弱性 (JWT Vulnerabilities)

#### 1. アルゴリズム混同攻撃 (Algorithm Confusion Attack)

`alg` ヘッダーを検証せずに受け入れるライブラリを悪用する。RS256 のトークンを HS256 に書き換え、**公開鍵を HMAC のシークレットとして**署名すると、検証側が公開鍵を HMAC 鍵として使ってしまい検証が通る。

```
# Attack scenario
Original (RS256): signed with private key, verified with public key
Attack   (HS256): attacker signs with the public key (now used as HMAC secret)
```

**対策**: 許可するアルゴリズムをサーバー側で固定（ホワイトリスト化）し、トークンの `alg` ヘッダーを信頼しない。鍵とアルゴリズムを対で管理する。

#### 2. none アルゴリズム攻撃 (None Algorithm Attack)

`alg: none` は「署名なし JWT」を意味し、これを受け入れるライブラリでは署名検証が丸ごとスキップされる。

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

**対策**: 本番環境では `alg: none` を必ず拒否し、受け入れるアルゴリズムを明示的に設定する。

#### 3. キー ID (`kid`) インジェクション

ヘッダーの `kid` パラメータを鍵の検索にそのまま使う実装では、SQL インジェクションやパストラバーサルに悪用されうる。

**対策**: `kid` 値をサニタイズ・検証し、ファイルパスではなく UUID 等の不透明な識別子を使う。

#### 4. トークン窃取 (Token Theft)

安全でない方法（XSS に読める場所）で保存された JWT は窃取される。

**対策**: HttpOnly Cookie への保存、短い有効期限、トークン更新メカニズムの実装。

### OAuth 2.0 のセキュリティ脅威 (Security Threats)

| 脅威 | 説明 | 対策 |
|------|------|------|
| CSRF | 認可レスポンスのクロスサイトリクエストフォージェリ | `state` パラメータ |
| 認可コード横取り | リダイレクト経路でのコード窃取 | PKCE |
| オープンリダイレクタ | 悪意のあるリダイレクト URI への誘導 | リダイレクト URI の完全一致検証 |
| トークン漏洩 | ログ・URL でのトークン露出 | トークン交換は POST で行い、URL に載せない |

これらの脅威と対策の体系的な整理は OAuth 2.0 Security BCP ([RFC 9700](https://datatracker.ietf.org/doc/html/rfc9700)) にまとまっている。

---

## ベストプラクティス (Best Practices)

### JWT セキュリティチェックリスト

- **アルゴリズム**: 許可するアルゴリズムをホワイトリスト化する。検証者が複数いる構成では RS256 または ES256（非対称鍵）を使う。`none` を拒否する。
- **有効期間**: アクセストークンは短命にする（15 分〜1 時間）。長いセッションにはリフレッシュトークンを使い、ローテーションを実装する。
- **クレーム検証**: 受信側で `iss`（発行者）の一致、`aud`（自分宛てであること）、`exp`・`nbf`（有効期間内であること）を必ず検証する。
- **ストレージ**: HttpOnly Cookie を使用し、`Secure` フラグ（HTTPS のみ）と `SameSite` 属性を設定する。機密性の高いアプリでは localStorage に保存しない（XSS で読み取られる）。
- **鍵管理**: 強力な鍵（最小 256 ビット）を使い、定期的にローテーションする。シークレットマネージャーで管理し、環境ごとに鍵を分離する。

### OAuth 2.0 セキュリティチェックリスト

- **フロー選択**: すべてのクライアントで認可コード + PKCE を使う。インプリシットフロー・パスワードグラントを使わない。
- **PKCE 設定**: チャレンジメソッドは `S256` を使う（`plain` は不可）。`code_verifier` は暗号論的に安全な乱数で生成する。
- **リダイレクト URI**: 完全一致で検証する（ワイルドカード禁止）。HTTPS を必須にする。
- **トークンセキュリティ**: 短命のアクセストークンとリフレッシュトークンローテーションを使う。可能なら DPoP / mTLS でトークンをクライアントにバインドする（送信者制約）。
- **状態管理**: CSRF 保護に `state` パラメータを、リプレイ保護に `nonce`（OIDC）を使う。

### 推奨ライブラリ (Recommended Libraries)

トークン処理は自作せず、実績のあるライブラリを使うこと。

| 言語 | JWT | OAuth/OIDC |
|------|-----|------------|
| Node.js | `jsonwebtoken`, `jose` | `passport`, `openid-client` |
| Python | `PyJWT`, `python-jose` | `Authlib`, `oauthlib` |
| Java | `jjwt`, `nimbus-jose-jwt` | Spring Security OAuth |
| Go | `golang-jwt/jwt` | `golang.org/x/oauth2` |
| Rust | `jsonwebtoken` | `oauth2` |

---

## 付録 A: トークン例 (Appendix A: Token Examples)

### JWT の例

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.POstGetfAytaZS82wHcjoTyoqhMyxXiWdR7Nn7A29DNSl0EiXLdwJ6xC6AfgZWF1bOsS_TuYI3OG85AmiExREkrS6tDfTQ2B3WXlrr-wp5AokiRbz3_oB4OxG-W9KcEEbDRcZc0nH3L7LzYptiy1PtAylQGxHTWZXtGz4ht0bAecBgmpdgXMguEIcoqPJ1n3pIWk_dUZegpqx0Lka21H6XxUTxiy8OcaarA8zdnPUnV6AmNP3ecFawIFYdvJB_cm-GvpCSbr8G8y_Mllj8f4x9nBH8pQux89_6gUY618iYv7tuPWBFfEbLxtF2pZS6YC1aSfLQxeNe8djT9YjpvRZA
```

デコードされたヘッダーは次のとおりである。

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

デコードされたペイロードは次のとおりである。

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "iat": 1516239022
}
```

### OAuth 2.0 トークンレスポンスの例

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "scope": "openid profile email"
}
```

---

## 付録 B: セキュリティインシデント対応 (Appendix B: Security Incident Response)

トークン漏洩時の対応手順は次のとおりである。

1. 漏洩したトークンを直ちに失効させる（OAuth の場合は失効エンドポイント）
2. 署名用秘密鍵が漏洩した場合は署名鍵をローテーションする（JWT は鍵の失効 = 全トークンの失効になる点に注意）
3. 影響を受けたユーザーに再認証を強制する
4. アクセスログで不正アクセスの有無を確認する
5. 追加の監視を実装する

---

## 参考文献 (References)

**一次資料 (Primary — RFC・標準)**

- Jones, M., Bradley, J., & Sakimura, N. (2015). JSON Web Token (JWT). *RFC 7519*. — [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc7519) — Accessed: 2026-07-06
- Hardt, D. (2012). The OAuth 2.0 Authorization Framework. *RFC 6749*. — [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc6749) — Accessed: 2026-07-06
- Jones, M., & Hardt, D. (2012). The OAuth 2.0 Authorization Framework: Bearer Token Usage. *RFC 6750*. — [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc6750) — Accessed: 2026-07-06
- Lodderstedt, T., Dronia, S., & Scurtescu, M. (2013). OAuth 2.0 Token Revocation. *RFC 7009*. — [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc7009) — Accessed: 2026-07-06
- Jones, M. (2015). JSON Web Algorithms (JWA). *RFC 7518*. — [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc7518) — Accessed: 2026-07-06
- Sakimura, N., Bradley, J., & Agarwal, N. (2015). Proof Key for Code Exchange by OAuth Public Clients. *RFC 7636*. — [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc7636) — Accessed: 2026-07-06
- Sheffer, Y., Hardt, D., & Jones, M. (2020). JSON Web Token Best Current Practices. *RFC 8725*. — [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc8725) — Accessed: 2026-07-06
- Lodderstedt, T., Bradley, J., Labunets, A., & Fett, D. (2025). Best Current Practice for OAuth 2.0 Security. *RFC 9700*. — [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc9700) — Accessed: 2026-07-06
- Sakimura, N., Bradley, J., Jones, M., de Medeiros, B., & Mortimore, C. (2014). [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html). OpenID Foundation. — Accessed: 2026-07-06
- [The OAuth 2.1 Authorization Framework (draft-ietf-oauth-v2-1)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1) — Accessed: 2026-07-06（ドラフト。版数・内容は更新されうる）

**二次資料 (Secondary — 解説・セキュリティリソース)**

- [OWASP OAuth 2.0 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html) — Accessed: 2026-07-06
- [OWASP JWT Cheat Sheet for Java](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html) — Accessed: 2026-07-06
- [Curity: JWT Security Best Practices](https://curity.io/resources/learn/jwt-best-practices/) — Accessed: 2026-07-06
- [Auth0: JSON Web Tokens](https://auth0.com/docs/secure/tokens/json-web-tokens) — Accessed: 2026-07-06
- [PortSwigger: JWT Attacks](https://portswigger.net/web-security/jwt) — Accessed: 2026-07-06
- [Stytch: A Developer's Guide to RFC 7519](https://stytch.com/blog/rfc-7519-jwt-part-1/) — Accessed: 2026-07-06
