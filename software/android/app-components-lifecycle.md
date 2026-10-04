# コンポーネントモデルとライフサイクル (Activities, Tasks & the Lifecycle Contract over Binder)

## 概要 (Overview)

Activity は「画面を表す UI クラス」として説明されることが多いが、プラットフォーム内部から見ると別の顔を持つ: **(1) system_server が Binder 越しに遠隔操縦する状態機械、(2) manifest に宣言された「システムがアプリのコードを起動できる入り口」、(3) タスク・バックスタックというユーザモデルの構成単位、(4) 「プロセスはいつ殺されてもよい」というメモリ管理戦略をアプリに受け入れさせる契約書**である。

本ドキュメントはこの 4 つの顔を、1 回の Activity 起動を Binder 境界の通過として通しで追うことで統合する。[`binder-internals.md`](binder-internals.md) が機構(ハンドル翻訳・oneway・1MB 制限・スレッドプール)を扱ったのに対し、本書はその**最大の利用者**を扱う——同期/oneway の使い分け・トランザクションサイズの掟・死亡通知が、ライフサイクルという形でアプリ開発者の目に見える現象(ANR・TransactionTooLargeException・プロセス再生成)に変換される様子を確かめる、Binder 理解の応用編である。分析深度は Level 3。

## 方法論 (Methodology)

- 「ホームでのタップ → 最初のフレーム表示」の 1 本の流れを、プロセス境界(Launcher / system_server / アプリ)と Binder インタフェース(IActivityTaskManager / IApplicationThread)の通過点で分節し、各段の担当と通信の性質(同期か oneway か)を特定する。
- システム側(ActivityTaskManagerService とその記録構造)とアプリ側(ActivityThread)の**二重の台帳**を先に固定し、ライフサイクル・タスク・状態保存をその同期問題として説明する。
- 一次資料は AOSP ソース(`ActivityTaskManagerService`, `ActivityThread`, `ClientTransaction`)と Android Developers の挙動仕様に置く。タイムアウト等の数値は AOSP 既定値であり構成で変わりうることを明示する。
- Jetpack (ViewModel/SavedStateHandle) はプラットフォーム契約の**上に築かれた応答**として位置づけ、ライブラリ自体の詳説はしない。

## 発見事項 (Findings)

1. **Activity は自分で状態遷移しない。** onCreate/onPause の呼び出し順・タイミングを決めるのは system_server (ActivityTaskManagerService, ATMS) であり、アプリ側 ActivityThread は Binder 経由で届く **ClientTransaction** の指示を実行するだけの端末である。「ライフサイクルコールバック」の実体は、**別プロセスからの Binder 上りコールをメインスレッドの Handler で直列化したもの**——だからメインスレッドを塞ぐとライフサイクルが止まり、ANR に至る。
2. **通信は非対称に設計されている: アプリ→システムは同期、システム→アプリは oneway。** `startActivity` はアプリが結果を待つ同期 Binder だが、逆向きの IApplicationThread(ライフサイクル指示)は**すべて oneway**である。理由は単純で、**system_server は数百のアプリのどれか一つにもブロックされてはならない**。[`binder-internals.md`](binder-internals.md) の「oneway は同一ノード宛て FIFO」がここで効く——順序は保たれ、しかし完了は保証されないので、システムは**タイムアウト監視**(下記 ANR)で補完する。
3. **タスクとバックスタックはアプリ側に存在しない。** ActivityRecord・Task はすべて system_server 内の台帳であり、「戻る」の意味論・Recents の一覧・マルチウィンドウの割当は、アプリのメモリ状態と独立にシステム側で決まる。**アプリプロセスが死んでもタスクは残る**——「プロセス＝キャッシュ」がユーザ体験として成立するのは、ユーザモデル(タスク)の台帳がアプリの寿命から分離されているからである。
4. **状態は三層に分かれ、それぞれ生存範囲が違う。** ①オブジェクト(ViewModel): 構成変更を生き延びるがプロセス死で消える。②保存状態(onSaveInstanceState の Bundle): **system_server 側に退避される**のでプロセス死を生き延びるが、再起動で消える。③永続化(DB/DataStore): すべてを生き延びる。この三層は「いつ殺されてもよい」契約への応答体系であり、②の Bundle が Binder トランザクションで運ばれることが、**サイズの掟(1MB 共有・TransactionTooLargeException)の出どころ**である。
5. **構成変更の destroy/recreate は「プロセス死のリハーサル」である。** 画面回転で Activity を作り直す設計は一見過激だが、状態とオブジェクトの分離(①と②)を**日常的に強制**する。これに耐えるアプリは自動的にプロセス死からの復元にも耐える——最も稀で再現しづらい障害経路を、最も頻繁な操作で訓練させる設計と読める。
6. **Intent と PendingIntent は名前解決とケイパビリティの二層である。** 明示的 Intent は宛先指名、暗黙的 Intent は PackageManager による能力ベースの解決(名前サービス)。**PendingIntent はそれを「私の身元で後で実行してよい」という委譲可能なトークン**に変えたもので、Binder ハンドル([`binder-internals.md`](binder-internals.md))と同じ「偽造不能・渡せば委譲」の文法を Intent 層で反復している——mutability の明示必須化([`security-model.md`](security-model.md))が重要なのはこのためである。
7. **ANR はライフサイクル版のウォッチドッグである。** 入力ディスパッチ(既定 5 秒)・BroadcastReceiver(前面 10 秒)・Service(前面 20 秒)等のタイムアウトを **system_server が計測**し、超過でダイアログ/kill に至る。oneway で指示を投げるシステムが完了を確かめる唯一の手段が時間であり、**ANR は「同期しない設計」の代金**である。メインスレッドの Binder 同期呼び出し・ロック待ちが典型原因になるのは、ライフサイクル処理と同じスレッドを奪い合うからである。

