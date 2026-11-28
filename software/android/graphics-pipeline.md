# グラフィックスパイプライン (Android Graphics: SurfaceFlinger, BufferQueue & Frame Pacing)

## 概要 (Overview)

タップから画素までの経路、すなわち**アプリの描画 (Choreographer / RenderThread) → バッファの受け渡し (BufferQueue / gralloc) → 合成 (SurfaceFlinger / Hardware Composer) → 表示 (VSYNC)** を通しで追う。Android のグラフィックスは「GPU で速く描く」話である以上に、**プロセス境界をまたぐ生産者・消費者パイプラインの設計**であり、Binder（制御路）と fd/共有バッファ（データ路）の分業、VSYNC を基準にした全体の位相同期、フレーム落ち（ジャンク）を防ぐパイプライン深度の管理が主題になる。

本ドキュメントが答える問い: なぜ描画は「アプリが画面に描く」のではなく「アプリがバッファを提出し、別プロセスが合成する」構造なのか。16.6ms（60Hz）の予算はどの区間に配分されるのか。トリプルバッファリングは何を買い、何のレイテンシを払うのか。分析深度は Level 3。

## 方法論 (Methodology)

- 1 フレームのライフサイクル（VSYNC 受信 → UI スレッド → RenderThread → キュー → 合成 → 表示）を軸に、各段の担当プロセス・スレッドと受け渡される資源（バッファ・フェンス・トランザクション）を特定する。
- 「制御はどこを通るか（Binder）／画素データはどこを通るか（共有バッファ、コピーなし）」を常に区別し、[`binder-internals.md`](binder-internals.md) の「大きなデータは fd をハンドルとして渡す」の最大の適用例として読む。
- 一次資料は AOSP のグラフィックス公式ドキュメント (source.android.com/docs/core/graphics) に置き、バージョン依存の機構（BLAST 等）は導入版を付記する。
- GPU 内部（シェーダ実行・タイルレンダリング）は本書の対象外とし、GPU を「コマンドを投げるとフェンス付きでバッファを埋める装置」として扱う。
- 機構レベルの基準線として 有野 (2017) Ⅰ巻（Android 7.0 Nougat、AOSP android-7.0.0_r6 準拠）の第 5 章（DisplayList と RenderThread）・第 6 章（gralloc / EGL / SurfaceFlinger / HWC / ViewRootImpl）を用い、そこから現行 AOSP（`frameworks/base/libs/hwui`、`frameworks/native/services/surfaceflinger`、`hardware/interfaces/graphics`）までの差分を技術詳細 5 に表として整理する。同書のコード片は著者による簡略化擬似コードであり、本書では引用せず機構のみを言い換えて用いる。

## 発見事項 (Findings)

1. **画素データは一度もコピーされない。** アプリの GPU が描いたバッファ（gralloc 確保・dma-buf fd）は、ハンドルが SurfaceFlinger へ渡るだけで、合成もそのメモリを直接読む。Binder で運ばれるのは**バッファの所有権とメタデータ**であり、[`binder-internals.md`](binder-internals.md) の「1MB 制限、大物は fd で」の帰結が、そのままパイプラインの基本構造になっている。
2. **同期の実体は「フェンス」である。** バッファの受け渡しは GPU がまだ描き終わる前に行われ（非同期）、「いつ読んでよいか」は同期フェンス (sync fence, dma-fence の fd) が運ぶ。CPU 側のパイプラインと GPU 側のパイプラインを疎結合に保つ要であり、フェンス待ちの位置がそのままレイテンシ地図になる。
3. **BufferQueue は背圧付きのリングである。** 生産者 (アプリ) は空きバッファを dequeue できなければブロックする。これは描画が消費 (合成) を追い越せない構造的な流量制御で、oneway Binder の背圧と同じ文法である。バッファ数（2〜3）がパイプライン深度＝レイテンシとスループットのトレードオフを決める。
4. **合成は「GPU に戻す」か「ディスプレイに任せる」かの動的な最適化である。** Hardware Composer (HWC) はレイヤをディスプレイのオーバーレイプレーンに直接割り当て（GPU を起こさず合成）、収まらない分だけ SurfaceFlinger が GPU (client composition) で畳む。**電力最適化としての合成戦略**が毎フレーム決定されている。
5. **全員が同じ VSYNC を見ているが、同じ位相では動いていない。** アプリの描画開始・SurfaceFlinger の合成開始は、ハードウェア VSYNC からの**位相オフセット**でずらされ、「アプリが描き終わった直後に合成が始まり、合成が終わった直後に走査が始まる」よう予算が区画割りされる。DispSync/VSYNC モデルはソフトウェアで位相を合成する PLL である。
6. **BLAST (Android 12 で既定化) はバッファ提出をトランザクションに載せ替えた。** BufferQueue の暗黙のキューではなく、`SurfaceControl.Transaction` に「このバッファを・このジオメトリで・この仲間と同時に」を明示して原子的に適用する。ウィンドウリサイズ時の「中身と枠がずれる」古典的な非同期アーティファクトを、**複数プロセスの状態変更の原子コミット**として解いた。
7. **リフレッシュレートは変数になった。** 可変レート (48/60/90/120Hz) 時代の SurfaceFlinger はアプリのフレーム提出ヒント・コンテンツ種別からレートを投票制で選ぶ。フレームペーシング（Swappy / Frame Pacing ライブラリ）は「速く描けたフレームを早く出しすぎない」制御であり、**速さではなく等間隔性がなめらかさ**であることを機構化している。
8. **Nougat (2016) から変わったのは「部品の実装」であり「分業の構造」ではない。** 記録（UI スレッド）と実行（RenderThread）の分離、View 単位の RenderNode キャッシュ、gralloc/EGL/HWC という端末依存の継ぎ目、「HWC で合成できる分だけ HWC、あふれを GPU」という二段合成は 有野 (2017) Ⅰ巻 第 5・6 章の記述のまま生きている。一方、表示リストの中身（独自 Op → Skia 記録）、HAL の形式（`hw_module_t` の .so → HIDL → AIDL / stable-C）、BufferQueue の所在（SurfaceFlinger 側 → アプリ側 BLAST）、リフレッシュレート（固定 60Hz → 可変）は総入れ替えされた（技術詳細 5）。

