# View システムとレイアウト (View Tree, Resources, Measure/Layout & ViewRootImpl)

## 概要 (Overview)

アプリの画面は「XML で書いた部品の木」として語られるが、プラットフォーム内部から見ると別の構造が見える: **(1) コンパイル済みバイナリリソースを mmap して高速に木を復元するデシリアライザ (LayoutInflater / AssetManager)、(2) 親が子に「許される大きさ」を 32bit 整数で渡す 2 パスの制約伝播系 (measure / layout)、(3) その木を 1 つのウィンドウとして WindowManagerService (WMS、[`window-manager.md`](window-manager.md)) に登録し、VSYNC ごとに走査するドライバ (ViewRootImpl)**。本書はこの 3 層を、`setContentView()` から最初の `performTraversals()` までの流れで通しに追う。

本ドキュメントが答える問い: なぜレイアウト XML は APK の中でテキストではなくバイナリなのか。なぜ `layout_weight` や入れ子の `RelativeLayout` は遅いのか。ViewRootImpl はなぜ View ではないのか。`requestLayout()` はどうやって画面の再描画に化けるのか。そして 2016 年 (Nougat) の設計の上に、Jetpack Compose・ConstraintLayout・edge-to-edge 強制 (Android 15〜) がどう積み増され、何が変わっていないのか。描画以降 (RenderThread・BLAST・SurfaceFlinger) は [`graphics-pipeline.md`](graphics-pipeline.md) に委ねる。分析深度は Level 3。

## 方法論 (Methodology)

- ベースラインは有野 (2017) Ⅰ巻 第 4 章 (View ツリーとレイアウト)・6.5 節 (ViewRootImpl)、Ⅱ巻 Appendix (View ツリーの状態保存) とする。同書は Android 7.0 Nougat (android-7.0.0_r6) 準拠で、掲載コードは著者による簡略化擬似コードであるため、機構の骨格の把握に用い、現行の詳細は AOSP ソースと Android Developers の公式文書で確認する。
- 現在の状態は AOSP `frameworks/base/core/java/android/view/` (`View`・`ViewGroup`・`ViewRootImpl`・`WindowManagerGlobal`・`LayoutInflater`・`IWindow.aidl`・`IWindowSession.aidl`) と、Android 15/16/17 の behavior changes、Compose / AAPT2 の公式文書に置く (Accessed: 2026-09-27)。バージョン依存の機構には導入版を付記する。
- 「データ (リソース・状態) はどの形式で・どのプロセス境界を通るか」「制約はどちら向きに流れるか (親→子／子→親)」を常に区別する。前者は [`binder-internals.md`](binder-internals.md)、後者は Compose との比較の軸になる。
- 入力配送 (InputChannel → dispatchTouchEvent) は [`input-pipeline.md`](input-pipeline.md)、メインスレッドの Looper/同期バリアは [`looper-handler-message-loop.md`](looper-handler-message-loop.md) に詳説を譲り、本書では接続点のみ扱う。

## 発見事項 (Findings)

1. **View ツリーは Composite パターンによる「領域の委譲」である。** 親は常に子の領域を包含するので、ヒット判定・描画・レイアウトはいずれも「自分の直接の子にだけ仕事を振る」再帰で書ける (有野 (2017) Ⅰ巻 4.1)。ViewGroup は外から見れば「自分の矩形を処理する View」にすぎず、階層の深さは呼び出し側から隠蔽される。この単純さが、後述の「多重 measure が入れ子で指数的に膨らむ」問題の裏面でもある。
2. **リソースがバイナリなのは「実行時パース」を前提に、そのコストをゼロに近づけるためである。** Android は 1 つの APK を画面密度・言語・向きの異なる無数の端末で動かすため、代替リソースの選択を**ビルド時ではなく実行時**に行う必要がある。そこで aapt/aapt2 は XML を属性 ID・型付き整数に前処理したバイナリ XML と `resources.arsc` に変換し、`ResXMLParser` はオフセットを進めるだけで木を辿る。Android 11 (API 30) 以降、`resources.arsc` は非圧縮・4 バイト境界整列でなければインストールできない——**mmap してページキャッシュを全プロセスで共有する**ことが設計の前提として明文化された。
3. **AttributeSet は「現在位置を持つ 1 個のパーサ」を全 View で使い回している。** LayoutInflater は要素ごとに属性オブジェクトを作らず、同一の XmlResourceParser (= AttributeSet) をカーソルだけ進めて各 View のコンストラクタに渡す (有野 (2017) Ⅰ巻 4.3.6)。GC を起こさない設計であり、コンストラクタの外で AttributeSet を保持してはならない理由でもある。スタイル・テーマの解決は `obtainStyledAttributes()` が JNI 越しに一括で行い、`TypedArray` で返す。
4. **measure は「親→子の制約 (MeasureSpec)」と「子→親の申告 (setMeasuredDimension)」の 2 方向で、layout は位置を決めるだけである。** MeasureSpec はモード 2bit (UNSPECIFIED / EXACTLY / AT_MOST) とサイズ 30bit を 1 つの int に詰める。LinearLayout の weight・RelativeLayout・wrap_content 親の下の weight 子などは同じ子を**複数回 measure** し、それが入れ子になると回数が乗算される ("double taxation")。Compose はこれを**型システムと実行時例外で禁止**した (子の measure は 1 回、事前情報は intrinsics で問う)。
5. **ViewRootImpl は View ではなく、ツリーとウィンドウシステムの間の「アダプタ兼ドライバ」である。** `WindowManagerGlobal.addView()` で生成され、`IWindowSession` (アプリ→WMS、同期) でウィンドウを登録・relayout し、`IWindow` (WMS→アプリ、**oneway**) でリサイズ・インセット変化を受ける。ツリーから見れば「親」だが、実体は Choreographer の TRAVERSAL コールバックで `performTraversals()` (measure → layout → draw) を回すスケジューラである。
6. **`requestLayout()` / `invalidate()` は「親を辿って ViewRootImpl に届くフラグ」であり、即時には何もしない。** 各 View はフラグを立てて親に伝播し、根の ViewRootImpl が `scheduleTraversals()` で次の VSYNC に 1 回だけ走査を予約する。何度呼んでもフレームあたり 1 回にまとめられる (coalescing) のは、この「汚れ印 + 予約」設計による。同時に `checkThread()` がメインスレッド以外からの呼び出しを拒否する——単一スレッド UI モデルの関所である。
7. **2016 年の骨格はほぼそのまま残り、変化は「上に載るもの」と「ウィンドウ境界の契約」に集中した。** MeasureSpec・2 パス・DecorView/`android:id/content`・ViewRootImpl・saveHierarchyState はそのままで、Compose も `ComposeView` という 1 個の View として ViewRootImpl の下に住む。一方、ウィンドウとアプリの境界では、edge-to-edge が API 35 で既定化・API 36 でオプトアウト不能となり、大画面では API 36/37 で向き・リサイズ制限が無視される——**「アプリは自分の矩形とインセットを動的に受け入れよ」**という契約の強化である。

