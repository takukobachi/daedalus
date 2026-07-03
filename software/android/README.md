# Android

Android プラットフォーム内部の分析。Linux カーネル（[`../operating-systems/`](../operating-systems/)
が扱う一般機構）の上に築かれた、モバイル固有の制約（バッテリー・メモリ・相互不信のアプリ群）
からの再設計を「汎用 Linux との差分」として読み解く。hardware/software 両スタックの
交点（Verified Boot・Keystore・車載 Android Automotive OS）にもまたがる領域。

## 構成

- `platform-architecture.md` — プラットフォームアーキテクチャ：Zygote と COW 分岐、
  Binder IPC（ケイパビリティ・呼び出し元保証）、ART（AOT/JIT ハイブリッド・並行コピー GC）、
  lmkd と「プロセス＝キャッシュ」、HAL/Treble/APEX、Verified Boot、多層サンドボックス
- `binder-internals.md` — Binder 深掘り：ドライバのオブジェクトモデル (node/ref) と
  ハンドル翻訳、1コピー転送と mmap バッファ、トランザクションスタックと再入、oneway の
  順序・背圧、参照カウントと linkToDeath、3ドメイン (binder/hwbinder/vndbinder)、Bad Binder

## 今後のテーマ候補

- ART ランタイム深掘り (ART internals) — DEX 形式、コンパイラ IR、GC の実装詳細
- グラフィックスパイプライン (graphics) — SurfaceFlinger, BufferQueue, Vulkan, フレームペーシング
- Android Automotive OS — 車載への展開（`../../hardware/automotive/` との合流点）
- アプリセキュリティモデル深掘り — 権限、scoped storage、Play Integrity
