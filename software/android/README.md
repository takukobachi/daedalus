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
- `app-components-lifecycle.md` — コンポーネントモデルとライフサイクル：Activity を
  「Binder で遠隔操縦される状態機械・システムが起動できる入り口・タスクの単位・
  プロセス死の契約書」として読む。ClientTransaction、二重台帳 (ATMS/ActivityThread)、
  状態の三層と TransactionTooLargeException、ANR、タスク偽装と BAL。
  `binder-internals.md` の機構が体感的現象に変換される応用編
- `art-runtime-internals.md` — ART 深掘り：DEX 形式と nterp、dex2oat と Optimizing
  コンパイラ、プロファイル誘導コンパイルの資材循環 (JIT → .prof → 背景 AOT → Cloud
  Profiles)、boot image と COW 共有、GC の系譜（並行コピー → userfaultfd マークコンパクト）、JNI
- `graphics-pipeline.md` — グラフィックスパイプライン：Choreographer と VSYNC 位相、
  RenderThread/HWUI、gralloc・BufferQueue・フェンス（ゼロコピーと背圧）、
  SurfaceFlinger と HWC 合成、BLAST、可変リフレッシュレートとフレームペーシング
- `security-model.md` — セキュリティモデル深掘り：権限検査の三層 (UID/SELinux ×
  フレームワーク × ハードウェア)、権限モデルの意味論と confused deputy、アプリ署名
  v1〜v4 と鍵ローテーション、scoped storage と FBE (CE/DE)、Keystore・鍵アテステーション・
  Play Integrity、エクスプロイト緩和 (CFI/MTE/Rust)
- `looper-handler-message-loop.md` — UI スレッドとメッセージループ：単一スレッド UI の理由、
  Looper/TLS、MessageQueue と epoll/eventfd、Handler、同期バリアと非同期メッセージ（フレーム優先の仕組み）、
  IdleHandler、Binder スレッド → `H` → UI スレッドの受け渡し、ロックフリー MessageQueue (DeliQueue)
- `input-pipeline.md` — 入力パイプライン：evdev と multitouch slot、EventHub/InputReader、
  InputReader〜InputDispatcher 間の処理段、SurfaceFlinger 由来のウィンドウ情報とヒットテスト、
  InputChannel (socketpair)、VSYNC 同期のバッチングと resampling、タップジャッキング対策
- `view-system-layout.md` — View システムとレイアウト：バイナリリソースと AssetManager、
  LayoutInflater・テーマ解決、PhoneWindow/DecorView、MeasureSpec と多重 measure のコスト、
  ViewRootImpl.performTraversals と WMS との契約、状態保存、Compose・edge-to-edge
- `package-management.md` — パッケージ管理とインストール：アプリ＝Linux ユーザーの UID 割り当て、
  PMS の分解（Computer スナップショット・InstallPackageHelper）、installd と artd、packages.xml (ABX)、
  ブートスキャン、インテント解決とパッケージ可視性
- `memory-lmkd.md` — メモリ管理と Low Memory Killer：カーネルのページ回収と OOM Killer、
  lowmemorykiller ドライバから PSI ベースの lmkd へ、oom_score_adj の算出 (OomAdjuster)、
  cached app freezer、zram と MGLRU、16KB ページ、アプリ別メモリ上限
- `boot-init-properties.md` — ブートと init：ブートローダーと fastboot/fastbootd、A/B・Virtual A/B・
  dynamic partitions・GKI ブートイメージ、2 段階 init と SELinux 読み込み、プロパティサービス、
  Android Init Language、Zygote と system_server の起動
- `app-launch-end-to-end.md` — アプリ起動の全行程：ランチャーのタップから最初のフレームまでを
  プロセス/スレッド/IPC 単位で追う。ATMS → zygote ソケット (USAP) → specialize → ActivityThread →
  bindApplication → ClientTransaction → 最初のフレーム、cold/warm/hot 起動と計測
- `process-thread-model.md` — プロセスとスレッドのモデル：アプリ＝UID、Zygote fork によるプロセス生成と
  `android:process`、アプリプロセスのスレッド一覧 (UI/RenderThread/Binder/ART デーモン)、スレッド優先度と
  task profiles・cgroup (v1/v2 の混在)、プロセス状態と oom_score_adj、freezer、seccomp/SELinux ドメイン
- `aidl-interfaces.md` — AIDL：`.aidl` から生成される Stub/Proxy と onTransact、in/out/inout・oneway・
  Parcelable、Java/C++/NDK/Rust バックエンド、stable AIDL (`@VintfStability`・凍結 API・版管理)、
  Binder 越しの例外、`@EnforcePermission`、Parcel 不整合型の脆弱性
- `system-server-services.md` — system_server と SystemService：SystemServer の起動段 (bootstrap/core/other/apex)、
  SystemServiceManager と boot phase、publishBinderService と LocalServices、内部スレッドと大域ロック、
  Watchdog とランタイム再起動、呼び出し元 ID と権限検査
- `window-manager.md` — WindowManagerService：WindowContainer 階層 (RootWindowContainer → DisplayContent →
  Task → ActivityRecord → WindowState)、ウィンドウ種別と Z 順、addWindow/relayout と SurfaceControl、
  フォーカスと入力への窓情報、insets、Shell Transitions、マルチウィンドウ、オーバーレイ攻撃対策

## 読む順序 (Reading Path)

下の層から順に読むと、各層が直下の層の機構だけで説明できるように並べてある。
[`platform-architecture.md`](platform-architecture.md) は全体の見取り図として最初に、
[`security-model.md`](security-model.md) は全層を横断する観点として最後に読む。

```text
1. Linux process / thread      → process-thread-model.md
        ↓                         （一般論: ../operating-systems/process-thread-scheduling.md）
2. Binder                      → binder-internals.md
        ↓
3. AIDL                        → aidl-interfaces.md
        ↓
4. system_server / SystemService → system-server-services.md
        ↓                         （起動まで: boot-init-properties.md ／ PMS: package-management.md）
5. Zygote / ART                → art-runtime-internals.md
        ↓                         （メモリと kill: memory-lmkd.md）
6. Activity / Task             → app-components-lifecycle.md
        ↓                         （全行程: app-launch-end-to-end.md）
7. Looper / Handler            → looper-handler-message-loop.md
        ↓
8. Window / Input / Graphics   → window-manager.md → view-system-layout.md
                                  → input-pipeline.md → graphics-pipeline.md
```

各ドキュメントは、有野和真『Androidを支える技術〈I〉〈II〉』(Android 7.0 Nougat 基準) を機構の基準線とし、
現行 AOSP との差分を「Nougat → 現在」の表で明示している。章とドキュメントの対応・読み替えガイドは
[`../../references/books/book-android-internals-arino.md`](../../references/books/book-android-internals-arino.md) を参照。

## 今後のテーマ候補

- Android Automotive OS — 車載への展開（`../../engineering/automotive/` との合流点）
- メディアスタック — Codec2、MediaSession、Stagefright 以後のプロセス分割の詳細