## アーキテクチャ (Architecture)

### setContentView から最初のフレームまで

```mermaid
sequenceDiagram
    participant A as Activity (UI thread)
    participant PW as PhoneWindow / LayoutInflater
    participant AM as AssetManager (libandroidfw, mmap)
    participant WMG as WindowManagerGlobal
    participant VRI as ViewRootImpl
    participant WMS as WindowManagerService (system_server)
    participant CH as Choreographer
    A->>PW: setContentView(R.layout.main)
    PW->>PW: installDecor(): DecorView + screen_* テンプレートを inflate
    PW->>AM: getLayout(id) → XmlResourceParser (バイナリ XML)
    PW->>PW: Factory2 → createView() で各タグを生成、android:id/content 下に接続
    Note over A: onResume 後 ActivityThread.handleResumeActivity
    A->>WMG: addView(decorView, WindowManager.LayoutParams)
    WMG->>VRI: new ViewRootImpl(); setView(decor)
    VRI->>VRI: requestLayout() → scheduleTraversals()
    VRI->>WMS: IWindowSession.addToDisplayAsUser(IWindow, attrs, InputChannel...)
    CH-->>VRI: VSYNC → CALLBACK_TRAVERSAL → performTraversals()
    VRI->>WMS: relayout() → 窓の frame / SurfaceControl / insets
    VRI->>VRI: performMeasure → performLayout → performDraw (→ RenderThread)
    WMS-->>VRI: IWindow.resized / insetsControlChanged (oneway)
```

- **制御は Binder・データは mmap**: レイアウト XML は Binder を通らず、APK 内の `resources.arsc` とバイナリ XML をアプリプロセスが直接 mmap して読む。WMS とのやり取りは窓の属性・frame・インセットといった「小さな制御データ」だけで、画素は [`graphics-pipeline.md`](graphics-pipeline.md) の BLAST 経路を通る。
- **二つの「窓」の混同に注意**: `PhoneWindow` (android.view.Window の実装、Activity の UI 側面を切り出した補助クラス) と `IWindow` (WMS が管理する窓の Binder 端点、実装は ViewRootImpl の内部クラス `W`) は無関係である (有野 (2017) Ⅰ巻 4.4 コラム)。前者はツリーを**組み立て**、後者はツリーを**窓として登録**する。

### ビュー階層の実体 (Nougat と現在で共通)

```
ViewRootImpl  ← View ではない。IWindow(W)・Surface・InputChannel を 1 つずつ保持
 └─ DecorView (FrameLayout 派生)  ← ツリーの最上位 View。Window Style に応じたテンプレートを inflate
     └─ LinearLayout 等 (screen_simple / screen_title …)
         ├─ ActionBar / title 用 View (テーマ次第で存在しない)
         └─ FrameLayout @android:id/content  ← "ContentParent"
             └─ setContentView した木の根 ("ContentRoot")
                 └─ … (Compose なら ComposeView → AndroidComposeView → LayoutNode 木)
```

## 技術詳細 (Technical Details)

### 1. コンパイル済みリソースと AssetManager — なぜバイナリか