## アーキテクチャ (Architecture)

### 全体像 — 二つの経路（制御と画素）

```
 [アプリプロセス]                          [system_server]        [surfaceflinger]
 UI スレッド ── measure/layout/record ─┐    WindowManager ◀──── ウィンドウ状態 (Binder)
 RenderThread ── Skia/HWUI → GL/Vulkan ─┤
      │ gralloc バッファに GPU 描画      │
      ▼                                 ▼
 BufferQueue/BLAST: バッファ + フェンス + Transaction ──(Binder: ハンドルのみ)──▶ 合成へ
                                                                    │
                                                     HWC HAL (ベンダ): レイヤ割当
                                                       ├─ オーバーレイ (DPU 直結)
                                                       └─ あふれ分 → GPU 合成
                                                                    ▼
                                                          ディスプレイ走査 (VSYNC)
```

- **描画と合成は別プロセス**: アプリがクラッシュ・停止しても画面全体は SurfaceFlinger が守る。逆に SurfaceFlinger は「全アプリのバッファを見られる」特権プロセスであり、隔離の要でもある。
- **WindowManager (system_server) は制御面のみ**（詳細は [`window-manager.md`](window-manager.md)）: どの Surface がどの Z 順・領域にあるかを決め、SurfaceControl として surfaceflinger に指示する。画素は一切通らない。

### 1 フレームの予算配分（60Hz・位相オフセットの考え方）

```
 HW VSYNC ─┬────────────── 16.6ms ──────────────┬─▶
           │ +app offset                         │
           │   Choreographer#doFrame:            │
           │   input → animation → traversal     │
           │   RenderThread: record→GPU submit   │
           │        +sf offset                   │
           │          SurfaceFlinger: latch      │
           │          (acquire fence 待ち) → 合成 │
           │                              走査 ──┘ 表示は 1〜2 VSYNC 後
```

- アプリ描画・合成が各々 1 VSYNC 期に収まる標準構成では、**タッチから表示まで 2〜3 フレーム**のパイプラインレイテンシが構造的に存在する。オフセットを詰めればレイテンシは減るが、締切に間に合わないリスク（ジャンク）が増える。つまり、位相設計そのものがレイテンシ/安定性のトレードオフである。

## 技術詳細 (Technical Details)

### 1. アプリ側 — Choreographer と RenderThread

