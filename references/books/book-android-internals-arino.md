# 有野和真『Androidを支える技術〈I〉〈II〉』読書ノート (Reading Notes)

## 書誌情報 (Bibliographic Information)

- 有野和真 (2017).『Androidを支える技術〈I〉60fpsを達成するモダンなGUIシステム』(WEB+DB PRESS plus シリーズ). 技術評論社. ISBN 978-4-7741-8759-4. サポートページ: [gihyo.jp/book/2017/978-4-7741-8759-4/support](http://gihyo.jp/book/2017/978-4-7741-8759-4/support) — Accessed: 2026-09-27
- 有野和真 (2017).『Androidを支える技術〈II〉真のマルチタスクに挑んだモバイルOSの心臓部』(WEB+DB PRESS plus シリーズ). 技術評論社. ISBN 978-4-7741-8861-4. サポートページ: [gihyo.jp/book/2017/978-4-7741-8861-4/support](http://gihyo.jp/book/2017/978-4-7741-8861-4/support) — Accessed: 2026-09-27

## 位置づけ (Scope & Baseline)

- **対象バージョン**: 本文は Android 7.0 Nougat (AOSP `android-7.0.0_r6`) を基準とし、下書きは 5.0 Lollipop (`android-5.1.1_r6`)。Ⅰ巻は 2016 年 12 月時点、Ⅱ巻は 2017 年 2 月時点の情報。カーネルは `kernel/msm.git` の `android-7.0.0_r0.18` を主に参照。バージョン依存の記述がない箇所は、4.0 ICS 以降なら概ね正しいという前提で書かれている。
- **方法**: Android の各層を網羅するのではなく、一つの問題（Ⅰ巻: 60fps を保つ GUI、Ⅱ巻: Activity とその再生成）について、最上位のアプリ API から HAL・カーネルまで関係する部分だけを縦に貫いて解説する。
- **コードの扱い**: 書籍中のソースコードは実際の AOSP コードを著者が説明用に大きく改変した疑似コードであり、そのままでは動かず、重要な点で実装と異なる場合もあると著者自身が明記している。Daedalus では書籍を**機構の理解と設計意図の二次資料**として扱い、コードや数値は現行 AOSP で照合してから引用する。
- **両巻の関係**: Ⅰ巻は GUI (View・描画・入力・ART)、Ⅱ巻は Activity (インストール・メモリ・プロセス・起動) を軸にし、Looper/Handler・Binder・init.rc・Zygote は両巻の速習節で重複して扱われる。Ⅱ巻 5.7 節「アプリの開始を見直す」が両巻の知識を束ねる山場と著者は位置づけている。

## 章と Daedalus ドキュメントの対応 (Chapter Map)

### Ⅰ巻 — 60fps を達成するモダンな GUI システム

| 章・節 | 内容 | 対応ドキュメント |
|---|---|---|
| 1 章 (1.1–1.2) | GUI への要求、GUI システムの三つの時代 (初期〜GB / Honeycomb〜ICS のハードウェアアクセラレーション / Lollipop〜Nougat の AOT とプロファイル誘導コンパイル) | [`graphics-pipeline.md`](../../software/android/graphics-pipeline.md)、[`art-runtime-internals.md`](../../software/android/art-runtime-internals.md) |
| 1 章 (1.3) | Activity・ActivityThread・プロパティ・Binder・SystemServer・Zygote・init.rc の速習 | [`platform-architecture.md`](../../software/android/platform-architecture.md)、[`boot-init-properties.md`](../../software/android/boot-init-properties.md) |
| 2 章 | Linux Input サブシステム、InputManagerService (InputReader / InputDispatcher)、InputChannel | [`input-pipeline.md`](../../software/android/input-pipeline.md) |
| 3 章 | UI スレッド、Looper・MessageQueue・Handler | [`looper-handler-message-loop.md`](../../software/android/looper-handler-message-loop.md) |
| 4 章 | View ツリー、リソースと AssetManager、LayoutInflater・テーマ、DecorView、measure/layout パス | [`view-system-layout.md`](../../software/android/view-system-layout.md) |
| 5 章 | GPU を使う理由、ThreadedRenderer/RenderThread、RenderNode と DisplayList、ListView の fling | [`graphics-pipeline.md`](../../software/android/graphics-pipeline.md) |
| 6 章 | HAL と gralloc、EGL、BufferQueue、SurfaceFlinger と HWC、ViewRootImpl、SurfaceView と MediaCodec | [`graphics-pipeline.md`](../../software/android/graphics-pipeline.md)、[`view-system-layout.md`](../../software/android/view-system-layout.md) |
| 7 章 | Dalvik バイトコードと dex、レジスタ型 VM、メモリ節約 (mmap・Zygote)、Dalvik JIT → AOT → Nougat のハイブリッド | [`art-runtime-internals.md`](../../software/android/art-runtime-internals.md)、[`memory-lmkd.md`](../../software/android/memory-lmkd.md) |

### Ⅱ巻 — 真のマルチタスクに挑んだモバイル OS の心臓部

| 章・節 | 内容 | 対応ドキュメント |
|---|---|---|
| 1 章 | Activity が実現する世界、インテント入門、全体の見取り図、init.rc 速習 | [`app-components-lifecycle.md`](../../software/android/app-components-lifecycle.md) |
| 2 章 | アプリ＝Linux ユーザーの UID モデル、installd、packages.xml、インテント解決 | [`package-management.md`](../../software/android/package-management.md)、[`security-model.md`](../../software/android/security-model.md) |
| 3 章 | Linux のページ回収と OOM Killer、Low Memory Killer ドライバと lmkd | [`memory-lmkd.md`](../../software/android/memory-lmkd.md) |
| 4 章 | ActivityStack とタスク、launchMode、oom_score_adj、ActivityRecord と Bundle、ライフサイクルの三視点 | [`app-components-lifecycle.md`](../../software/android/app-components-lifecycle.md)、[`memory-lmkd.md`](../../software/android/memory-lmkd.md) |
| 5 章 | app_process と Zygote、ActivityThread (main・attach・H)、5.7 アプリの開始 | [`app-launch-end-to-end.md`](../../software/android/app-launch-end-to-end.md) |
| 6 章 | ブートローダー (LK) と fastboot、init プロセス、プロパティ、init.rc | [`boot-init-properties.md`](../../software/android/boot-init-properties.md) |
| 付録 | onSaveInstanceState の既定実装と View ツリーの状態保存 | [`view-system-layout.md`](../../software/android/view-system-layout.md)、[`app-components-lifecycle.md`](../../software/android/app-components-lifecycle.md) |

## 読み替えガイド — 2016 年から古くなった点 (What Has Changed Since the Book)

書籍の**設計思想と機構の骨格**はほぼ現在も有効である。古くなったのは主に「どのクラス・どの HAL・どのプロセスがそれを担うか」という実装の配置であり、ソースを読むときは次の読み替えが必要になる。根拠と導入バージョンの詳細は各ドキュメントの「Nougat → 現在」の表を参照。

| 書籍の記述 (Nougat) | 現在の読み替え | 詳細 |
|---|---|---|
| ActivityManagerService が Activity/タスクを管理、`ActivityStack`・`ActivityStackSupervisor` | Activity/タスク管理は `com.android.server.wm` の ActivityTaskManagerService (Android 10〜)。`ActivityStack` は `Task` に統合、Supervisor は `ActivityTaskSupervisor` (Android 12〜)。TaskFragment (12L〜) | [`app-components-lifecycle.md`](../../software/android/app-components-lifecycle.md) §6 |
| ActivityThread のクラス `H` が `LAUNCH_ACTIVITY` 等のメッセージで分岐 | ClientTransaction (Android 9〜) に集約され、`H` は `EXECUTE_TRANSACTION` を TransactionExecutor に渡す | [`app-launch-end-to-end.md`](../../software/android/app-launch-end-to-end.md) §7 |
| HWUI 独自の `DisplayListOp` 群 | Skia 記録の表示リスト (SkiaPipeline, Android 9 で既定)。GPU バックエンドは Vulkan の方向 | [`graphics-pipeline.md`](../../software/android/graphics-pipeline.md) §5 |
| gralloc / HWC を `hw_get_module()` で dlopen する HAL、HWC1 の `prepare()`/`set()` | Treble 以後の HIDL を経て AIDL の allocator/composer、mapper は stable-C。HWC2 以降は validate/present | [`graphics-pipeline.md`](../../software/android/graphics-pipeline.md) §5 |
| BufferQueue が SurfaceFlinger 側にあり、アプリはその producer | BLAST: バッファをアプリ側のキューから `SurfaceControl.Transaction` で原子的に提出 | [`graphics-pipeline.md`](../../software/android/graphics-pipeline.md) |
| 入力のウィンドウ情報は WindowManager から InputDispatcher へ | ヒットテスト用のウィンドウ情報は SurfaceFlinger から配信。Reader と Dispatcher の間に palm rejection・ポインタ管理などの段が増えた | [`input-pipeline.md`](../../software/android/input-pipeline.md) §7 |
| カーネル内 lowmemorykiller ドライバが shrinker で kill、lmkd は adj を書くだけ | ドライバは Linux 4.12 で削除。userspace lmkd が PSI で逼迫を検知して kill。cached プロセスは freezer で凍結 | [`memory-lmkd.md`](../../software/android/memory-lmkd.md) §7 |
| installd が dexopt も担当、BackgroundDexOptService が夜間コンパイル | ART Service と artd (Android 14〜)。ART 自体が Mainline モジュールとして Play 経由で更新 (Android 12〜) | [`art-runtime-internals.md`](../../software/android/art-runtime-internals.md) §6、[`package-management.md`](../../software/android/package-management.md) §7 |
| packages.xml はテキスト XML | バイナリ XML (ABX)。パッケージ可視性 (Android 11〜) がインテント解決に加わる | [`package-management.md`](../../software/android/package-management.md) §7 |
| LK ブートローダー、boot / recovery パーティション、ramdisk が常にルート | ベンダ独自ブートローダー＋AVB、A/B・Virtual A/B・dynamic partitions (super)、fastbootd、GKI の boot / vendor_boot / init_boot、2 段階 init | [`boot-init-properties.md`](../../software/android/boot-init-properties.md) §8 |
| View とレイアウト XML が UI の唯一の記述手段 | Jetpack Compose が主流の選択肢に (ComposeView として同じ ViewRootImpl 上で動く)。edge-to-edge の強制 (Android 15〜) | [`view-system-layout.md`](../../software/android/view-system-layout.md) §9 |

**変わっていないもの**: UI スレッドの単一 Looper モデル、「記録は UI スレッド・発行は RenderThread」の二段構え、アプリ＝Linux ユーザーの UID サンドボックス、ActivityRecord 側に残る Bundle、Zygote による fork と COW 共有、init による epoll ループとサービス監視、「プロセスはいつ殺されてもよい」という契約。書籍が最も強調する、なぜそう設計されたかという議論は、ほぼそのまま現在の Android を理解する土台として使える。

## 読み方の提案 (How to Use These Notes)

1. 書籍の章を読み、機構の骨格と設計意図をつかむ。
2. 対応する Daedalus ドキュメントの「Nougat → 現在」の表で、クラス名・HAL・プロセス配置を現行版に読み替える。
3. 実際のソースは cs.android.com で現行の配置 (上表の右列) を辿る。書籍のコードは疑似コードなので、行単位の照合には使わない。