- **型付き整数への前処理**: aapt が出力するバイナリ XML では、属性名は `android:orientation` → `0x010100c4` のような**リソース ID**に、値は (型, データ) の組に置換される。例えば `"horizontal"` は (type 0x10 = 整数, 0x0)、`"50dp"` は (type 0x05 = dimension, `0x3201`) で、上位ビットが数値 50、下位ニブルが単位 (0=px, 1=dp, 2=sp) を表す (有野 (2017) Ⅰ巻 4.2.3、`aapt dump xmltree` による観察)。`@dimen/...` や `?attr/...` のような**実行時にしか決まらない参照は型タグ付きで温存**し、構成 (密度・言語・向き) に応じた解決を AssetManager に委ねる。つまり「ビルド時に潰せるものはすべて潰し、端末依存の選択だけを残す」分業である。
- **ResXMLParser とオフセット走査**: Java の `XmlResourceParser` はネイティブ `ResXMLParser` (libandroidfw) の薄いラッパで、ノード間はヘッダのサイズを足すだけで移動できる。文字列比較なしに数値比較だけで進むため、`ListView`/`RecyclerView` の各行を毎回 inflate する設計が実用速度で成立した (有野 (2017) Ⅰ巻 4.3 コラム)。
- **aapt → aapt2**: AAPT2 は処理を **compile** (各リソースを `.flat` 中間形式へ、変更ファイルだけ再コンパイル可能) と **link** (全 `.flat` を統合して APK・`R` を生成) に分割し、App Bundle 向けには `--proto-format` で Protobuf 表現を出す。`--enable-sparse-encoding` は `resources.arsc` のエントリを疎表現にし、サイズ・メモリ・起動遅延を減らす代わりに個々の参照をわずかに遅くする——ここでも最適化対象は「実行時に mmap された表の常駐コスト」である。
- **mmap 前提の明文化 (Android 11〜)**: targetSdk 30 以上では、圧縮された、または 4 バイト境界に整列していない `resources.arsc` を含む APK はインストールできない。圧縮されていると mmap できず各プロセスが RAM に展開する必要があるためで、**リソース表は全アプリで共有されるページキャッシュ**であるという Nougat 期からの設計意図が、インストール時検証に格上げされた。

### 2. LayoutInflater・Factory2・テーマ解決

- **タグ → クラス**: `createViewFromTag()` はタグ名に `.` がなければ `onCreateView()` (Phone 用実装が `android.widget.`・`android.webkit.`・`android.app.` を順に前置) を、あれば完全修飾名で `createView()` を呼ぶ。クラスはリフレクションでロードし、`(Context, AttributeSet)` コンストラクタを取り出して**静的マップにキャッシュ**する (有野 (2017) Ⅰ巻 4.3.4–4.3.5)。カスタム View にこの 2 引数コンストラクタが必須なのはこのためである。
- **Factory2 による差し替え**: `createViewFromTag()` はまず `Factory2`/`Factory` に生成を委ねる。AppCompat は自らを Factory2 として登録し、`AppCompatViewInflater` が `Button`→`AppCompatButton`、`TextView`→`AppCompatTextView` など十数種を**タグはそのままにクラスだけ差し替える**。テーマ属性 `viewInflaterClass` で差し替え器自体を交換でき、Material Components はこれで `MaterialButton` 等へ置換する。書籍はこの層を扱わないが (当時も存在)、現在の Views アプリでは「XML に書いたクラスと実際に生成されるクラスが違う」のが常態であり、バックポートとデザインシステム適用の主要な挿入点になっている。
- **LayoutParams は親の語彙**: `layout_` で始まる属性は子に書かれるが、解釈するのは親である。inflater は子の生成時に親の `generateLayoutParams(attrs)` を呼び、親固有の LayoutParams サブクラス (LinearLayout なら weight 付き) を子にぶら下げる (有野 (2017) Ⅰ巻 4.8.1)。子クラスは weight の存在を知らない——**レイアウト規則を親の実装に閉じ込める**設計で、ConstraintLayout もこの枠内で制約を LayoutParams として受け取る。
- **スタイル解決の優先順位**: `obtainStyledAttributes(attrs, R.styleable.X, defStyleAttr, defStyleRes)` は「XML 直書き → `style=` 属性 → テーマ中の defStyleAttr が指すスタイル → defStyleRes → テーマ直下の値」の順で解決し、必要な属性群を**1 回の JNI 呼び出しで配列にまとめて**返す。`TypedArray.recycle()` はこの配列をプール再利用するための作法である。テーマとスタイルの唯一の違いは、`windowNoTitle` などの Window Style がテーマ経由でしか効かず、DecorView 下のテンプレート選択 (`screen_simple`・`screen_title` 等) に使われる点である (同 4.4.2)。

### 3. MeasureSpec と 2 パスレイアウト — 何がコストを生むか

```text
// 擬似コード (AOSP View.MeasureSpec と等価の要約; 仮定: 32bit int, サイズ < 2^30)
// 出典: core/java/android/view/View.java line 31324 `MODE_SHIFT = 30` (aosp-mirror snapshot (2025-11) の main)
MODE_SHIFT = 30
UNSPECIFIED = 0 << 30   // 制約なし (ScrollView の中身など)
EXACTLY     = 1 << 30   // 固定値 or match_parent 確定時
AT_MOST     = 2 << 30   // wrap_content: 上限だけ与える
makeMeasureSpec(size, mode) = (size & ~MODE_MASK) | (mode & MODE_MASK)
// 計算量: エンコード/デコードは O(1)、割り当てゼロ
```