- **Choreographer** は VSYNC イベント（surfaceflinger からアプリへ配られる）を受け、フレームコールバックを **input → animation → traversal (measure/layout/draw)** の固定順で実行する。「VSYNC に揃えて 1 フレーム分の仕事を 1 回だけやる」ための基準クロック配布機構。
- **描画の記録と実行の分離**: UI スレッドは View 階層を**表示リスト (display list) に記録するだけ**で、GPU コマンド発行は **RenderThread** が行う（HWUI、実体は Skia。バックエンドは GL または Vulkan、近年は Vulkan が既定方向、GL は ANGLE で Vulkan 上に実装する方向）。UI スレッドのブロック（＝入力応答性）と GPU 待ちを分離するための二段パイプライン。
- **アニメーションの委譲**: リング進捗のような定型アニメーションは RenderThread 単独で進められ（Animated Vector 等）、UI スレッドがジャンクしても止まらない。
- **なぜそもそも GPU で GUI を描くのか（電力と解像度）**: 有野 (2017) Ⅰ巻 5.1 は、画面合成・ピクセル操作では各画素が独立しており、単純な演算を大量に並列実行できる点に理由を置く。CPU は汎用命令のために 1 実行ユニット内の直列ゲート段数が多く、同じクロックを出すには電圧を上げる必要がある（CMOS の動的電力は概ね C·V²·f に比例する）。GPU/HWC は段数の少ないユニットを多数並べるため、同じ仕事を低電圧・少ゲート・低クロックでこなせる。解像度が G1 (320×480) から Nexus 6P (1440×2560) で約 24 倍になったことが、この差を「あれば嬉しい」から「必須」に変えた。代償はメモリ消費（表示リスト・バッファの保持）で、同書はハードウェア描画化した ICS 以降の方が必要スペックが高いことも認めている。
- **GUI システムの三つの時代**: 有野 (2017) Ⅰ巻 1.2 は現行 GUI を、それが主に作られた版で三層に分ける。すなわち、(1) Android 初期〜Gingerbread: View ツリー・measure/layout・入力（InputManagerService と Looper）の基礎、(2) Honeycomb〜ICS: DisplayList と OpenGL ES によるハードウェアアクセラレーション描画（本書の主対象）、(3) Lollipop〜Nougat: ART の AOT → JIT/プロファイルガイド併用による引っ掛かり対策である。描画の議論が GC・JIT に及ぶのは、第三の時代がフレーム予算の (a) 区間（技術詳細 4）を守るための改良だったからである。基礎層は [`view-system-layout.md`](view-system-layout.md)・[`looper-handler-message-loop.md`](looper-handler-message-loop.md)・[`input-pipeline.md`](input-pipeline.md) が扱う。
- **再記録 (re-record) と再発行 (re-issue) の区別**: 表示リストは描画 API 呼び出しをコマンドオブジェクト化した列（Command パターン）で、View のサブツリーごとに RenderNode に分割保持され、親は子を、別の RenderNode の描画を指示する 1 ノードで間接参照する（有野 (2017) Ⅰ巻 5.3.1–5.3.2）。これにより再描画要因は 4 段に分かれる（同 5.3.3）。すなわち、①他ウィンドウが退いただけ（アプリは何もせず SurfaceFlinger が再合成）、②表示リストは有効でプロパティだけ変化（位置・alpha・変換。**`draw()` を呼ばず** RenderThread が既存リストを新しいプロパティで**再発行**）、③表示リスト無効（`invalidate()`。その View の `draw()` だけを**再記録**し、親の間接参照はそのまま）、④レイアウト無効（`requestLayout()`。measure/layout からやり直し）である。コストは ①≪②≪③≪④ で、なめらかな UI とは「フレームごとの変化を可能な限り ① と ② に押し込む」設計である。ソフトウェア描画時代は重なった View の移動が下の全 View の `draw()` を誘発したのと対照的で（同 5.3 コラム）、表示リストはシステムが「何が描かれているか」を知っているからこそ差分更新できる。
- **スクロール／fling の実例**: ListView の fling は、指を離した時点の速度から `Choreographer` の `CALLBACK_ANIMATION` に毎 VSYNC の Runnable を登録し、各フレームで子 View を `offsetTopAndBottom` で平行移動する。この移動は View の `mTop/mBottom` 更新に加えて子の **RenderNode のプロパティ（Y オフセット）を書き換えて dirty にするだけ**で、子のサブツリーがどれほど複雑でも measure/layout/`draw()` は走らない（有野 (2017) Ⅰ巻 5.4.3–5.4.4、ケース ②）。新たに入ってくる行だけが inflate/bind と ③ の再記録を払う。この構造は RecyclerView でも同じで、現行 AOSP の `View.offsetTopAndBottom()`（`frameworks/base/core/java/android/view/View.java`）もハードウェア描画時は `mRenderNode.offsetTopAndBottom()` によるプロパティ更新＋`invalidateViewProperty()` の経路をとる。つまり、スクロールの「安さ」の源泉は 2016 年から変わっていない。

### 2. バッファ管理 — gralloc・BufferQueue・フェンス

