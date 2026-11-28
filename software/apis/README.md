# API 仕様 (API Specifications)

実システム・プロトコルの仕様分析 (specification analysis)。一般原理を扱う他領域と
異なり、特定のプロダクト・標準仕様（決済、認証 …）を対象に、公式ドキュメントや
RFC を一次資料として API・データモデル・プロトコルフローを再構成する層。

## 構成

- `stripe-connect-specification.md` — Stripe Connect：マルチパーティ決済の仕様
  （コントローラープロパティと Accounts v2 による現行アカウントモデルとレガシーの
  アカウントタイプ, charge types と資金フロー, 埋め込みコンポーネント, Webhook,
  日本向け料金（2026-10 時点））
- `card-payment-networks.md` — カード決済ネットワーク：四者モデル、ISO 8583
  （MTI・ビットマップ・データ要素）と ISO 20022 (cain) への移行の位置づけ、
  承認・清算・精算の分離、インターチェンジ（EU の上限規制）、EMV の ARQC / ARPC、
  EMVCo ネットワークトークン、EMV 3DS 2.x、PSD2 SCA と免除、Visa の紛争処理、
  PCI DSS v4.0.1 の適用範囲（2026-11 時点）。`stripe-connect-specification.md` が
  抽象化している層で、端末側は
  [`../../hardware/peripherals/nfc-contactless-payments.md`](../../hardware/peripherals/nfc-contactless-payments.md)
- `authentication-tokens-specification.md` — 認証トークン (JWT / OAuth 2.0 / OIDC)：
  トークン形式 (RFC 9068)、認可コード + PKCE、PAR / RAR / トークン交換、
  送信者制約 (DPoP / mTLS)、RFC 9700 準拠のセキュリティ考察と OAuth 2.1 ドラフトの状況。
  [`../networking/tls-internals.md`](../networking/tls-internals.md) が扱うトランスポート層
  セキュリティの上に立つ、アプリケーション層の認証・認可
- `webauthn-passkeys-specification.md` — WebAuthn / パスキー (passkeys)：
  W3C WebAuthn Level 3（2026-08 勧告）と FIDO CTAP 2.2 の仕様分析。
  登録・認証の儀式、RP ID とオリジン束縛によるフィッシング耐性、UP/UV、
  発見可能な資格情報と条件付き UI、同期型パスキーと BE/BS フラグ、
  hybrid トランスポート、アテステーションとプライバシー、アカウント回復、
  脅威モデル。`authentication-tokens-specification.md` の手前に立つ
  「最初の本人確認」の層