- **なぜ int 1 個か**: measure はフレームごとに全ノードで呼ばれうる最頻出経路であり、制約をオブジェクトにすると GC 圧になる。モードとサイズを 1 語に詰めることで、制約伝播をスタック上の値渡しだけで完結させている (Compose の `Constraints` も inline value class で同じ工夫をしている)。
- **子のモードは「親のモード × 子の LayoutParams」の表で決まる**: 例えば親が EXACTLY 700px で子が `wrap_content` なら子には AT_MOST(残り幅)、`match_parent` なら EXACTLY(残り幅) が渡る (`ViewGroup.getChildMeasureSpec()`)。子は受けた制約の範囲でサイズを計算し `setMeasuredDimension()` で申告し、親はその合計から自分のサイズを決める (有野 (2017) Ⅰ巻 4.6–4.7)。
- **多重 measure の発生源**: weight 付き LinearLayout は「weight なしの子を先に measure → 余りを weight 比で配り、weight 子を EXACTLY で再 measure」する。親が `wrap_content` なら、最大サイズで一度測って本来のサイズを得たうえで再度測る (同 4.8 コラム)。RelativeLayout は依存関係の解決のため水平・垂直で 2 回測る。公式文書はこれを **double taxation** と呼び、ルート付近・深い階層・RecyclerView の各行で致命的になるとする。入れ子の深さ d で各段が 2 回測れば最悪 O(2^d) 回の measure になる。
- **ViewRootImpl 側の追加測定**: `performTraversals()` は `WRAP_CONTENT` の窓 (ダイアログ等) では `measureHierarchy()` で好ましい幅を試しながら複数回測ることがあり、`View.measure()` は `(widthSpec, heightSpec)` をキーにした measure キャッシュ (`mMeasureCache`, `LongSparseLongArray`) と `PFLAG_FORCE_LAYOUT` フラグで「同じ制約・汚れなし」の再測定を省く (`ViewRootImpl.measureHierarchy` は `core/java/android/view/ViewRootImpl.java` line 3264、`View.measure` は `View.java` lines 28320-28402 でキャッシュ参照は `forceLayout ? -1 : mMeasureCache.indexOfKey(key)` (line 28358); いずれも aosp-mirror snapshot (2025-11) の main)。カスタム View の `onMeasure()` が**複数回呼ばれても最後の結果が正となるよう冪等に**書く必要があるのはこのためである。
- **layout と gravity**: measure 完了後の layout パスは各子の left/top を決めるだけで、`gravity` (自分の中身の寄せ方) と `layout_gravity` (親に対する自分の寄せ方、LayoutParams) はほぼここで処理される——ただし RelativeLayout のように onMeasure 内で計算する例外がある (同 4.8.2 コラム)。

### 4. ViewRootImpl・WindowManagerGlobal・IWindow/IWindowSession

- **生成と登録**: `ActivityThread.handleResumeActivity()` が `getWindow().getDecorView()` を `WindowManager.addView()` に渡し、アプリプロセス内のシングルトン `WindowManagerGlobal` が ViewRootImpl を生成して `setView()` する (有野 (2017) Ⅰ巻 6.5.2)。`setView()` は `IWindowSession.addToDisplayAsUser()` (同期 Binder) で内部クラス `W` (IWindow の実装) を WMS に登録し、InputChannel を受け取る。以後、1 ViewRootImpl につき窓・Surface・InputChannel が 1 組対応する。`onCreate()` 終了までに `setContentView()` を呼ぶ慣習は、この時点で DecorView が揃っている必要から来る。
- **非対称な Binder 契約**: `IWindowSession` の `relayout()` は「新しい窓パラメータを渡し、画面上の frame と surface を返す」同期呼び出しだが、現行 AIDL では `relayoutAsync` (line 110)・`finishDrawing` (140)・`updateRequestedVisibleTypes` (271) など多くが oneway になった (`core/java/android/view/IWindowSession.aidl`, aosp-mirror snapshot (2025-11) の main)。逆向きの `IWindow` は**インタフェース全体が oneway** (`IWindow.aidl` line 44 `oneway interface IWindow`) で、`resized()`・`insetsControlChanged()`・`dispatchAppVisibility()` などを WMS が一方的に通知する。system_server が任意のアプリに引きずられないという [`app-components-lifecycle.md`](app-components-lifecycle.md) と同じ原理が窓の層でも貫かれている。
- **Surface の出自**: 書籍は Java の `mSurface` と native バッファの接続 (relayout 経由の SurfaceControl 生成) を「追い切れなかった」と率直に記している (同 6.5.4 コラム)。現在は relayout が `SurfaceControl` を返し、ViewRootImpl がそれを親に **BLASTBufferQueue** を作ってバッファを Transaction で提出する構造が明確になっている (Android 12 頃〜、詳細は [`graphics-pipeline.md`](graphics-pipeline.md))。
- **performTraversals の骨格**: 初回・窓サイズ変更・インセット変化時は `relayoutWindow()` を挟み、`performMeasure()` → `performLayout()` (layout 中の追加 `requestLayout()` は第 2 ラウンドで処理) → `performDraw()` (ThreadedRenderer 経由で表示リストを記録し RenderThread へ) の順に進む。入力は同じ ViewRootImpl の InputChannel から `dispatchPointerEvent()` → `dispatchTouchEvent()` の再帰に入る (同 6.5.6、[`input-pipeline.md`](input-pipeline.md))。