- **gralloc HAL** がバッファを確保する。フォーマット・用途フラグ（GPU 書込/合成読出/カメラ書込…）から**ベンダ固有の最適レイアウト**（タイル・圧縮 AFBC 等）を選び、dma-buf fd として各プロセスへ渡す。「用途の集合で確保する」API 設計が、後続の全段（GPU・DPU・カメラ ISP）でのゼロコピー共有を可能にしている。
- **なぜ EGL と gralloc が「端末依存の継ぎ目」なのか**: 有野 (2017) Ⅰ巻 6.1–6.3 の整理では、Android の描画は (1) バッファ確保・解放（gralloc）、(2) バッファを描画先として GL に結び付ける EGL（`eglCreateWindowSurface`/`eglMakeCurrent`。Nougat では `ThreadedRenderer` 初期化時に RenderThread 側で行う）、(3) OpenGL ES 実装、(4) 合成ハードウェア（HWC）の四つを移植者が用意する前提で組まれ、BufferQueue・SurfaceFlinger は端末非依存側に置かれる。継ぎ目を「CPU メモリ上の画素配列」という汎用 API にすれば移植は楽だが、GPU メモリで描いた結果をアプリのメモリへ、さらに SurfaceFlinger へ、再び GPU/HWC 側へと 3 回コピーすることになる。継ぎ目を「不透明なハンドル」にし、確保・描画・合成の三者を同じベンダが実装すれば、ハンドル（数バイト）だけがプロセス間を動きコピーは 0 回になる（同 6.3.3、図 6.8/6.9）。可搬性と性能という相反要求を、**端末依存部を最小限に保ちつつ「最適化の効く箇所だけはベンダに開く」** ことで両立させたのがこの構成で、発見事項 1 のゼロコピーはここから来る。後の HIDL/AIDL 化はこの継ぎ目の**位置は変えず、境界の形式（.so の関数表 → 版管理された IPC/stable-C インタフェース）だけを変えた**（技術詳細 5）。
- **Surface の正体**: Surface は BufferQueue の生産者側 (`IGraphicBufferProducer`) を Binder 越しに握るプロキシであり、画素そのものではない（有野 (2017) Ⅰ巻 6.4.1–6.4.2）。Nougat では BufferQueue は SurfaceFlinger プロセス内に作られ、アプリにはそのプロキシが渡されていた。BLAST 以降はこの所在が逆転し、キュー本体 (`BLASTBufferQueue`) がアプリ側に置かれて、取り出したバッファを Transaction で SurfaceFlinger に渡す形になった。
- **BufferQueue の状態機械**: 各バッファは FREE → DEQUEUED（アプリが取得）→ QUEUED（提出）→ ACQUIRED（合成中）→ FREE を巡る。dequeue できなければ生産者はブロックする。これが**背圧**である。ダブルバッファは低レイテンシだが締切ミスに弱く、トリプルバッファは 1 フレームのレイテンシで安定性を買う。
- **同期フェンス**: 「GPU がこのバッファを描き終えた」(acquire fence)・「合成側がこのバッファを読み終えた」(release fence) を dma-fence の fd で運ぶ。CPU はフェンスを**待たずに**ハンドルだけ先に渡せるため、CPU/GPU/DPU の三者が最大限並行に走る。フェンス待ちの計測（どの段で締切を割ったか）が Systrace/Perfetto のフレーム分析の中身である。
- **BLAST**: バッファを `SurfaceControl.Transaction` に添付し、ジオメトリ変更・親子関係・他ウィンドウの変更と**原子的に**コミットする (Android 11 で DeviceConfig `wm_use_blast_adapter` 既定 false の実験機能として入り、Android 12 で既定有効化: `services/core/java/com/android/server/wm/WindowManagerService.java` は android-11.0.0_r1 の lines 1186-1188 で `DeviceConfig.getBoolean(..., WM_USE_BLAST_ADAPTER_FLAG, false)`、android-12.0.0_r1 の lines 1252-1253 で `Settings.Global.getInt(resolver, DEVELOPMENT_USE_BLAST_ADAPTER_VR, 1) == 1`。`addWindow` がこの値で `ADD_FLAG_USE_BLAST` を返し (android-12.0.0_r1 lines 1747-1748)、`ViewRootImpl` が `mUseBLASTAdapter = true` とする (同タグ `core/java/android/view/ViewRootImpl.java` lines 1201-1202))。複数プロセス（アプリ＋IME＋システム UI）にまたがる画面遷移の同期が、トランザクションのマージとして表現できるようになった。

### 3. 合成 — SurfaceFlinger と HWC

- SurfaceFlinger は VSYNC (+sf offset) で起き、各レイヤの**最新の到着済みバッファを latch** し、可視レイヤのリストを HWC HAL に提示する。
- **HWC の割当判断**: ディスプレイコントローラ (DPU) のオーバーレイプレーン数・スケーラ・回転能力の範囲でレイヤを直接割当（`DEVICE` 合成）し、収まらないレイヤ群は SurfaceFlinger が GPU で 1 枚にまとめる（`CLIENT` 合成）。典型画面（壁紙＋アプリ＋ステータスバー＋ナビバー）が DPU だけで合成できるよう、プレーン数はハードウェア設計段階で決められている。これは**電力の議論がハードウェア構成に遡る**好例である。
- **問い合わせ型 API の系譜**: 書籍時点 (HWC1) では SurfaceFlinger が全レイヤを `hwc_layer_1` のリストにして `prepare()` を呼び、HWC が各レイヤに `HWC_OVERLAY`（自分で合成）か `HWC_FRAMEBUFFER`（GPU に回せ）を書き戻す。SurfaceFlinger は後者を GL で `FB_TARGET` という 1 枚にまとめ、それを含む残り全部を `set()` で HWC に渡す。当時 Google は最低 4 レイヤを合成できることを要求し、ステータスバー・ナビバー・アプリ・FB_TARGET の典型構成が HWC だけで閉じるよう設計されていた（有野 (2017) Ⅰ巻 6.4.6–6.4.8）。能力を仕様で固定せず**毎フレーム問い合わせる**ことで、ハード差を吸収しつつ上位は HWC の制約を知らずに Surface を増やせる。同書はこれを CPU キャッシュと DRAM の関係に喩えている。現行の `validateDisplay` → `getChangedCompositionTypes` → `presentDisplay`（DEVICE/CLIENT）も同じ交渉の形式化である。
- **仮想ディスプレイ・画面録画**も同じ機構の別出口（合成結果をバッファへ書く）であり、外部ディスプレイ・Cast は「もう一つの消費者」として自然に増設される。
- **VSYNC モデル**: ハードウェア VSYNC は較正時だけ有効化し、通常はソフトウェアモデル (DispSync) が位相・周期を推定してアプリ/合成用のオフセット付き VSYNC を配る。これにより割り込み削減（電力）と位相制御を両立する。

### 4. 可変リフレッシュレートとフレームペーシング

