# Android

Android プラットフォーム内部の分析。Linux カーネル（[`../operating-systems/`](../operating-systems/)
が扱う一般機構）の上に築かれた、モバイル固有の制約（バッテリー・メモリ・相互不信のアプリ群）
からの再設計を「汎用 Linux との差分」として読み解く。hardware/software 両スタックの
交点（Verified Boot・Keystore・車載 Android Automotive OS）にもまたがる領域。
同じ制約への対照的な解である [`../ios/`](../ios/) と相互参照で対比する。

## 構成

- `platform-architecture.md` — プラットフォームアーキテクチャ：Zygote と COW 分岐、
  Binder IPC（ケイパビリティ・呼び出し元保証）、ART（AOT/JIT ハイブリッド・並行コピー GC）、
  lmkd と「プロセス＝キャッシュ」、HAL/Treble/APEX、Verified Boot、多層サンドボックス
- `binder-internals.md` — Binder 深掘り：ドライバのオブジェクトモデル (node/ref) と
  ハンドル翻訳、1コピー転送と mmap バッファ、トランザクションスタックと再入、oneway の
  順序・背圧、参照カウントと linkToDeath、3ドメイン (binder/hwbinder/vndbinder)、Bad Binder
- `art-runtime-internals.md` — ART 深掘り：DEX 形式と nterp、dex2oat と Optimizing
  コンパイラ、プロファイル誘導コンパイルの資材循環 (JIT → .prof → 背景 AOT → Cloud
  Profiles)、boot image と COW 共有、GC の系譜（並行コピー → userfaultfd マークコンパクト）、JNI
- `graphics-pipeline.md` — グラフィックスパイプライン：Choreographer と VSYNC 位相、
  RenderThread/HWUI、gralloc・BufferQueue・フェンス（ゼロコピーと背圧）、
  SurfaceFlinger と HWC 合成、BLAST、可変リフレッシュレートとフレームペーシング

## 今後のテーマ候補

- Android Automotive OS — 車載への展開（`../../hardware/automotive/` との合流点）
- アプリセキュリティモデル深掘り — 権限、scoped storage、Play Integrity
- メディアスタック — Codec2、MediaSession、Stagefright 以後のプロセス分割の詳細