### 5. requestLayout / invalidate の伝播

```
View.requestLayout()                         View.invalidate()
  PFLAG_FORCE_LAYOUT を立てる                   dirty 矩形 / PFLAG_DIRTY を立てる
  mParent.requestLayout() … 再帰で上へ          parent.onDescendantInvalidated() … 上へ (HW 描画時, API 26〜)
        ▼                                            ▼
ViewRootImpl.requestLayout()                  ViewRootImpl.onDescendantInvalidated()/invalidate()
  checkThread()  ← 別スレッドなら CalledFromWrongThreadException
  mLayoutRequested = true
        └──────────── scheduleTraversals() ────────────┘
             MessageQueue.postSyncBarrier()   ← 以降の同期メッセージを堰き止め
             Choreographer.postCallback(CALLBACK_TRAVERSAL, mTraversalRunnable)
                  └─ 次の VSYNC で performTraversals() が 1 回だけ走る
```

- **coalescing と優先度**: `ViewRootImpl.scheduleTraversals()` (`ViewRootImpl.java` lines 2980-2983, aosp-mirror snapshot (2025-11) の main) は `mTraversalScheduled` を立てたうえで `mHandler.getLooper().getQueue().postSyncBarrier()` を呼ぶ。`mTraversalScheduled` により 1 フレーム内の多数の要求は 1 回の走査にまとまる。同期バリアは、走査が VSYNC に遅れないよう通常の Handler メッセージ (アプリのポスト) を一時的に後回しにし、非同期フラグ付きの VSYNC メッセージだけを通す仕組みで、[`looper-handler-message-loop.md`](looper-handler-message-loop.md) の同期バリアの最大の利用者である。
- **invalidate は measure を起こさない**: 見た目だけの変化は `invalidate()` で draw だけを、サイズが変わりうる変化 (テキスト長の変化など) は `requestLayout()` で measure/layout も要求する。RenderNode のプロパティ (translation・alpha) の変更は再記録すら不要で RenderThread 側で反映される——**変化の種類ごとに再実行範囲を最小化する**という原則は、Compose の「状態を読んだフェーズだけ再実行」に一般化される (§7)。

### 6. View ツリーの状態保存

- **経路**: `Activity.onSaveInstanceState()` の既定実装は `PhoneWindow.saveHierarchyState()` を呼び、これが `mContentParent.saveHierarchyState(SparseArray<Parcelable>)` でツリーを再帰し、フォーカス中の View の ID なども Bundle に詰める (有野 (2017) Ⅱ巻 Appendix A.1)。各 View の `dispatchSaveInstanceState()` は **ID を持ち、かつ `saveEnabled` が偽でない**ときだけ `onSaveInstanceState()` の結果を ID をキーに格納する。EditText の入力が何もしなくても回転後に残るのはこの既定動作による。
- **ID がキーであることの帰結**: 同じ ID の View が複数あると復元が上書きされ、ID のない View は保存されない。保存物は Parcelable → Bundle → Binder で system_server の ActivityRecord に退避されるため、1MB のトランザクション上限と「プロセス死後も型が解決できること」の制約を受ける ([`app-components-lifecycle.md`](app-components-lifecycle.md) §3、[`binder-internals.md`](binder-internals.md))。
- **現在**: 経路自体は不変で、Fragment 状態・`SavedStateRegistry` (Jetpack) が同じ Bundle に相乗りする。Compose の `rememberSaveable` は `SaveableStateRegistry` を介してこの Bundle に保存されるので、**View の ID の代わりに composition 内の位置 (キー) で状態を対応づける**点だけが異なる。

### 7. 現在 — Jetpack Compose は ViewRootImpl の上で何を置き換えたか

