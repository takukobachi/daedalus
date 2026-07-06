# Specifications

実システム・プロトコルの仕様分析 (specification analysis)。一般原理を扱う他領域と
異なり、特定のプロダクト・標準仕様（決済、認証 …）を対象に、公式ドキュメントや
RFC を一次資料として API・データモデル・プロトコルフローを再構成する層。

## 構成

- `stripe-connect-specification.md` — Stripe Connect：マルチパーティ決済の仕様
  （アカウントタイプ, charge types, 資金フロー, Webhook, オンボーディング）
- `authentication-tokens-specification.md` — 認証トークン (JWT / OAuth 2.0 / OIDC)：
  トークン形式、署名検証、認可フロー、セキュリティ考察。
  [`../networking/tls-internals.md`](../networking/tls-internals.md) が扱うトランスポート層
  セキュリティの上に立つ、アプリケーション層の認証・認可

> **Note**: `authentication-tokens-specification.md` はリポジトリ初期の英日全文併記
> スタイルで書かれており、現行の言語ポリシー（日本語主体・技術用語は初出時に英語併記）
> への改稿候補。`stripe-connect-specification.md` は v2.0 で改稿済み。

## 今後のテーマ候補

- WebAuthn / パスキー (passkeys) — FIDO2/CTAP、公開鍵ベース認証への移行。
  `authentication-tokens-specification.md` の共有シークレット型認証の後継となる様式
- 決済ネットワークの下部構造 (payment networks) — カードネットワーク (ISO 8583)、
  3-D セキュア、清算と精算。`stripe-connect-specification.md` が抽象化している層