- SurfaceFlinger はレイヤごとの**希望フレームレート**（アプリの `setFrameRate` ヒント、動画のネイティブ fps、操作中か否か）を集めて表示モードを投票で決める。60fps 動画を 120Hz パネルで再生する類の**整数倍関係の管理**が中心課題。
- **フレームペーシング**: 描画が速く終わったフレームを即提出すると、次の重いフレームとの表示間隔が不均一になりカクつく。Swappy (Android Game SDK) はフレームの**提出タイミングを意図的に遅らせて**表示間隔を等しくする。これは「レイテンシを少し払って等間隔性を買う」方式であり、スループット・レイテンシ・ジッタの三者では**ジッタが知覚品質を支配する**というディスプレイ系の経験則を機構化したものである。
- **ジャンクの分類**: 締切ミスは (a) UI スレッド起因（layout・GC・Binder 同期呼び出しの混入）、(b) RenderThread/GPU 起因、(c) 合成側起因に分けられ、Perfetto のフレームタイムラインが各フレームの実際の律速段を記録する。[`art-runtime-internals.md`](art-runtime-internals.md) の GC ポーズ設計（サブミリ秒）は、この予算の (a) を守るためにある。

### 5. Nougat (書籍) から現在への変遷

有野 (2017) Ⅰ巻は Android 7.0 (2016 年 12 月時点) を記述する。以下は同書の機構記述を基準線に、現行 AOSP（2026 年 9 月時点、Android 16 世代）との差分を並べたもの。版表記は AOSP/公式文書で確認できた範囲に限り、確証の弱いものは注記した。

| 機構 | Nougat (有野 2017) | 現在 | 変化の理由 |
|------|------------------|------|-----------|
| 表示リストの中身 | HWUI 独自の `DisplayListOp` 列（`DrawRectOp` 等）を `DisplayListCanvas` が記録し、RenderThread が `defer()`（変換の統合・隠れたチャンクの削除）→ `replay()` で `OpenGLRenderer` 経由の GL 呼び出しに変換（Ⅰ巻 5.2–5.3） | Android 9 で Skia パイプライン（`libs/hwui/pipeline/skia/SkiaOpenGLPipeline` / `SkiaVulkanPipeline`）が既定になり、記録内容は Skia の描画命令列、実行は Skia の GPU バックエンドに委譲。公開 API も `RecordingCanvas`/`android.graphics.RenderNode`（API 29）に | 独自 GL レンダラの保守（ドライバ差・最適化）を Chrome/Flutter と共用する Skia に寄せ、GL と Vulkan を同じ記録から実行できるようにするため。**記録/実行分離と RenderNode 単位のキャッシュという骨格は不変** |
| GPU API | OpenGL ES のみ（EGL で描画先指定） | Skia は 2024 年以降の新チップセットで Vulkan を使用（Google, Vulkanised 2026）。GLES は ANGLE（GLES 3.2 on Vulkan）で置き換える方針で、一部端末では既に唯一の GLES ドライバ。Android 16 新規端末は Vulkan 1.4 必須 | ベンダ GPU ドライバを Vulkan 一本に絞り、ドライバ品質・更新性を上げるため。移行は端末・版ごとの allow/deny リストで段階的（全面移行の完了時期は未確定） |
| gralloc HAL | `hw_get_module("gralloc")` で `/system/lib/hw` または `/vendor/lib/hw` の `gralloc.*.so` を dlopen し、`hw_module_t`→`gralloc_module_t`（`lock/unlock/registerBuffer`）と `alloc_device_t`（`alloc/free`）の関数表を得る同一プロセス内ライブラリ。確保は SurfaceFlinger が行い、ハンドルを Binder でアプリへ（Ⅰ巻 6.2） | Android 8 の Treble で HIDL `IAllocator`（独立プロセスの binderized サービス）＋ `IMapper`（passthrough）に分割（2.0→3.0→4.0）。その後 AIDL `IAllocator` に移行し、mapper は AIDL にせず **stable-C の `AIMapper`（gralloc 5 / `mapper.<suffix>.so`、allocator-V2 必須）** に（Android 14 で導入: フレームワーク側クライアント `frameworks/native/libs/ui/Gralloc5.cpp` は android-14.0.0_r1 に初出し、android-13.0.0_r1 には無い。新規端末では Android 15 (FCM レベル 202404) で必須になった。VINTF の互換性マトリクスは、Android 14 (`compatibility_matrix.8.xml`) では native と HIDL のどちらかの mapper があればよいとし、Android 15 (`compatibility_matrix.202404.xml`) では HIDL の mapper の項目を除いて native の mapper 5.0 を必須にした） | ベンダ実装とフレームワークを別々に更新可能にする（Treble）。mapper は呼び出し頻度が高くポインタを扱うため IPC 化せず、HIDL 廃止後の受け皿として安定 C ABI を選んだ（stable-c README） |
| 合成 HAL | HWC1: `prepare()` で OVERLAY/FRAMEBUFFER を判定、`set()` で表示。最低 4 レイヤ要求（Ⅰ巻 6.4.6–6.4.8） | HWC2 は実は Android 7.0 で導入済み（書籍は HWC1 の語彙で説明）: `validateDisplay`/`presentDisplay` と DEVICE/CLIENT 合成、明示的 fence。Android 8 で HIDL Composer 2.x、Android 13 で **AIDL `composer3`**（FMQ 廃止・parcelable コマンド、HDR 混在時の SDR レイヤ減光、`DISPLAY_DECORATION` 合成型） | フェンス意味論の明確化と GPU フォールバックの整理、Treble 化、HDR・角丸/切り欠きなど表示ハードの新機能を型付きで表現するため。**「問い合わせてあふれを GPU へ」の交渉構造は不変** |
| バッファの受け渡し | BufferQueue は SurfaceFlinger プロセス内に生成され、アプリは `IGraphicBufferProducer` プロキシを Surface として保持。ジオメトリ（位置・サイズ）は別経路の WindowManager/Transaction（Ⅰ巻 6.4.1–6.4.3） | Android 12 でアプリ既定が **BLAST** (WMS の `mUseBLAST` 既定値が 11 の false から 12 で 1 に; 上記 BLAST 項の行番号参照): キュー本体 (`BLASTBufferQueue`) がアプリ側にあり、dequeue/queue/acquire/release をアプリ内で完結させ、取得したバッファを `SurfaceControl.Transaction` に載せてジオメトリと同時に提出 | バッファとジオメトリが別経路だと、リサイズ・画面遷移で「中身と枠が 1 フレームずれる」。両者を 1 トランザクションで原子的に適用するため（発見事項 6） |
| リフレッシュレート | 60Hz 固定前提。Choreographer が 1/60 秒ごとに呼ぶコールバック内に描き終えれば落ちない（Ⅰ巻 5.4.2） | 可変レート（`Surface.setFrameRate` は Android 11/API 30〜、`View.setRequestedFrameRate` と Adaptive Refresh Rate は Android 15〜）。SurfaceFlinger がレイヤの投票でモードを選び、ARR 対応端末は離散 VSync ステップで表示側をコンテンツに合わせる | 高リフレッシュ化で「常に最大 Hz」は電力的に割に合わず、動画の 24/30fps との整数倍関係も要るため。**予算は定数 16.6ms から変数へ** |
| RenderThread 側アニメーション | fling は UI スレッドの Choreographer コールバックで RenderNode プロパティを更新（Ⅰ巻 5.4）。`RenderNodeAnimator`（Lollipop〜の `ViewPropertyAnimator`/リップル等の一部）と AVD は RenderThread で進行 | 同構造を継承。スクロールは今も UI スレッドで位置計算 → `View.offsetTopAndBottom()` 経由の RenderNode プロパティ更新（ケース ②）であり、RenderThread への完全オフロードではない（`core/java/android/view/View.java` の `offsetTopAndBottom()` が `mTop/mBottom` 更新後に `mRenderNode.offsetTopAndBottom(offset)` を呼ぶ — aosp-mirror 2025-11 スナップショットの lines 20609-20645 で確認） | プロパティ更新だけなら UI スレッドのコストは小さく、入力に追従するスクロールは UI スレッドの入力処理と同期している必要があるため（入力経路は [`input-pipeline.md`](input-pipeline.md)） |
| SurfaceView / TextureView / MediaCodec | SurfaceView は窓に「穴」を開けて直下に別 Surface（別 BufferQueue）を用意し、`MediaCodec.configure(..., surface, ...)` でデコーダが gralloc バッファへ直接出力、HWC がそのまま合成（CPU/GPU 不要、Ⅰ巻 6.6.2）。デコーダは mediaserver から分離途中（6.7 コラム、stagefright 対策） | SurfaceView は独立レイヤ（現在は SurfaceControl の子レイヤ）として HWC オーバーレイ合成が可能で、Android 7.0 以降は親ウィンドウとの位置同期も改善。TextureView はバッファをアプリの GPU 描画のテクスチャとして取り込むため 1 回の GPU 合成と遅延を余分に払う。メディアは media.codec 等の専用・低権限プロセスへ分離済み | 動画は「ハードデコーダ → HWC」直結が電力的に最適で、画面上の自由な変形が要る時だけ TextureView を使う、という選択肢の構造は書籍時点から同じ。変わったのは同期性（BLAST/Transaction）と隔離（Stagefright 以降のメディア権限分割） |

