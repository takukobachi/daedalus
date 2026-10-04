# Browsers

ブラウザ内部 (browser internals) の分析。[`../android/`](../android/README.md)・[`../ios/`](../ios/README.md) に続く**第三のプラットフォーム分析**であり、署名もインストールもない任意コードを即時に実行するという最も敵対的な前提の下で、プロセス隔離・IPC・実行系・グラフィックスがどう再設計されるかを読む。ハードウェアの投機実行 (Spectre) がプロセスモデルを書き換えたという意味で、hardware/software 両スタックを縦断する題材でもある。主対象は Chromium、対照として WebKit/Firefox。

## 構成

- `browser-architecture.md` — ブラウザアーキテクチャ：マルチプロセス構成と
  「compromised renderer」信頼モデル、サイト分離と OOPIF (Spectre への構造的応答)、
  サンドボックス (namespaces/seccomp-BPF ほか OS 別の檻) と Mojo ケイパビリティ IPC、
  レンダリングパイプライン (LayoutNG・コンポジタ・Viz) と VSYNC 締切、V8 の 4 ティア
  JIT (Ignition→Sparkplug→Maglev→TurboFan)・Orinoco GC・V8 sandbox、攻撃連鎖と
  メモリ安全対策の積層
- `v8-internals.md` — V8 の内部：「動的言語の実行時の規則性への投機」という一つの
  体系として、hidden class (map) と遷移木、インラインキャッシュとフィードバックベクタ、
  4 ティアの資金配分、deopt（設計された撤退）、Orinoco GC の並行化、ポインタ圧縮、
  Torque と V8 sandbox。`../compilers-runtimes/` の一般論の実装各論
- `web-security-model.md` — ウェブセキュリティモデル：origin という主体、
  「読みは禁止・送りは自由」という既定と CSRF/XS-Leaks、XSS と CSP/Trusted Types、
  SameSite 既定化の教訓、CORS = 緩和のプロトコル、Spectre が書かせたヘッダ群
  (COOP/COEP/CORP)、Permissions Policy とケーパビリティへの収斂。
  `../security/` 横串のウェブ応用層

## 今後のテーマ候補

- ネットワーキングと読み込み (loading & networking) — リソーススケジューリング、
  プリロード/プリコネクト、HTTP キャッシュ、Service Worker。`../networking/` との接点
- 拡張機能アーキテクチャ (extensions) — Manifest V3、権限モデル、コンテンツ
  スクリプトの隔離ワールド。「ブラウザ内のアプリ」のセキュリティ設計
