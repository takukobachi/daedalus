# Browsers

ブラウザ内部 (browser internals) の分析。[`../android/`](../android/README.md)・[`../ios/`](../ios/README.md) に続く**第三のプラットフォーム分析**であり、最も敵対的な前提——署名もインストールもない任意コードの即時実行——の下で、プロセス隔離・IPC・実行系・グラフィックスがどう再設計されるかを読む。ハードウェアの投機実行 (Spectre) がプロセスモデルを書き換えたという意味で、hardware/software 両スタックを縦断する題材でもある。主対象は Chromium、対照として WebKit/Firefox。

## 構成

- `browser-architecture.md` — ブラウザアーキテクチャ：マルチプロセス構成と
  「compromised renderer」信頼モデル、サイト分離と OOPIF (Spectre への構造的応答)、
  サンドボックス (namespaces/seccomp-BPF ほか OS 別の檻) と Mojo ケイパビリティ IPC、
  レンダリングパイプライン (LayoutNG・コンポジタ・Viz) と VSYNC 締切、V8 の 4 ティア
  JIT (Ignition→Sparkplug→Maglev→TurboFan)・Orinoco GC・V8 sandbox、攻撃連鎖と
  メモリ安全対策の積層

## 今後のテーマ候補

- V8 深掘り (V8 internals) — hidden class 遷移の実装、Maglev/TurboFan の IR、
  ポインタ圧縮、Torque。`../compilers-runtimes/` の一般論の実装各論
- ウェブセキュリティモデル (web security model) — 同一生成元ポリシーの意味論、
  XSS/CSRF、COOP/COEP/CORP、Permissions Policy。`../security/` のウェブ応用層
- ネットワーキングと読み込み (loading & networking) — リソーススケジューリング、
  プリロード/プリコネクト、HTTP キャッシュ、Service Worker。`../networking/` との接点
- 拡張機能アーキテクチャ (extensions) — Manifest V3、権限モデル、コンテンツ
  スクリプトの隔離ワールド。「ブラウザ内のアプリ」のセキュリティ設計