- **ホスティング**: `ComponentActivity.setContent {}` は `ComposeView` を `setContentView()` し、その中の `AndroidComposeView` (ViewGroup) が LayoutNode 木の根となる。したがって窓・インセット・入力・描画のドライバは依然として ViewRootImpl と Choreographer であり、Compose は「ContentRoot 以下の 1 個の巨大な View」として 2 パスの外枠に収まる。`AndroidComposeView.onMeasure()/onLayout()` が内部の measure/layout を駆動し、合成 (composition) は Recomposer が Choreographer のフレームに同期して行う。
- **3 フェーズと再実行範囲**: Compose は composition (何を出すか) → layout (measure と placement) → drawing の 3 フェーズを持ち、状態を**どのフェーズで読んだか**によって再実行範囲が決まる (placement で読めば measure は再実行されない)。View の requestLayout/invalidate の区別を、開発者の手動呼び出しから**スナップショット状態の読み取り追跡**に自動化したものと読める。
- **単一パス measure と intrinsics**: 公式文書は「子は 1 回しか measure してはならず、2 回測ると実行時例外」と明記し、各ノードを 1 回訪問するので木の大きさに対して線形時間になるとする。weight 的な「先に中身を知りたい」要求は **intrinsics** (`IntrinsicSize.Min/Max`) で、本 measure の前に問い合わせる。これは View の double taxation を**API の段階で表現不能にした**設計変更である (ただし intrinsics 自体は子の問い合わせコストを持つので、濫用すれば同種のコストは戻る)。
- **ConstraintLayout (Views 側の回答)**: 入れ子をやめて 1 階層の制約系にし、制約ソルバで位置とサイズを解く。公式の性能ガイドも RelativeLayout・weight 付き LinearLayout の入れ子を避け ConstraintLayout で平坦化することを推奨している。「多重 measure を減らす」ではなく「多重 measure が乗算される入れ子自体を無くす」アプローチである。

### 8. 現在 — ウィンドウ境界の契約: edge-to-edge とインセット

- **Nougat の前提**: 書籍の `screen_title.xml` のテンプレートは `android:fitsSystemWindows="true"` を持ち、システムバーの分だけ DecorView 内で内容を避ける作りだった (有野 (2017) Ⅰ巻 4.4.2)。
- **Android 15 (API 35 target)**: edge-to-edge が既定となり、ステータスバー/ジェスチャナビは透明、内容はインセットを適用しない限りバーの背後に描かれる。`setStatusBarColor`・`setDecorFitsSystemWindows` 等は非推奨かつ無効化され、ディスプレイカットアウトも非フローティング窓では `ALWAYS` と解釈される。`Configuration` の画面サイズもシステムバーを差し引かなくなった。
- **Android 16 (API 36 target)**: オプトアウト属性 `windowOptOutEdgeToEdgeEnforcement` が無効化。さらに最小幅 600dp 以上では `screenOrientation`・`resizableActivity`・`min/maxAspectRatio` が無視され、窓いっぱいに表示される (一時的オプトアウトは API 37 target で廃止、Android 17 は 2026-06 に安定版)。
- **意味**: View ツリーから見ると、根に渡る MeasureSpec (窓サイズ) と `WindowInsets` が**任意に・頻繁に変わる入力**であることが強制された。Nougat のマルチウィンドウで `IWindow.resized()` の出番が増えたと書籍が述べた変化 (同 6.5.3) の延長線上にあり、「窓は固定サイズの画面」という前提は完全に捨てられた。

### 9. 何が変わり、何が変わっていないか (2016 Nougat → 現在)

| 機構 | Nougat (書籍) | 現在 | 変化の理由 |
|---|---|---|---|
| リソースコンパイラ | aapt (一括処理) | aapt2: compile(.flat)/link 分割、Protobuf (AAB)、sparse encoding | インクリメンタルビルド、App Bundle 配信、常駐メモリ削減 |
| `resources.arsc` の格納 | mmap が設計意図 (暗黙) | API 30+ は非圧縮・4B 整列必須 | mmap 不能時の RAM 展開を禁止し共有ページを保証 |
| バイナリ XML・AttributeSet | ResXMLParser のカーソル共有 | 不変 | 実行時の代替リソース選択という前提が不変 |
| View 生成 | PhoneLayoutInflater の接頭辞探索 + リフレクション | 不変 + Factory2 による AppCompat/Material 置換が常態 | バックポートとデザインシステムの一括適用 |
| 制約表現 | MeasureSpec (2bit mode + 30bit size) | 不変 (Compose は Constraints: min/max 幅高) | 最頻出経路で割り当てを避ける設計が有効なまま |
| 多重 measure | weight / RelativeLayout で発生、開発者の注意任せ | Views では ConstraintLayout で平坦化、Compose は 2 回目を例外化・intrinsics | 入れ子で乗算されるコストを構造的に排除 |
| ツリー走査の駆動 | ViewRootImpl.performTraversals + Choreographer | 不変 (Compose も ComposeView として同じ走査に乗る) | 窓・入力・VSYNC の単一ドライバという役割は不変 |
| 無効化伝播 | invalidateChild の再帰 | `onDescendantInvalidated` (API 26〜; `ViewParent.java` の default メソッドとして android-8.0.0_r1 line 71 に初出、7.1.2_r1 には無い) による HW 描画向け経路 | 表示リスト再記録範囲の最小化 |
| WMS との契約 | addToDisplay / relayout (同期中心) | addToDisplayAsUser、relayoutAsync 等の oneway 化、SurfaceControl + BLAST | マルチユーザ、非同期化、窓とバッファの原子的更新 |
| システムバーとの関係 | fitsSystemWindows のテンプレートが避ける | edge-to-edge 強制 (API 35)、オプトアウト不能 (API 36) | 全画面・透過バー・折りたたみ端末での一貫性 |
| 窓サイズ | マルチウィンドウ導入直後 (resize 増加) | 大画面では向き・比率制限を無視 (API 36/37) | 大画面・デスクトップ窓化への適応を必須化 |
| 状態保存 | saveHierarchyState → SparseArray (ID キー) → Bundle | 不変 + SavedStateRegistry / rememberSaveable が相乗り | 輸送 (Bundle/Binder) の制約が不変 |