- 表全体を貫く傾向は、**継ぎ目の位置は据え置き、継ぎ目の形式と同期点を強化する**ことである。gralloc/HWC の境界は 2016 年と同じ場所にあるが、`.so` の関数表から版管理された IPC/C ABI へ、暗黙キューから明示トランザクションへ、固定周期から交渉される周期へ、と「暗黙の前提」を一つずつ明示的な契約に置き換えている。
- 書籍自身が追い切れなかったと明言する箇所（Java の `mSurface` がどこで gralloc バッファと結び付くか、`relayoutWindow` → `createSurface` 周辺。Ⅰ巻 6.5.4 コラム）は、現行では「WMS（[`window-manager.md`](window-manager.md)）が `SurfaceControl` を作ってアプリに返し、アプリは自前の `BLASTBufferQueue` からバッファを取る」形に単純化されており、当時の不透明さ自体が BLAST 化の動機の一端だったと読める（この因果は本書の解釈）。

## セキュリティ考察 (Security Considerations)

- **GPU ドライバはアプリから直接届くカーネル攻撃面**: あらゆるアプリが GPU を叩けるため、ベンダ GPU ドライバ（Mali・Adreno）の脆弱性は Binder と並ぶ権限昇格の常連であり、実際に in-the-wild 悪用が繰り返し報告されている。ドライバの ioctl 面の広さ・メモリ管理の複雑さ（マッピングの寿命）が原因で、[`binder-internals.md`](binder-internals.md) の「中核機構＝攻撃面の中核」と同じ構図。GPU ドライバの一部をユーザ空間へ出す・Rust 化するといった構造的対策が進行中の領域である。
- **dma-buf ハンドルは権限である**: gralloc バッファの fd を渡すことは画素メモリへのアクセス権の委譲であり、誤って強力なバッファ（他アプリの画面内容）を渡す経路はサンドボックス迂回になる。SurfaceFlinger が「全画面を見られる」単一プロセスである以上、そこへの到達性 (SELinux) と、スクリーンショット・録画 API の権限ゲートが実質的な壁になる。
- **保護コンテンツ (DRM)**: `SECURE` フラグ付きバッファは GPU/CPU から読めないメモリ（TEE 管理）に置かれ、HWC が直接合成する。スクリーンショットには黒く写る。「OS 自身にも見せない」合成経路が、通常経路と並走している（Widevine L1 の要件）。
- **画面からの情報漏洩はバッファ以外からも**: 描画時間の差・GPU の共有状態を測るサイドチャネル、オーバーレイを使った UI 偽装 (tapjacking) など、パイプラインの性質そのものが攻撃面になる系譜がある。オーバーレイ表示中のタッチ制限・`FLAG_SECURE` はその対症である。