## アーキテクチャ (Architecture)

### 二重の台帳 — システム側とアプリ側

```
 [system_server]                              [アプリプロセス]
 ActivityTaskManagerService (ATMS)            ActivityThread (メインスレッドの主)
   ├─ Task (バックスタック・Recents 単位)        ├─ ActivityClientRecord (トークン→実体)
   │    └─ ActivityRecord (真実の状態台帳)       ├─ Activity インスタンス
   ├─ WindowManagerService: WindowState         └─ ViewRootImpl (ウィンドウごと)
   └─ ProcessRecord (oom_adj の入力)
        ▲ 同期 Binder (IActivityTaskManager)         ▲ oneway Binder (IApplicationThread)
        │  startActivity, reportIdle …               │  ClientTransaction (ライフサイクル指示)
        └────────────── アプリ ──────────────────────┘
 対応付けは IBinder トークン: ActivityRecord ⇄ ActivityClientRecord を
 偽造不能なハンドルで結び、以後の全指示・報告がこのトークンを鍵に流れる
```

- **真実はシステム側にある**: アプリが見る「自分の状態」は ATMS の台帳の写しであり、両者は Binder トランザクションの列で結果整合になる。二重台帳の同期こそがライフサイクル機構の本体である。
- ActivityRecord の可視性が **oom_adj**([`platform-architecture.md`](platform-architecture.md) の lmkd)の入力になる——「前面か否か」の判定と「殺してよいか」の判定が同じ台帳から出ることが、ライフサイクルとメモリ管理の一体性の実装である。

### 1 回の起動 — Binder 境界の通過点

```
 Launcher ──同期──▶ ATMS.startActivity
                     ├─ Intent 解決 (PackageManager)・権限/BAL 検査
                     ├─ ActivityRecord/Task を台帳に作成
                     ├─ (プロセス無し) Zygote へ fork 依頼 → ActivityThread.main
                     ├─ 前面 Activity へ oneway: PauseActivityItem ──▶ onPause
                     │    (完了報告を最大 ~500ms 待つ: 遅いアプリに引きずられない)
                     └─ 新アプリへ oneway: ClientTransaction
                          [LaunchActivityItem → ResumeActivityItem]
                                └──▶ ActivityThread: Handler 経由でメインスレッドへ
                                      onCreate → onStart → onResume
                                      setContentView → ViewRootImpl が WMS に addWindow
                                      relayout で Surface 取得 → 最初のフレーム描画
                                      ──同期──▶ ATMS: activityIdle (起動完了の報告)
```