## セキュリティ考察 (Security Considerations)

- **窓の追加は権限である**: `WindowManager.addView()` を他アプリの上に向けて呼ぶには `SYSTEM_ALERT_WINDOW` と設定画面での許可が要り (Android 6.0〜、書籍 6.6.1)、Android 8.0 以降は `TYPE_APPLICATION_OVERLAY` 型に限定された。オーバーレイは UI 偽装・タップジャッキングの主要手段であり、`filterTouchesWhenObscured` や Android 12 の「信頼できないオーバーレイ越しのタッチ遮断」は、IWindow 単位の Z 順を WMS が握っていることを利用した対策である（WMS の窓階層は [`window-manager.md`](window-manager.md)） ([`graphics-pipeline.md`](graphics-pipeline.md)・[`security-model.md`](security-model.md))。
- **リソースパーサは不信入力を読む**: `resources.arsc` とバイナリ XML は APK から来る攻撃者制御のバイト列で、それを解釈する libandroidfw (C++) はアプリだけでなく PackageManager・ランチャー・system_server からも呼ばれる。オフセット走査という高速化の設計は、境界検査の欠落が即メモリ破壊になる設計でもあり、ネイティブパーサの堅牢化・ファジング ([`../security/fuzzing-program-analysis.md`](../security/fuzzing-program-analysis.md)) の対象領域である。インストール時の構造検証 (整列要求など) は性能要件であると同時に入口の正規化として働く。
- **インフレーションはコード実行である**: レイアウトのタグ名はクラス名としてリフレクションでロードされる。通常は自 APK 内のリソースだが、`createPackageContext()` で他パッケージのレイアウトを inflate すると相手のクラスを自プロセスで実行しうる (`CONTEXT_INCLUDE_CODE` を伴う場合)。RemoteViews が inflate 可能なクラスを `@RemoteView` 注釈付きの許可リストに限定しているのは、この経路を閉じるためである。
- **状態保存は漏洩経路になりうる**: saveHierarchyState は ID 付きの EditText の内容を自動で Bundle に入れ、Bundle は system_server に退避される。機密を表示する EditText や独自 View の `onSaveInstanceState()` は、明示しない限り内容をそのまま保存対象にしうるため、意図せず永続・転送される。`saveEnabled=false` と `FLAG_SECURE` (最近のタスクのサムネイル・スクリーンショット抑止) は別層の対策として併用する必要がある。
- **edge-to-edge とセキュリティ UI**: 内容がシステムバーの背後に描かれる既定化は、アプリがバー領域に「システム風」の UI を描ける余地を広げる。ステータスバー・ナビゲーションの実体は SystemUI の別窓で Z 順が上にあるため直接の乗っ取りにはならないが、偽の権限ダイアログ等の UI 偽装への注意はより重要になる。

## 結論 (Conclusions)

- Android の View システムの中核は、**実行時に代替リソースを選ぶ**という要件から逆算された「mmap 可能なバイナリ表 + カーソル共有パーサ + リフレクション生成」と、**親子の制約を int 1 個で伝える 2 パスレイアウト**である。どちらも「毎フレーム・毎行で呼ばれる経路から割り当てと文字列処理を消す」という同じ原理に立ち、2016 年から今日まで骨格は変わっていない。
- ViewRootImpl は View ではなく、ツリーを窓として WMS に登録し (IWindowSession 同期 / IWindow oneway)、`requestLayout`/`invalidate` の汚れ印を VSYNC ごとの 1 回の `performTraversals()` に集約するドライバである。単一スレッド検査と同期バリアは、このドライバが UI スレッドの Looper を占有するための仕組みである。
- 10 年間の変化は、骨格を置き換えるのではなく**上に載せる**形で起きた: AppCompat/Material は Factory2 でクラスを差し替え、ConstraintLayout は入れ子を平坦化し、Compose は ComposeView として ViewRootImpl の下に住みつつ、多重 measure を例外化し再実行範囲を状態読み取りから自動導出した。
- 最も強い変化はウィンドウ境界の契約にある。edge-to-edge の強制 (API 35/36) と大画面での向き・比率制限の無視 (API 36/37) は、根に渡る MeasureSpec とインセットが「任意に変わる入力」であることをアプリに受け入れさせた。これはライフサイクルにおける「プロセスはいつ殺されてもよい」と同型の、**環境の変動を例外ではなく常態として扱わせる契約**である。

### 次の接続先 (Related Topics)