## 結論 (Conclusions)

- Android グラフィックスの本質は「プロセス境界をまたぐゼロコピー生産者・消費者系」であり、制御（Binder トランザクション）と画素（dma-buf ハンドル＋フェンス）の分離、BufferQueue の背圧、VSYNC 位相による予算区画が基本構造をなす。個々の機構は Binder・mmap 共有・cgroup と同じ「共有はハンドルで、同期は明示で」という Android 全体の設計文法の適用である。
- 合成の HWC/GPU 動的振り分けと VSYNC のソフトウェアモデルは、**電力が第一級の設計変数**であることの現れで、プレーン数というハードウェア構成にまで遡って最適化されている。
- BLAST・可変リフレッシュレート・フレームペーシングという近年の変化は、いずれも「非同期パイプラインの見えないずれ（ジオメトリ・表示間隔）を明示的な同期点に変える」方向であり、なめらかさの正体が等間隔性＝ジッタ制御であることを機構として認めた設計である。
- SurfaceFlinger の特権性・GPU ドライバの到達性・バッファハンドルの委譲は、そのままセキュリティ境界の地図である。グラフィックスは性能の話であると同時に、Android の隔離モデルの主要な検証対象である。

### 次の接続先 (Related Topics)

- [`platform-architecture.md`](platform-architecture.md) — SurfaceFlinger・system_server を含む全体像。「1 タップの旅」の描画区間の詳細が本書。
- [`binder-internals.md`](binder-internals.md) — 制御路の実体。fd 翻訳・oneway 背圧・1MB 制限が本書のバッファ受け渡し設計を規定する。
- [`art-runtime-internals.md`](art-runtime-internals.md) — フレーム予算の中で GC が守るべき制約の出どころ。
- [`../../hardware/hw-sw-interface/mmio-dma-interrupts.md`](../../hardware/hw-sw-interface/mmio-dma-interrupts.md) — dma-buf・フェンス・DPU の下にある DMA と割り込みの一般論。
- [`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) — RenderThread・surfaceflinger の優先度 (RT/cgroup) 設計の基盤。
- [`view-system-layout.md`](view-system-layout.md) — 本書の上流。View ツリーの measure/layout/draw と `invalidate()`/`requestLayout()` の区別が、技術詳細 1 の再描画 4 ケースの入口になる。
- [`looper-handler-message-loop.md`](looper-handler-message-loop.md) — Choreographer のフレームコールバックが乗る UI スレッドのメッセージループ（同期バリアによる traversal 優先）。
- [`input-pipeline.md`](input-pipeline.md) — 入力 → ViewRootImpl → View 配信の経路。VSYNC に揃えた入力バッチングとスクロール／fling の起点。
- [`window-manager.md`](window-manager.md) — SurfaceControl の親子・Z 順を決める制御面。

## 参考文献 (References)

**一次資料 (Primary — AOSP)**

- AOSP, *Graphics architecture*（BufferQueue, SurfaceFlinger, HWC, VSYNC, Fence の公式体系）. — [source.android.com/docs/core/graphics](https://source.android.com/docs/core/graphics) — Accessed: 2026-07-03。
- AOSP, *Hardware Composer HAL* ／ *Implementing gralloc*. — [source.android.com/docs/core/graphics/implement-hwc](https://source.android.com/docs/core/graphics/implement-hwc) — Accessed: 2026-07-03。
- Android Developers, *Frame pacing library (Swappy)* ／ *setFrameRate*. — [developer.android.com/games/sdk/frame-pacing](https://developer.android.com/games/sdk/frame-pacing) — Accessed: 2026-07-03。可変レート時代のペーシングの規範。
- Perfetto, *Frame timeline / jank classification*（フレーム律速段の計測法）. — [perfetto.dev/docs](https://perfetto.dev/docs/data-sources/frametimeline) — Accessed: 2026-07-03。
- AOSP, *AIDL for Hardware Composer HAL*（composer3、Android 13〜、FMQ 廃止・SDR 減光・`DISPLAY_DECORATION`）. — [source.android.com/docs/core/graphics/aidl-hwc](https://source.android.com/docs/core/graphics/aidl-hwc) — Accessed: 2026-09-27。
- AOSP, *IMapper stable-c HAL*（gralloc 5、allocator-V2 必須、HIDL 廃止に伴う安定 C API 化）. `hardware/interfaces/graphics/mapper/stable-c/README.md`. — [android.googlesource.com](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/graphics/mapper/stable-c/README.md) — Accessed: 2026-09-27。
- AOSP, HWUI Skia パイプライン実装. `frameworks/base/libs/hwui/pipeline/skia/`（`SkiaOpenGLPipeline.cpp`, `SkiaVulkanPipeline.cpp`）. — [android.googlesource.com](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/libs/hwui/pipeline/skia) — Accessed: 2026-09-27。
- AOSP, *Implement Vulkan*（Android 13 新規端末 Vulkan 1.3、Android 16 新規端末 Vulkan 1.4 必須）. — [source.android.com/docs/core/graphics/implement-vulkan](https://source.android.com/docs/core/graphics/implement-vulkan) — Accessed: 2026-09-27。
- AOSP, *Adaptive refresh rate*（Android 15〜）／ Android Developers, *Optimize frame rate with adaptive refresh rate*（`View.setRequestedFrameRate`）. — [source.android.com/docs/core/graphics/arr](https://source.android.com/docs/core/graphics/arr) ／ [developer.android.com](https://developer.android.com/develop/ui/views/animations/adaptive-refresh-rate) — Accessed: 2026-09-27。
- Android Developers Blog, *High refresh rate rendering on Android*（2020、`Surface.setFrameRate` API 30）. — [android-developers.googleblog.com](https://android-developers.googleblog.com/2020/04/high-refresh-rate-rendering-on-android.html) — Accessed: 2026-09-27。
- Elliott, I. (2026). *Vulkan on Android*. Vulkanised 2026（Google。「Skia uses Vulkan on new chipsets since 2024」、ANGLE が旧 GLES ドライバを置換する方針）. — [vulkan.org](https://vulkan.org/user/pages/09.events/vulkanised-2026/1200-Ian-Elliot-Vulkan%20on%20Android.pptx.pdf) — Accessed: 2026-09-27。一次（ベンダ自身の発表）だが方針・目標の表明であり、実装完了の証拠ではない点に注意。
- Wall, M. (2016). *Hardware Composer 2.0*. Linux Plumbers Conference 2016（HWC2 は Android 7.0 から SurfaceFlinger が使用）. — [linuxplumbersconf.org](https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/4185/original/LPC%20HWC%202.0%20&%20drm_hwcomposer%20.pdf) — Accessed: 2026-09-27。
- AOSP `hardware/interfaces/compatibility_matrices/` — [compatibility_matrix.8.xml (android-14.0.0_r1)](https://android.googlesource.com/platform/hardware/interfaces/+/refs/tags/android-14.0.0_r1/compatibility_matrices/compatibility_matrix.8.xml)・[compatibility_matrix.202404.xml (android-15.0.0_r1)](https://android.googlesource.com/platform/hardware/interfaces/+/refs/tags/android-15.0.0_r1/compatibility_matrices/compatibility_matrix.202404.xml) — Accessed: 2026-11-28。mapper HAL の要件（Android 14 は native か HIDL のどちらか、Android 15 は native mapper 5.0 が必須）。

**二次資料 (Secondary)**

- 有野和真 (2017).『Androidを支える技術〈I〉60fpsを達成するモダンなGUIシステム』技術評論社. ISBN 978-4-7741-8759-4.（Android 7.0 Nougat / android-7.0.0_r6 準拠、2016 年 12 月時点の情報。1.2 GUI システムの三つの時代、第 5 章 DisplayList・RenderThread・ListView の fling、第 6 章 gralloc・EGL・BufferQueue・SurfaceFlinger/HWC・ViewRootImpl・SurfaceView と MediaCodec）。本書の機構的基準線。コード片は著者の簡略化擬似コード。
- 有野和真 (2017).『Androidを支える技術〈II〉真のマルチタスクに挑んだモバイルOSの心臓部』技術評論社. ISBN 978-4-7741-8861-4.（Nougat 準拠、2017 年 2 月時点。プロセス管理・メモリなど本書の周辺文脈）。
- BLASTBufferQueue の所在移動（BufferQueue がアプリ側へ）に関する解説（Android 12）. — [blog.simowce.com/all-about-blastbbq](https://blog.simowce.com/all-about-blastbbq/) — Accessed: 2026-09-27。三次資料寄りの技術ブログであり、一次の裏付けは AOSP `frameworks/native/libs/gui/BLASTBufferQueue.cpp` の実装と、`WindowManagerService.java`／`ViewRootImpl.java` の android-11.0.0_r1 ↔ android-12.0.0_r1 比較 (GitHub `aosp-mirror/platform_frameworks_base`)。

- Google グラフィックスチームのカンファレンス発表（"Drawn out: how Android renders" Google I/O 2018 ほか、RenderThread・BLAST の解説）. — [youtube.com/@AndroidDevelopers](https://www.youtube.com/@AndroidDevelopers) — Accessed: 2026-07-03。
- Google Project Zero / ベンダセキュリティ告示（Mali・Adreno GPU ドライバ脆弱性の in-the-wild 事例分析）. — [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/) — Accessed: 2026-07-03。