- 描画以降は [`graphics-pipeline.md`](graphics-pipeline.md) の世界(BLAST・SurfaceFlinger)に接続する。**Surface の取得すら Binder の往復**であり、「制御は Binder・画素は共有バッファ」の分業がここでも貫かれる。

## 技術詳細 (Technical Details)

### 1. ClientTransaction — ライフサイクル指示の輸送形式

- 指示は個別の RPC ではなく、**コールバック列＋最終目標状態**を束ねた ClientTransaction として届く。受け手の TransactionExecutor が現在状態から目標状態への**経路を補完**する(RESUMED が目標で現在が未作成なら CREATE→START→RESUME を順に実行)。
- この設計は oneway の性質への適応である: 個別指示を投げると**途中状態への割り込み・順序逆転**の管理が地獄になるが、「目標状態」を宣言すれば受け手が冪等に経路を再計算できる。**分散システムの「命令でなく宣言を送れ」**([`../distributed-systems/`](../distributed-systems/))の縮小版が、単一端末内の 2 プロセス間で反復されている。
- 全コールバックはメインスレッドの Looper に載る。**ライフサイクル・入力・描画・アプリ自身のコードが同一キューを奪い合う**——ANR・ジャンクの根本原因はこの一点集中にあり、だからこそ「メインスレッドで Binder 同期呼び出しをするな」が第一の掟になる。

### 2. タスク・バックスタック・起動モード

- **Task** は「ユーザから見た 1 つの作業」で、複数アプリの Activity が同居しうる(メール→添付を開く PDF ビューア)。`launchMode` (standard / singleTop / singleTask / singleInstance)・Intent フラグ (NEW_TASK, CLEAR_TOP)・`taskAffinity` が「新しい Activity をどのタスクのどこに積むか」を決める——**名前解決(Intent)の次に位置する、配置の意味論**である。
- **Recents はタスクの一覧**であり、スワイプで消すのはタスクの削除(プロセスの kill とは別)。逆にプロセスが lmkd に殺されてもタスクとサムネイルは残り、選べば ActivityRecord の台帳から**保存状態付きで再生成**される——ユーザには「ずっと居たアプリ」に見える。
- **予測型バックジェスチャ** (Android 13+) は「戻る」の遷移先を事前に描くため、バックイベントをコールバック登録制に変えた——バックスタックの意味論がシステム側にある(発見事項 3)からこそ、遷移先のプレビューをシステムが合成できる。

### 3. 状態保存 — Bundle の旅と三層の使い分け

```
 onSaveInstanceState(bundle)  … API 27 以前は onStop の前、API 28 (Android 9) 以降は onStop の後に呼ばれる
   └─ Bundle は Parcel 化され ──Binder──▶ system_server の ActivityRecord に退避
 (プロセスが lmkd に殺される)
 ユーザが Recents から復帰:
   └─ 新プロセス fork → ClientTransaction(LaunchActivityItem + 保存済み Bundle)
        └─ onCreate(savedInstanceState) / onRestoreInstanceState で復元
```

- **Bundle が Binder で運ばれる**ことの帰結: (a) 1MB のプロセス共有上限([`binder-internals.md`](binder-internals.md))に食い込むため、**Bitmap や大きなリストを入れた瞬間に TransactionTooLargeException が待つ**。(b) 中身は Parcelable として再構築されるので、**プロセス死後も型が解決できる**必要がある。「保存状態は小さな再構築レシピ、実データは永続層」という使い分けは、輸送機構の制約から導かれる必然である。
- **ViewModel** は ActivityThread 側の(構成変更をまたいで保持される)ストアに住み、Binder を通らない——だから大きくてよいが、プロセス死で消える。**SavedStateHandle** は両者の橋渡しで、三層(発見事項 4)を 1 つの API に畳んだもの。プラットフォーム契約(Bundle/Binder)の制約が、そのまま Jetpack の設計を規定している好例である。

### 4. プロセス重要度との結合 — ライフサイクルは oom_adj の入力である