- [`graphics-pipeline.md`](graphics-pipeline.md) — performDraw 以降: RenderThread、BLAST、SurfaceFlinger。
- [`input-pipeline.md`](input-pipeline.md) — InputChannel から ViewRootImpl、dispatchTouchEvent の再帰まで。
- [`looper-handler-message-loop.md`](looper-handler-message-loop.md) — scheduleTraversals の同期バリアと Choreographer の実体。
- [`app-components-lifecycle.md`](app-components-lifecycle.md) — handleResumeActivity と保存状態 Bundle の旅。
- [`app-launch-end-to-end.md`](app-launch-end-to-end.md) — setContentView から最初のフレームまでを起動全体の中で。
- [`package-management.md`](package-management.md) — APK 検証 (resources.arsc 整列を含む) とリソースの出どころ。
- [`binder-internals.md`](binder-internals.md) — IWindow/IWindowSession の同期・oneway の機構的裏付け。
- [`../ios/core-animation-render-server.md`](../ios/core-animation-render-server.md) — 対比: レイヤーツリーをレンダーサーバに渡す iOS の設計。
- [`../browsers/browser-architecture.md`](../browsers/browser-architecture.md) — 対比: DOM/CSS レイアウトとの比較 (制約伝播と多重レイアウト)。
- [`window-manager.md`](window-manager.md) — IWindowSession/IWindow の相手側 (WMS) の内部。
- [`aidl-interfaces.md`](aidl-interfaces.md) — IWindow の `oneway interface` 等、AIDL 定義の読み方。

## 参考文献 (References)

**一次資料 (Primary — AOSP / Android 公式)**

- AOSP ソース: `frameworks/base/core/java/android/view/View.java` (`MeasureSpec`, `measure`, `requestLayout`, `dispatchSaveInstanceState`)、`ViewGroup.java` (`getChildMeasureSpec`, `generateLayoutParams`)、`ViewRootImpl.java` (`setView`, `scheduleTraversals`, `performTraversals`, `checkThread`)、`WindowManagerGlobal.java` (`addView`)、`LayoutInflater.java` (`createViewFromTag`, `Factory2`). — [cs.android.com](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/view/) — Accessed: 2026-09-27。
- AOSP, `IWindow.aidl` (oneway interface) ／ `IWindowSession.aidl` (`addToDisplayAsUser`, `relayout`, `relayoutAsync`). — [github.com/aosp-mirror/platform_frameworks_base](https://github.com/aosp-mirror/platform_frameworks_base/blob/main/core/java/android/view/IWindowSession.aidl) — Accessed: 2026-09-27 (ミラー取得時点の main。最新 AOSP とは差がありうる)。
- Android Developers, *View.MeasureSpec*. — [developer.android.com/reference/android/view/View.MeasureSpec](https://developer.android.com/reference/android/view/View.MeasureSpec) — Accessed: 2026-09-27。
- Android Developers, *AAPT2*. — [developer.android.com/tools/aapt2](https://developer.android.com/tools/aapt2) — Accessed: 2026-09-27。
- Android Developers, *Behavior changes: apps targeting Android 11* (resources.arsc の非圧縮・4 バイト整列要件). — [developer.android.com/about/versions/11/behavior-changes-11](https://developer.android.com/about/versions/11/behavior-changes-11) — Accessed: 2026-09-27。
- Android Developers, *Behavior changes: Android 15* (edge-to-edge 強制) ／ *Android 16* (オプトアウト無効化、大画面の向き・リサイズ) ／ *Android 17* (大画面オプトアウト廃止). — [developer.android.com/about/versions/15/behavior-changes-15](https://developer.android.com/about/versions/15/behavior-changes-15), [16](https://developer.android.com/about/versions/16/behavior-changes-16), [17](https://developer.android.com/about/versions/17/behavior-changes-17) — Accessed: 2026-09-27。
- Android Developers, *Jetpack Compose phases* ／ *Intrinsic measurements in Compose layouts*. — [developer.android.com/develop/ui/compose/phases](https://developer.android.com/develop/ui/compose/phases), [intrinsic-measurements](https://developer.android.com/develop/ui/compose/layouts/intrinsic-measurements) — Accessed: 2026-09-27。
- Android Developers, *Performance and view hierarchies* (double taxation, ConstraintLayout 推奨). — [developer.android.com/topic/performance/rendering/optimizing-view-hierarchies](https://developer.android.com/topic/performance/rendering/optimizing-view-hierarchies) — Accessed: 2026-09-27。
- Android Developers, *AppCompatViewInflater* (`viewInflaterClass`). — [developer.android.com/reference/androidx/appcompat/app/AppCompatViewInflater](https://developer.android.com/reference/androidx/appcompat/app/AppCompatViewInflater) — Accessed: 2026-09-27。

**二次資料 (Secondary)**

- 有野和真 (2017).『Androidを支える技術〈I〉60fpsを達成するモダンなGUIシステム』技術評論社. ISBN 978-4-7741-8759-4. 第 4 章 (View ツリー・リソース・LayoutInflater・DecorView・measure/layout)、6.5 節 (ViewRootImpl)。Android 7.0 準拠、掲載コードは簡略化擬似コード。
- 有野和真 (2017).『Androidを支える技術〈II〉真のマルチタスクに挑んだモバイルOSの心臓部』技術評論社. ISBN 978-4-7741-8861-4. Appendix A (View ツリーの状態保存)。
- Android Developers Blog, *Android 17 is here* (2026-06、安定版リリース). — [android-developers.googleblog.com/2026/06/Android-17.html](https://android-developers.googleblog.com/2026/06/Android-17.html) — Accessed: 2026-09-27。
