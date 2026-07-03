# iOS

iOS プラットフォーム内部の分析。XNU カーネル (Mach + BSD) の上に「署名と強制ポリシで
全実行コードを統制する」ユーザランドを載せた設計を、[`../android/`](../android/) と同じ
切り口（プロセス・IPC・実行系・メモリ・セキュリティ）で読み解く。同じモバイル制約
（バッテリー・メモリ・相互不信のアプリ群）への **Android とは対照的な別解**として、
両者の対比から IPC・メモリ管理・セキュリティモデルの設計空間を照らすことを狙う。

## 構成

- `platform-architecture.md` — プラットフォームアーキテクチャ：XNU (Mach/BSD 二重構造)、
  launchd とオンデマンド起動、dyld 共有キャッシュ、ObjC/Swift ランタイムと ARC（対 GC）、
  圧縮＋Jetsam（対 lmkd）、コード署名・サンドボックス・Secure Enclave・PAC/SPTM
- `mach-ipc-xpc.md` — Mach IPC / XPC 深掘り：ポート権利のケイパビリティモデルと
  名前翻訳、send-once と通知 (dead-name / no-senders)、OOL 転送の COW 再マップ、
  launchd bootstrap、XPC 層、audit token と voucher、Binder との設計比較
- `security-architecture.md` — セキュリティアーキテクチャ深掘り：ブートチェーンと
  パーソナライズ署名、Secure Enclave 内部 (sepOS・UID 鍵・Memory Protection Engine)、
  Data Protection 4 クラスと Keychain、コード署名の強制 (AMFI・トラストキャッシュ・
  CoreTrust・エンタイトルメント)、サンドボックスと TCC、緩和の積層 (PAC → PPL →
  SPTM/TXM → MIE)、Android との対応表

## 今後のテーマ候補

- Core Animation / レンダーサーバ — 描画パイプライン（`../android/graphics-pipeline.md` の対比対象）
- Swift 並行性 (Swift Concurrency) — actor・structured concurrency とランタイム実装
- WebKit / JavaScriptCore — 唯一の JIT 特権とそのサンドボックス構造