- ATMS の台帳が更新されるたび、ProcessRecord の重要度(前面/可視/サービス/cached)が再計算され、lmkd の kill 順([`platform-architecture.md`](platform-architecture.md))と cgroup の CPU 配分([`process-thread-model.md`](process-thread-model.md)、[`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md))に反映される。**onStop はコールバックであると同時に「殺してよいリストへの登載通知」** であり、ライフサイクルの各点は開発者向け API とリソース管理の二重の意味を持つ。
- プロセス側の死は Binder の死亡通知(linkToDeath)で ATMS に届き、台帳(ActivityRecord)は保存状態を抱えて生き残る——[`binder-internals.md`](binder-internals.md) の「看取りの機構」が、ユーザ体験(Recents からの完全復帰)に直結する経路がこれである。

### 5. ANR — 時間による同期の回復

- oneway で指示するシステムは、完了を**時間**でしか検証できない。ATMS/InputDispatcher は「入力がウィンドウに 5 秒読まれない」「onPause の報告が来ない」等を監視し、超過で ANR(トレース収集 → ダイアログ → kill)に至る。
- メインスレッドを塞ぐ三大原因——重い処理・**Binder 同期呼び出しの連鎖**(相手のスレッドプール枯渇に巻き込まれる)・ロック競合——はいずれも「ライフサイクルと同じキューを奪う」ことで顕在化する。[`binder-internals.md`](binder-internals.md) の「Binder スレッドは枯渇する」のアプリ側の像が ANR である。キューそのものの機構(同期バリア・非同期メッセージ・epoll 待ち)は [`looper-handler-message-loop.md`](looper-handler-message-loop.md) を参照。

### 6. Nougat (書籍) から現在への構造変化 — 台帳の担当者は変わり、契約は変わらない

有野 (2017) Ⅱ巻 第4章は、Nougat の ActivityManagerService (AMS) と **ActivityStack**(タスクを束ねるスタック)を軸に、ライフサイクルを ActivityStack・Activity インスタンス・AMS という三つの視点から読み解く。同章の見取り図——**真実の状態はシステム側の ActivityRecord にあり、アプリ側の Activity はその一時的な実体にすぎない**——は本書の「二重の台帳」と同じ構図であり、現在もそのまま通用する。変わったのは台帳を持つクラスの配置である。

| 機構 | Nougat (有野 2017 Ⅱ巻 4 章) | 現在 | 変化の理由 |
|---|---|---|---|
| Activity 管理の担当 | AMS (`com.android.server.am`) がプロセス管理と Activity/タスク管理を兼務 | Android 10 で **ActivityTaskManagerService** (`com.android.server.wm`) に分離。AMS はプロセス・サービス・ブロードキャスト・oom_adj を担当 | 肥大化した AMS から Activity/タスク管理を切り出し、WindowManager と同じロック (`WindowManagerGlobalLock`)・同じ階層で扱うため（[`window-manager.md`](window-manager.md)） |
| スタックとタスク | `ActivityStack` がタスク群を保持し、`ActivityStackSupervisor` が全体を統括 | Android 12 (API 31) で `ActivityStack` は **`Task` に統合**(Task が入れ子になれる)、Supervisor は `ActivityTaskSupervisor` に改名。階層の根は `RootWindowContainer` | マルチウィンドウ・分割画面・フリーフォームで「スタック」と「タスク」の区別が意味を失い、WindowContainer の木に一本化された |
| タスク内の分割 | なし(1 タスク = 1 列のバックスタック) | Android 12L で **TaskFragment** 導入。1 タスク内で複数 Activity を横並びに表示 (Activity embedding) | 大画面・折りたたみ端末で「2 画面を 1 タスクに」を表現するため |
| ライフサイクル指示 | AMS が `IApplicationThread` の個別メソッド (`schedulePauseActivity` 等) を oneway で呼び、アプリ側クラス `H` の `handleMessage()` が分岐 | Android 9 で **ClientTransaction** に集約(本書 §1)。`H` は `EXECUTE_TRANSACTION` を受けて TransactionExecutor に渡す | 個別 RPC の順序と途中状態を管理するより、「目標状態の宣言」を冪等に適用するほうが堅牢 |
| Bundle の居場所 | ActivityRecord が保持し、プロセスが kill されても残る(同 4.3) | 同じ。Android 13 で lazy bundle 化(復元時の型混同対策、セキュリティ考察参照) | 契約(プロセス死を生き延びる小さな再構築レシピ)は不変 |

- 読み替えの要点: 書籍の「ActivityStack」は現在の **Task (と親子の Task)**、「AMS の Activity 管理部分」は **ATMS**、「`H` のライフサイクルメッセージ群」は **ClientTransaction の Item 群** (`LaunchActivityItem`・`PauseActivityItem` ほか) に対応する。ソースを読むときは `com.android.server.am` ではなく `com.android.server.wm` を探す。
- 起動の全行程(Zygote fork から最初のフレームまで)をプロセス・スレッド単位で追う議論は [`app-launch-end-to-end.md`](app-launch-end-to-end.md) に分けた。

## セキュリティ考察 (Security Considerations)

- **入り口の管理が第一線である**: exported な Activity への外部 Intent は「他アプリのコードを自分の文脈で走らせる」経路であり、Intent リダイレクト(受け取った Intent 内の Intent を無検証で startActivity する)は confused deputy の典型。`android:exported` 明示必須化(Android 12)と、**Background Activity Launch (BAL) 制限**(Android 10+、背面アプリの不意の前面奪取を原則禁止・以後も送信側/受信側の opt-in 明示へ強化)は、入り口の既定を閉じる系譜である([`security-model.md`](security-model.md) の「暗黙の既定→明示必須化」パターン)。
- **タスクは偽装の攻撃面である**: `taskAffinity` の悪用で他アプリのタスクに自分の Activity を積む **StrandHogg** 型の攻撃は、「ユーザはタスクをアプリと同一視する」というユーザモデルの仮定を突いた。バックスタックの意味論がシステム側にあるからこそ、修正もシステム側(affinity の制限・可視性検査)で行われた——ユーザモデルの台帳はセキュリティ境界でもある。
- **保存状態 Bundle は不信入力である**: 復元経路の Bundle にはカスタム Parcelable が混在でき、直列化/復元の不一致バグは権限昇格(system_server 内での型混同)に使われてきた。Android 13 での **lazy bundle**(触るまで復元しない)化と Parcel 実装の堅牢化はこの系譜への応答で、[`binder-internals.md`](binder-internals.md) の「Parcel は信頼できない入力」がライフサイクル経路にもそのまま適用されることを示す。
- **PendingIntent は身元の又貸しである**: mutable な PendingIntent を広く配ると、受け手が Intent の中身を書き換えて**発行者の権限で**任意コンポーネントに届けられる。FLAG_IMMUTABLE の既定化(Android 12)は、ケイパビリティの委譲を「内容固定」に絞る修正である。
- **画面は最後の攻撃面である**: オーバーレイによるタップジャッキング・フィッシング画面のタスクへの挿入は、Activity のウィンドウとしての顔を突く。`FLAG_SECURE`・オーバーレイ中のタッチ遮断([`graphics-pipeline.md`](graphics-pipeline.md))・「どのアプリが前面か」の API 制限(GET_TASKS の廃止)が対応する防壁である。

## 結論 (Conclusions)

- Activity の本質は UI クラスではなく**契約**である: system_server が Binder で遠隔操縦する状態機械として実装され、「いつ殺されてもよい」メモリ戦略・タスクというユーザモデル・入り口というセキュリティ境界を、1 つのコンポーネント宣言に束ねている。
- 通信設計の非対称(上り同期・下り oneway)がこの領域のほぼ全ての現象を説明する: ClientTransaction の「目標状態」設計は oneway の順序問題への応答であり、ANR は完了を時間でしか検証できないことの代金であり、TransactionTooLargeException は状態退避が Binder を通ることの影である。**Binder の機構的性質([`binder-internals.md`](binder-internals.md))とライフサイクルの体感的現象は、同じものの両面である**。
- 状態の三層(ViewModel / Bundle / 永続層)と構成変更のリハーサル効果は、「プロセス＝キャッシュ」をアプリ開発者が実際に履行可能にするための教育的設計であり、Jetpack はプラットフォーム契約の制約(Binder 輸送・プロセス死)を API に翻訳したものとして読める。
- タスク偽装・Intent リダイレクト・Bundle 復元・PendingIntent と、この層の脆弱性はすべて「台帳と身元の翻訳点」に集中しており、[`security-model.md`](security-model.md) で述べた confused deputy の一般論の最頻出現場がコンポーネントモデルである。Activity を深く理解することは、Android の資源管理・IPC・セキュリティの交点を理解することに等しい。

### 次の接続先 (Related Topics)

- [`binder-internals.md`](binder-internals.md) — 本書の輸送層。oneway 順序・1MB 制限・linkToDeath・スレッド枯渇の機構的裏付け。
- [`platform-architecture.md`](platform-architecture.md) — lmkd・oom_adj・Zygote。ライフサイクルが接続する資源管理の全体像。
- [`graphics-pipeline.md`](graphics-pipeline.md) — activityIdle 後の世界。Surface 取得と最初のフレームの詳細。
- [`security-model.md`](security-model.md) — exported/PendingIntent/BAL の権限モデル上の位置づけ。
- [`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) — 二重台帳の結果整合・「宣言を送る」設計の一般論。
- [`app-launch-end-to-end.md`](app-launch-end-to-end.md) — 本書「1 回の起動」の拡大版。Zygote fork・bindApplication・最初のフレームまでをプロセス/スレッド単位で追う。
- [`looper-handler-message-loop.md`](looper-handler-message-loop.md) — ライフサイクル指示が載るメインスレッドのキューの機構。
- [`memory-lmkd.md`](memory-lmkd.md) — 本書 §4 の先。oom_score_adj の算出と lmkd・freezer による kill/凍結の詳細。
- [`package-management.md`](package-management.md) — 暗黙的 Intent の解決 (resolveIntent) とパッケージ可視性。
- [`system-server-services.md`](system-server-services.md) — ATMS/AMS をホストする system_server の構成と Watchdog。
- [`window-manager.md`](window-manager.md) — ActivityRecord/Task と同じ WindowContainer 木を管理する WMS。

## 参考文献 (References)

**一次資料 (Primary — AOSP / Android 公式)**

- Android Developers, *The activity lifecycle* ／ *Tasks and the back stack* ／ *Saving UI states*. — [developer.android.com/guide/components/activities](https://developer.android.com/guide/components/activities/activity-lifecycle) — Accessed: 2026-07-03。ライフサイクル・タスク・状態保存の規範仕様。
- AOSP ソース: `services/core/java/com/android/server/wm/ActivityTaskManagerService.java`、`core/java/android/app/ActivityThread.java`、`core/java/android/app/servertransaction/`（ClientTransaction/TransactionExecutor 実装）. — [cs.android.com](https://cs.android.com/) — Accessed: 2026-07-03。
- Android Developers, *ANRs* ／ *Restrictions on starting activities from the background*. — [developer.android.com/topic/performance/vitals/anr](https://developer.android.com/topic/performance/vitals/anr) — Accessed: 2026-07-03。タイムアウト種別と BAL 制限の公式解説。
- Android Developers, *PendingIntent*（mutability の規範）. — [developer.android.com/reference/android/app/PendingIntent](https://developer.android.com/reference/android/app/PendingIntent) — Accessed: 2026-07-03。

**二次資料 (Secondary)**

- Promon (2019). *StrandHogg: Android vulnerability*（taskAffinity 悪用によるタスク偽装の技術分析）. — [promon.io/security-news/strandhogg](https://promon.io/security-news/strandhogg) — Accessed: 2026-07-03。
- Android Developers Blog, *Lazy bundles / Parcel 堅牢化と Bundle 系脆弱性への対応*（Android 13 リリース解説）. — [android-developers.googleblog.com](https://android-developers.googleblog.com/) — Accessed: 2026-07-03。
- 有野和真 (2017).『Androidを支える技術〈II〉真のマルチタスクに挑んだモバイルOSの心臓部』技術評論社. ISBN 978-4-7741-8861-4 — 第4章 (ActivityStack・タスク・ActivityRecord と Bundle・ライフサイクルの三視点)、付録 (View ツリーの状態保存)。Android 7.0 Nougat 基準。書籍内コードは著者による簡略化であり、現行 AOSP との対応は [`../../references/books/book-android-internals-arino.md`](../../references/books/book-android-internals-arino.md) を参照。
