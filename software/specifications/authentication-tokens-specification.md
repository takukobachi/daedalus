# Authentication Token Specification: JWT and OAuth 2.0
# 認証トークン仕様書: JWT と OAuth 2.0

**Version**: 1.0
**Last Updated**: 2025-11-30
**Author**: Claude @ Daedalus

---

## Table of Contents / 目次

1. [Overview / 概要](#overview--概要)
2. [JWT (JSON Web Token)](#jwt-json-web-token)
3. [OAuth 2.0](#oauth-20)
4. [OpenID Connect (OIDC)](#openid-connect-oidc)
5. [JWT vs OAuth: Comparison / 比較](#jwt-vs-oauth-comparison--比較)
6. [Security Considerations / セキュリティ考察](#security-considerations--セキュリティ考察)
7. [Best Practices / ベストプラクティス](#best-practices--ベストプラクティス)
8. [References / 参考文献](#references--参考文献)

---

## Overview / 概要

**English**:
This document provides a comprehensive technical analysis of modern authentication and authorization mechanisms, focusing on JWT (JSON Web Token) and OAuth 2.0. These technologies form the foundation of secure identity management in web applications and APIs.

**日本語**:
本ドキュメントは、JWT（JSON Web Token）およびOAuth 2.0を中心とした、現代の認証・認可メカニズムに関する包括的な技術分析を提供します。これらの技術は、Webアプリケーションおよび API におけるセキュアなアイデンティティ管理の基盤を構成しています。

### Key Terminology / 主要用語

| English | 日本語 | Description |
|---------|--------|-------------|
| Authentication | 認証 | Verifying the identity of a user or system |
| Authorization | 認可 | Granting permission to access resources |
| Token | トークン | A piece of data representing authentication/authorization state |
| Claim | クレーム | A statement about an entity (typically the user) |
| Scope | スコープ | The extent of access granted by a token |

---

## JWT (JSON Web Token)

### Overview / 概要

**English**:
JSON Web Token (JWT) is a compact, URL-safe means of representing claims to be transferred between two parties. Defined in [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519), JWTs are self-contained tokens that carry information (claims) in a JSON format, digitally signed to ensure integrity.

**日本語**:
JSON Web Token（JWT）は、二者間で転送されるクレームを表現するための、コンパクトでURLセーフな手段です。[RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)で定義されており、JWTはJSON形式で情報（クレーム）を含む自己完結型トークンで、完全性を保証するためにデジタル署名されています。

### Structure / 構造

A JWT consists of three parts separated by periods (`.`):

```
xxxxx.yyyyy.zzzzz
[Header].[Payload].[Signature]
```

#### 1. Header / ヘッダー

**English**:
The header typically consists of two parts: the type of token (JWT) and the signing algorithm being used.

**日本語**:
ヘッダーは通常、トークンのタイプ（JWT）と使用される署名アルゴリズムの2つの部分で構成されます。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

#### 2. Payload / ペイロード

**English**:
The payload contains the claims. Claims are statements about an entity (typically, the user) and additional data.

**日本語**:
ペイロードにはクレームが含まれます。クレームは、エンティティ（通常はユーザー）に関する記述と追加データです。

##### Registered Claims / 登録済みクレーム

| Claim | Name | Description / 説明 |
|-------|------|-------------------|
| `iss` | Issuer | Token issuer / トークン発行者 |
| `sub` | Subject | Subject of the token / トークンの対象者 |
| `aud` | Audience | Intended recipient / 想定される受信者 |
| `exp` | Expiration Time | Expiration timestamp / 有効期限（タイムスタンプ） |
| `nbf` | Not Before | Token not valid before this time / この時刻より前は無効 |
| `iat` | Issued At | Token issue timestamp / トークン発行時刻 |
| `jti` | JWT ID | Unique identifier / 一意識別子 |

##### Example Payload / ペイロード例

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622,
  "role": "admin"
}
```

#### 3. Signature / 署名

**English**:
The signature is created by taking the encoded header, encoded payload, a secret key, and the algorithm specified in the header.

**日本語**:
署名は、エンコードされたヘッダー、エンコードされたペイロード、秘密鍵、およびヘッダーで指定されたアルゴリズムを使用して作成されます。

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

### Supported Algorithms / サポートされるアルゴリズム

| Algorithm | Type | Description / 説明 |
|-----------|------|-------------------|
| **HS256** | Symmetric | HMAC with SHA-256 (shared secret) |
| **HS384** | Symmetric | HMAC with SHA-384 |
| **HS512** | Symmetric | HMAC with SHA-512 |
| **RS256** | Asymmetric | RSA signature with SHA-256 |
| **RS384** | Asymmetric | RSA signature with SHA-384 |
| **RS512** | Asymmetric | RSA signature with SHA-512 |
| **ES256** | Asymmetric | ECDSA with P-256 and SHA-256 |
| **ES384** | Asymmetric | ECDSA with P-384 and SHA-384 |
| **ES512** | Asymmetric | ECDSA with P-521 and SHA-512 |
| **none** | None | Unsecured JWT (DANGEROUS) |

### JWT Flow Diagram / JWT フロー図

```mermaid
sequenceDiagram
    participant Client as Client / クライアント
    participant Auth as Auth Server / 認証サーバー
    participant API as Resource Server / リソースサーバー

    Client->>Auth: 1. Login (credentials)
    Auth->>Auth: 2. Validate credentials
    Auth->>Client: 3. Return JWT
    Client->>API: 4. Request + JWT (Authorization: Bearer)
    API->>API: 5. Verify JWT signature
    API->>Client: 6. Return protected resource
```

### Characteristics / 特徴

| Characteristic | Description (EN) | 説明 (JP) |
|----------------|------------------|-----------|
| **Stateless** | No server-side session storage required | サーバー側のセッションストレージ不要 |
| **Self-contained** | Contains all necessary information | 必要な情報をすべて含む |
| **Compact** | Small size, suitable for URLs | 小さいサイズ、URLに適している |
| **Verifiable** | Signature ensures integrity | 署名が完全性を保証 |

---

## OAuth 2.0

### Overview / 概要

**English**:
OAuth 2.0 is an authorization framework defined in [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) that enables third-party applications to obtain limited access to HTTP services. It focuses on client developer simplicity while providing specific authorization flows for various application types.

**日本語**:
OAuth 2.0は、[RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)で定義された認可フレームワークで、サードパーティアプリケーションがHTTPサービスへの限定的なアクセスを取得することを可能にします。クライアント開発者の簡便性に重点を置きながら、様々なアプリケーションタイプに対応した認可フローを提供します。

### Roles / 役割

```
┌─────────────────────────────────────────────────────────────────┐
│                     OAuth 2.0 Roles                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │  Resource Owner  │         │      Client      │             │
│  │  リソースオーナー   │         │   クライアント    │             │
│  │  (End User)      │         │  (Application)   │             │
│  └────────┬─────────┘         └────────┬─────────┘             │
│           │                            │                        │
│           │ Authorizes                 │ Requests               │
│           ▼                            ▼                        │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │ Authorization    │◄────────│  Resource        │             │
│  │ Server           │         │  Server          │             │
│  │ 認可サーバー       │         │  リソースサーバー  │             │
│  └──────────────────┘         └──────────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Role | English | 日本語 |
|------|---------|--------|
| **Resource Owner** | Entity capable of granting access | アクセスを許可できるエンティティ（通常はエンドユーザー） |
| **Client** | Application requesting access | アクセスを要求するアプリケーション |
| **Authorization Server** | Issues tokens after authentication | 認証後にトークンを発行するサーバー |
| **Resource Server** | Hosts protected resources | 保護されたリソースをホストするサーバー |

### Grant Types / 認可タイプ

#### 1. Authorization Code Grant / 認可コードグラント

**English**:
The most secure flow, recommended for server-side applications. Uses a temporary authorization code exchanged for tokens.

**日本語**:
最も安全なフローで、サーバーサイドアプリケーションに推奨されます。一時的な認可コードをトークンと交換します。

```mermaid
sequenceDiagram
    participant User as User / ユーザー
    participant Client as Client App
    participant Auth as Authorization Server
    participant Resource as Resource Server

    User->>Client: 1. Access application
    Client->>Auth: 2. Redirect to authorization endpoint
    Auth->>User: 3. Login & consent prompt
    User->>Auth: 4. Grant permission
    Auth->>Client: 5. Redirect with authorization code
    Client->>Auth: 6. Exchange code for tokens
    Auth->>Client: 7. Return access token (+ refresh token)
    Client->>Resource: 8. Request with access token
    Resource->>Client: 9. Return protected resource
```

#### 2. Authorization Code with PKCE / 認可コード + PKCE

**English**:
An enhanced version of Authorization Code flow with Proof Key for Code Exchange (PKCE). **Now recommended for ALL clients** including confidential clients.

**日本語**:
Proof Key for Code Exchange（PKCE）を使用した認可コードフローの拡張版。**現在はすべてのクライアント**（機密クライアントを含む）に推奨されています。

```
┌─────────────────────────────────────────────────────────────────┐
│                    PKCE Flow                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Generate code_verifier (random string)                      │
│     code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r..."         │
│                                                                  │
│  2. Generate code_challenge                                      │
│     code_challenge = BASE64URL(SHA256(code_verifier))           │
│                                                                  │
│  3. Send code_challenge with authorization request              │
│     /authorize?...&code_challenge=E9Melhoa...&code_challenge_method=S256
│                                                                  │
│  4. Exchange code with code_verifier                            │
│     /token?...&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r...  │
│                                                                  │
│  5. Server verifies: SHA256(code_verifier) == code_challenge    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 3. Client Credentials Grant / クライアントクレデンシャルグラント

**English**:
Used for machine-to-machine (M2M) communication where no user is involved.

**日本語**:
ユーザーが関与しないマシン間（M2M）通信に使用されます。

```
POST /token HTTP/1.1
Host: authorization-server.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=xxxxxxxxxx
&client_secret=yyyyyyyyyy
&scope=read:data
```

#### 4. Implicit Grant (DEPRECATED) / インプリシットグラント（非推奨）

**English**:
⚠️ **DEPRECATED in OAuth 2.1**. Was designed for browser-based apps but exposes tokens in URLs. Use Authorization Code with PKCE instead.

**日本語**:
⚠️ **OAuth 2.1で非推奨**。ブラウザベースのアプリ向けに設計されましたが、URLでトークンが露出します。代わりに認可コード + PKCEを使用してください。

#### 5. Resource Owner Password Credentials (DEPRECATED) / リソースオーナーパスワードクレデンシャル（非推奨）

**English**:
⚠️ **DEPRECATED in OAuth 2.1**. Requires sharing user credentials with the client. Only for legacy systems with high trust.

**日本語**:
⚠️ **OAuth 2.1で非推奨**。ユーザー認証情報をクライアントと共有する必要があります。高い信頼性を持つレガシーシステムでのみ使用。

### Token Types / トークンタイプ

| Token Type | Purpose (EN) | 目的 (JP) | Lifetime |
|------------|--------------|-----------|----------|
| **Access Token** | Access protected resources | 保護されたリソースへのアクセス | Short (minutes to hours) |
| **Refresh Token** | Obtain new access tokens | 新しいアクセストークンの取得 | Long (days to months) |

---

## OpenID Connect (OIDC)

### Overview / 概要

**English**:
OpenID Connect (OIDC) is an identity layer built on top of OAuth 2.0. It adds authentication capabilities to OAuth's authorization framework by introducing the ID Token.

**日本語**:
OpenID Connect（OIDC）は、OAuth 2.0の上に構築されたアイデンティティレイヤーです。IDトークンを導入することで、OAuthの認可フレームワークに認証機能を追加します。

### Relationship with OAuth 2.0 / OAuth 2.0との関係

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                    ┌─────────────────────┐                      │
│                    │   OpenID Connect    │                      │
│                    │   (Authentication)  │                      │
│                    │   認証               │                      │
│                    └──────────┬──────────┘                      │
│                               │                                  │
│                    ┌──────────▼──────────┐                      │
│                    │     OAuth 2.0       │                      │
│                    │   (Authorization)   │                      │
│                    │   認可               │                      │
│                    └──────────┬──────────┘                      │
│                               │                                  │
│                    ┌──────────▼──────────┐                      │
│                    │       HTTP          │                      │
│                    └─────────────────────┘                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ID Token / IDトークン

**English**:
The ID Token is a JWT that contains claims about the authentication event and the authenticated user.

**日本語**:
IDトークンは、認証イベントと認証されたユーザーに関するクレームを含むJWTです。

#### ID Token Claims / IDトークンのクレーム

| Claim | Required | Description (EN) | 説明 (JP) |
|-------|----------|------------------|-----------|
| `iss` | Yes | Issuer identifier | 発行者識別子 |
| `sub` | Yes | Subject identifier | サブジェクト識別子 |
| `aud` | Yes | Audience | オーディエンス |
| `exp` | Yes | Expiration time | 有効期限 |
| `iat` | Yes | Issued at time | 発行時刻 |
| `auth_time` | Conditional | Authentication time | 認証時刻 |
| `nonce` | Conditional | String to associate session | セッション関連付け文字列 |
| `acr` | Optional | Authentication context class | 認証コンテキストクラス |
| `amr` | Optional | Authentication methods | 認証方法 |
| `azp` | Optional | Authorized party | 認可された当事者 |

### OIDC Scopes / OIDCスコープ

| Scope | Claims Returned | 返されるクレーム |
|-------|-----------------|-----------------|
| `openid` | `sub` | サブジェクト識別子 |
| `profile` | `name`, `family_name`, `given_name`, `nickname`, `picture`, etc. | プロフィール情報 |
| `email` | `email`, `email_verified` | メールアドレス |
| `address` | `address` | 住所 |
| `phone` | `phone_number`, `phone_number_verified` | 電話番号 |

---

## JWT vs OAuth: Comparison / 比較

### Fundamental Difference / 根本的な違い

| Aspect | JWT | OAuth 2.0 |
|--------|-----|-----------|
| **Type / タイプ** | Token Format / トークン形式 | Authorization Protocol / 認可プロトコル |
| **Purpose / 目的** | Encode and transmit claims / クレームのエンコードと送信 | Delegate access / アクセスの委任 |
| **Statefulness** | Stateless / ステートレス | Stateful (requires auth server) / ステートフル |
| **Revocation / 失効** | Difficult (no built-in mechanism) / 困難 | Built-in revocation endpoint / 組み込みの失効エンドポイント |

### When to Use / 使い分け

```
┌─────────────────────────────────────────────────────────────────┐
│                    Decision Matrix / 選択マトリクス              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Use JWT when / JWTを使用する場合:                               │
│  ├─ Stateless authentication needed / ステートレス認証が必要     │
│  ├─ Microservices architecture / マイクロサービスアーキテクチャ   │
│  ├─ API authentication / API認証                                │
│  └─ Server-to-server communication / サーバー間通信              │
│                                                                  │
│  Use OAuth when / OAuthを使用する場合:                           │
│  ├─ Third-party access delegation / サードパーティアクセス委任    │
│  ├─ Social login (Google, Facebook, etc.) / ソーシャルログイン   │
│  ├─ User consent for data access / データアクセスのユーザー同意   │
│  └─ API access for external apps / 外部アプリのAPIアクセス       │
│                                                                  │
│  Use Both Together / 両方を組み合わせる場合:                      │
│  ├─ OAuth issues JWT as access token                            │
│  ├─ OIDC ID tokens are JWTs                                     │
│  └─ Complex enterprise authentication                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Security Considerations / セキュリティ考察

### JWT Vulnerabilities / JWT脆弱性

#### 1. Algorithm Confusion Attack / アルゴリズム混同攻撃

**English**:
Attackers exploit libraries that accept the `alg` header without validation, switching from RS256 to HS256 and using the public key as the HMAC secret.

**日本語**:
攻撃者は、`alg`ヘッダーを検証せずに受け入れるライブラリを悪用し、RS256からHS256に切り替え、公開鍵をHMACシークレットとして使用します。

```
# Attack scenario / 攻撃シナリオ
Original (RS256): Signed with private key, verified with public key
Attack (HS256):   Attacker signs with public key (now used as HMAC secret)
```

**Mitigation / 対策**:
- Always whitelist allowed algorithms / 許可されたアルゴリズムを常にホワイトリスト化
- Never trust the `alg` header from the token / トークンの`alg`ヘッダーを信頼しない

#### 2. None Algorithm Attack / noneアルゴリズム攻撃

**English**:
Some libraries accept `alg: none` which means no signature verification.

**日本語**:
一部のライブラリは`alg: none`を受け入れ、署名検証が行われません。

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

**Mitigation / 対策**:
- Reject tokens with `alg: none` in production / 本番環境では`alg: none`のトークンを拒否
- Explicitly configure accepted algorithms / 受け入れるアルゴリズムを明示的に設定

#### 3. Key ID (kid) Injection / キーID (kid) インジェクション

**English**:
The `kid` parameter can be exploited for SQL injection or path traversal attacks.

**日本語**:
`kid`パラメータは、SQLインジェクションやパストラバーサル攻撃に悪用される可能性があります。

**Mitigation / 対策**:
- Sanitize and validate `kid` values / `kid`値をサニタイズして検証
- Use UUIDs instead of file paths / ファイルパスの代わりにUUIDを使用

#### 4. Token Theft / トークン窃取

**English**:
JWTs stored insecurely can be stolen via XSS attacks.

**日本語**:
安全でない方法で保存されたJWTは、XSS攻撃によって窃取される可能性があります。

**Mitigation / 対策**:
- Store tokens in HttpOnly cookies / HttpOnly Cookieにトークンを保存
- Use short expiration times / 短い有効期限を使用
- Implement token refresh mechanisms / トークン更新メカニズムを実装

### OAuth 2.0 Security Threats / OAuth 2.0のセキュリティ脅威

| Threat / 脅威 | Description (EN) | 説明 (JP) | Mitigation / 対策 |
|---------------|------------------|-----------|-------------------|
| **CSRF** | Cross-site request forgery | クロスサイトリクエストフォージェリ | Use `state` parameter |
| **Code Interception** | Authorization code stolen | 認可コードの窃取 | Use PKCE |
| **Open Redirector** | Malicious redirect URI | 悪意のあるリダイレクトURI | Strict URI validation |
| **Token Leakage** | Tokens exposed in logs/URLs | ログ/URLでのトークン露出 | Use POST for token exchange |

---

## Best Practices / ベストプラクティス

### JWT Best Practices / JWTのベストプラクティス

```
┌─────────────────────────────────────────────────────────────────┐
│                 JWT Security Checklist                           │
│                 JWTセキュリティチェックリスト                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Algorithm Security / アルゴリズムセキュリティ                    │
│  □ Whitelist allowed algorithms / アルゴリズムをホワイトリスト化  │
│  □ Use RS256 or ES256 for public clients / 公開クライアントには  │
│    RS256またはES256を使用                                        │
│  □ Reject "none" algorithm / "none"アルゴリズムを拒否            │
│                                                                  │
│  Token Lifetime / トークンの有効期間                              │
│  □ Short expiration (15min-1hr) / 短い有効期限（15分〜1時間）     │
│  □ Use refresh tokens for long sessions / 長いセッションには      │
│    リフレッシュトークンを使用                                     │
│  □ Implement token rotation / トークンローテーションを実装        │
│                                                                  │
│  Claim Validation / クレーム検証                                  │
│  □ Verify iss (issuer) / iss（発行者）を検証                     │
│  □ Verify aud (audience) / aud（オーディエンス）を検証            │
│  □ Check exp (expiration) / exp（有効期限）を確認                 │
│  □ Validate nbf (not before) / nbf（有効開始時刻）を検証          │
│                                                                  │
│  Storage / ストレージ                                             │
│  □ Use HttpOnly cookies / HttpOnly Cookieを使用                  │
│  □ Set Secure flag (HTTPS only) / Secureフラグを設定              │
│  □ Set SameSite attribute / SameSite属性を設定                   │
│  □ Never store in localStorage for sensitive apps /              │
│    機密性の高いアプリではlocalStorageに保存しない                  │
│                                                                  │
│  Key Management / 鍵管理                                          │
│  □ Use strong keys (min 256 bits) / 強力な鍵を使用（最小256ビット）│
│  □ Rotate keys regularly / 定期的に鍵をローテーション              │
│  □ Use secrets manager / シークレットマネージャーを使用           │
│  □ Separate keys per environment / 環境ごとに鍵を分離             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### OAuth 2.0 Best Practices / OAuth 2.0のベストプラクティス

```
┌─────────────────────────────────────────────────────────────────┐
│                OAuth 2.0 Security Checklist                      │
│                OAuth 2.0セキュリティチェックリスト                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Flow Selection / フロー選択                                      │
│  □ Use Authorization Code + PKCE for all clients /               │
│    すべてのクライアントで認可コード + PKCEを使用                   │
│  □ Avoid Implicit flow / インプリシットフローを避ける             │
│  □ Avoid Password grant / パスワードグラントを避ける              │
│                                                                  │
│  PKCE Configuration / PKCE設定                                    │
│  □ Use S256 challenge method / S256チャレンジメソッドを使用       │
│  □ Generate cryptographically random code_verifier /             │
│    暗号的にランダムなcode_verifierを生成                          │
│                                                                  │
│  Redirect URI / リダイレクトURI                                   │
│  □ Exact match validation / 完全一致検証                         │
│  □ No wildcards in production / 本番環境ではワイルドカードなし    │
│  □ HTTPS required / HTTPS必須                                    │
│                                                                  │
│  Token Security / トークンセキュリティ                            │
│  □ Use short-lived access tokens / 短命のアクセストークンを使用   │
│  □ Implement refresh token rotation / リフレッシュトークン        │
│    ローテーションを実装                                           │
│  □ Bind tokens to client (DPoP/mTLS) / トークンをクライアントに   │
│    バインド                                                       │
│                                                                  │
│  State Management / 状態管理                                      │
│  □ Use state parameter for CSRF protection /                     │
│    CSRF保護にstateパラメータを使用                                │
│  □ Use nonce for replay protection /                             │
│    リプレイ保護にnonceを使用                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Recommendations / 実装推奨事項

#### Recommended Libraries / 推奨ライブラリ

| Language | JWT Library | OAuth/OIDC Library |
|----------|-------------|-------------------|
| **Node.js** | `jsonwebtoken`, `jose` | `passport`, `openid-client` |
| **Python** | `PyJWT`, `python-jose` | `Authlib`, `oauthlib` |
| **Java** | `jjwt`, `nimbus-jose-jwt` | `Spring Security OAuth` |
| **Go** | `golang-jwt/jwt` | `golang.org/x/oauth2` |
| **Rust** | `jsonwebtoken` | `oauth2` |

---

## References / 参考文献

### Standards and RFCs / 標準とRFC

| Document | Title | URL |
|----------|-------|-----|
| RFC 7519 | JSON Web Token (JWT) | [datatracker.ietf.org/doc/html/rfc7519](https://datatracker.ietf.org/doc/html/rfc7519) |
| RFC 6749 | The OAuth 2.0 Authorization Framework | [datatracker.ietf.org/doc/html/rfc6749](https://datatracker.ietf.org/doc/html/rfc6749) |
| RFC 7636 | PKCE for OAuth Public Clients | [datatracker.ietf.org/doc/html/rfc7636](https://datatracker.ietf.org/doc/html/rfc7636) |
| RFC 8725 | JWT Best Current Practices | [datatracker.ietf.org/doc/html/rfc8725](https://datatracker.ietf.org/doc/html/rfc8725) |
| RFC 7518 | JSON Web Algorithms (JWA) | [datatracker.ietf.org/doc/html/rfc7518](https://datatracker.ietf.org/doc/html/rfc7518) |
| OIDC Core | OpenID Connect Core 1.0 | [openid.net/specs/openid-connect-core-1_0.html](https://openid.net/specs/openid-connect-core-1_0.html) |

### Security Resources / セキュリティリソース

- [OWASP OAuth 2.0 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)
- [OWASP JWT Cheat Sheet for Java](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [Curity JWT Best Practices](https://curity.io/resources/learn/jwt-best-practices/)
- [Auth0 JWT Documentation](https://auth0.com/docs/secure/tokens/json-web-tokens)
- [PortSwigger JWT Attacks](https://portswigger.net/web-security/jwt)

### Additional Reading / 追加資料

- [OAuth 2.0 Security Best Current Practice (Draft)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [OAuth 2.1 Draft Specification](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)
- [Stytch: A Developer's Guide to RFC 7519](https://stytch.com/blog/rfc-7519-jwt-part-1/)

---

## Appendix A: Token Examples / 付録A: トークン例

### JWT Example / JWT例

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.POstGetfAytaZS82wHcjoTyoqhMyxXiWdR7Nn7A29DNSl0EiXLdwJ6xC6AfgZWF1bOsS_TuYI3OG85AmiExREkrS6tDfTQ2B3WXlrr-wp5AokiRbz3_oB4OxG-W9KcEEbDRcZc0nH3L7LzYptiy1PtAylQGxHTWZXtGz4ht0bAecBgmpdgXMguEIcoqPJ1n3pIWk_dUZegpqx0Lka21H6XxUTxiy8OcaarA8zdnPUnV6AmNP3ecFawIFYdvJB_cm-GvpCSbr8G8y_Mllj8f4x9nBH8pQux89_6gUY618iYv7tuPWBFfEbLxtF2pZS6YC1aSfLQxeNe8djT9YjpvRZA
```

**Decoded Header / デコードされたヘッダー**:
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Decoded Payload / デコードされたペイロード**:
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "iat": 1516239022
}
```

### OAuth 2.0 Token Response Example / OAuth 2.0トークンレスポンス例

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

## Appendix B: Security Incident Response / 付録B: セキュリティインシデント対応

### Token Compromise Response / トークン漏洩時の対応

**English**:
1. Immediately revoke the compromised token (if using OAuth)
2. Rotate signing keys if private key compromised
3. Force re-authentication for affected users
4. Review access logs for unauthorized access
5. Implement additional monitoring

**日本語**:
1. 漏洩したトークンを直ちに失効させる（OAuthの場合）
2. 秘密鍵が漏洩した場合は署名鍵をローテーション
3. 影響を受けたユーザーに再認証を強制
4. アクセスログで不正アクセスを確認
5. 追加の監視を実装

---

**Document End / ドキュメント終了**

*Navigate the labyrinth of authentication with wisdom and precision.*
*知恵と精度を持って、認証の迷宮を進め。*
