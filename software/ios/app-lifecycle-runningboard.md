# アプリライフサイクルと RunningBoard (Scenes, Assertions & Background Execution)

## 概要 (Overview)

[`../android/app-components-lifecycle.md`](../android/app-components-lifecycle.md) は Activity を「Binder で遠隔操縦される状態機械・プロセス死の契約書」として読んだ。本ドキュメントはその iOS 版——**UIApplication/UIScene のライフサイクル、SpringBoard/FrontBoard/RunningBoard によるプロセス統治、実行権の「assertion (主張)」モデル、バックグラウンド実行の委譲と BGTaskScheduler、状態復元、ウォッチドッグ**——を同じ切り口で読む。

対比の焦点は統治のモデルにある。Android はコンポーネントの状態(前面 Activity の有無など)から oom_adj を**導出**するのに対し、iOS は「このプロセスは走ってよい」という**assertion の集合**を RunningBoard が台帳管理し、その合成から実行可否・Jetsam 優先度・メモリ上限を決める。**状態からの導出 対 権利の付与**——同じ「プロセスはいつでも止められる」世界の、対照的な会計方式である。分析深度は Level 3。

## 方法論 (Methodology)

- Android 版と同じ流れ(タップ → 起動 → 前面 → 背面 → 凍結/kill → 復帰)を追い、各遷移で「誰が決め、どの機構で伝わり、アプリには何が見えるか」を特定する。
- 統治側デーモン(SpringBoard/FrontBoard/RunningBoard/dasd)の役割分担を先に固定し、ライフサイクルコールバックをその**帰結**として説明する。
- 一次資料は Apple Developer ドキュメントと WWDC セッションに置く。RunningBoard 等のシステムデーモン内部は Apple が仕様を公開しないため、公式言及の範囲と、公開ログ・セキュリティ研究者の解析に依る範囲を明示する。
- 各節で Android の対応機構([`../android/app-components-lifecycle.md`](../android/app-components-lifecycle.md))と対比する。

## 発見事項 (Findings)

1. **iOS のライフサイクルは「単一の状態機械」であり、Android より語彙が少ない。** not running → foreground (inactive/active) → background → suspended。Activity のような画面単位の再生成はなく、構成変更での destroy/recreate も存在しない(サイズ変化は通知で済ませる)。語彙の少なさは**プロセスと UI の対応が固定的**(1 アプリ 1 プロセス、Extension は別枠)であることの帰結で、Android の「コンポーネント単位の契約」に対し iOS は「プロセス単位の契約」である。
2. **UIScene (iOS 13+) は「タスク」の iOS 版だが、台帳の置き場が逆である。** マルチウィンドウ(iPad)のために UI を scene 単位に分割し、**scene session** の一覧はシステム側に永続する——ここまでは Android のタスクと同じ。しかし Android がバックスタックの意味論を丸ごと system_server に置いたのに対し、iOS の scene 内ナビゲーション(戻る階層)は完全にアプリの私事である。**システムが管理するのは「窓」まで、Android は「画面遷移の歴史」まで**——ユーザモデルへの介入深度の差が台帳の境界に現れている。
3. **RunningBoard は「実行権の複式簿記」である。** プロセスの実行可否・Jetsam 帯域・メモリ上限・CPU 制限は、visibility(FrontBoard が取得)・オーディオ再生(mediaserverd が取得)・バックグラウンドタスク等の **assertion の合成**として決まる。重要なのは**多くの assertion を本人でなく関連デーモンが取得する**ことで、「オーディオを鳴らしているから走ってよい」の判定が、アプリの自己申告ではなく**実際にオーディオを処理しているデーモンの証言**になる——ケイパビリティ的な会計であり、偽装が構造的に難しい。
4. **suspend は SIGSTOP 的な凍結であり、アプリには「時間の欠落」として見える。** 背面遷移から数秒で全スレッドが止まり、CPU 時間は完全にゼロになる。復帰時はスレッドがそのまま再開する——プロセス死と違い**メモリ上の状態は無傷**で、大半の「アプリ切り替え」はこの凍結解除である。Android が cached プロセスを走行可能なまま置いた歴史に対し、iOS は初日から凍結を既定にした。そして Android 11+ の **cached apps freezer** (cgroup freezer) は同じ答えへの合流である——15 年越しの収斂。
5. **バックグラウンド処理は「アプリが続ける」のではなく「OS に委譲する」。** 大型ダウンロードは NSURLSession background セッションとして **nsurlsessiond デーモンに引き渡され**、アプリは殺されてよい(完了時に再起動されて通知を受ける)。VoIP 着信は PushKit、位置追跡は locationd と、**継続性が要る仕事は該当デーモンが肩代わり**する。Android の ForegroundService(アプリ自身が通知を掲げて走り続ける)との最大の構造差で、「誰の CPU 時間として会計されるか」ごと OS 側に移る。
6. **BGTaskScheduler の裏には機械学習スケジューラ (DAS) がいる。** BGAppRefreshTask の実行時機は、ユーザの利用パターン・充電状態・ネットワーク・熱状態を入力とする Duet Activity Scheduler (dasd) が決める。「よく朝 7 時に開くアプリは 6 時台に refresh される」——**利用予測が実行権の割当に直結**する設計で、Android の App Standby Buckets(利用頻度によるクォータ階層)と同じ問題への、より予測駆動な解である。
7. **ウォッチドッグは ANR の兄弟だが、対話の余地なく殺す。** 起動・resume・suspend 遷移でメインスレッドが締切(起動 ~20 秒等)を超えると、システムは即座にプロセスを kill する(クラッシュログの例外コード 0x8badf00d)。Android の ANR がダイアログでユーザに委ねるのに対し、iOS は問答無用——**「遅いアプリはいないのと同じ」**という規範の差がここにも出る。両者とも根は同じで、遠隔操縦されるメインスレッドが応答しない事態を時間で検出するしかない([`../android/app-components-lifecycle.md`](../android/app-components-lifecycle.md) の発見事項 7)。

## アーキテクチャ (Architecture)

### 統治側デーモンの分業

```
 [SpringBoard]   ホーム画面・ロック画面・システムジェスチャの主体 (それ自体が 1 アプリ)
      │ FrontBoard: アプリの「窓」と前面/背面の遷移を管理 (SpringBoard 内のフレームワーク)
      ▼
 [RunningBoard]  プロセス統治の台帳: assertion の集合 → 実行状態・Jetsam 帯域・メモリ上限
      │    ├─ FrontBoard が「可視」assertion を取得/解放
      │    ├─ mediaserverd が「オーディオ再生中」assertion を取得
      │    ├─ dasd が「バックグラウンドタスク実行中」assertion を取得
      │    └─ 期限付き assertion (背面遷移の猶予・beginBackgroundTask)
      ▼
 [launchd]       プロセスの起動・XPC 名前解決 ([`platform-architecture.md`](platform-architecture.md))
 [dasd]          Duet Activity Scheduler: BGTask の実行時機を予測的に決定
 [backboardd]    入力イベント配送・レンダーサーバ ([`core-animation-render-server.md`](core-animation-render-server.md))
```

- Android との対応: ATMS+WMS の一体台帳に対し、iOS は**窓 (FrontBoard)・実行権 (RunningBoard)・起動 (launchd)・時機 (dasd) が別デーモン**に分かれる。「デーモンの連邦」([`platform-architecture.md`](platform-architecture.md))がプロセス統治にも貫かれ、単一障害点を避ける構造になっている。

### 状態遷移とその実装

```
 タップ → FrontBoard がシーン生成を決定 → RunningBoard に可視 assertion
   → launchd が posix_spawn (未起動時) → UIApplicationMain → scene 接続
   → sceneWillEnterForeground → sceneDidBecomeActive        [active]
 ホームへスワイプ:
   → sceneWillResignActive → sceneDidEnterBackground        [background]
   → 猶予 (~数秒, beginBackgroundTask で ~30 秒まで延長可)
   → 可視 assertion 解放・期限切れ → 全スレッド凍結          [suspended]
 メモリ逼迫: Jetsam が suspended の低帯域から kill ([`platform-architecture.md`](platform-architecture.md))
 再タップ: 凍結解除 (プロセス健在) or 再起動＋状態復元 (kill 済み)
```

## 技術詳細 (Technical Details)

### 1. Assertion モデル — 実行権の会計

- assertion は「**誰が・どのプロセスに・どの属性を・いつまで**」の 4 つ組で、RunningBoard が取得・解放・期限切れを台帳管理する。属性の合成が実行状態(runnable/suspended)・Jetsam 優先度帯・メモリ上限 (highwater mark) を一意に決める——[`platform-architecture.md`](platform-architecture.md) で述べた Jetsam の優先度バンドは、この台帳の**出力**である。
- Android の oom_adj 導出と本質は同じ「プロセス重要度の一元台帳」だが、iOS は**権利の存在そのもの**を第三者(デーモン)が証明する点で、自己申告の余地が小さい。ForegroundService の通知悪用(実体のない常駐)のような手口が iOS で成立しにくいのは、この会計方式の帰結である。
- 期限付き assertion (`beginBackgroundTask`) は「もう少しだけ走らせて」の正規表現で、期限超過はウォッチドッグ kill——**猶予は権利であり、権利には期限がある**。

### 2. Scene とマルチウィンドウ

- UIScene は「UI の 1 インスタンス」で、iPad では同一アプリの複数 scene が並ぶ。scene session の台帳(どの窓が存在したか)はシステム側に永続し、プロセスが死んでも窓の存在は残る——Android のタスク/Recents と同じ「ユーザモデルの台帳はアプリより長生きする」原則。
- ただし**scene 内の画面階層はアプリの私有物**である。Android の ActivityRecord スタックに相当する構造をシステムは持たず、復元はアプリが `NSUserActivity` / state restoration API で自前再構築する。介入が浅い分、復元の品質はアプリ実装に依存する——Android が Bundle 退避を**契約として強制**するのと対照的に、iOS の状態復元は**推奨に留まり**、実際「Recents から戻ったら最初の画面」というアプリが iOS には多い。**契約の強制度が UX の一貫性を決める**という比較論点がここにある。

### 3. バックグラウンド実行の委譲パターン

| 要件 | iOS の解 | Android の解 |
|---|---|---|
| 大型ダウンロード | nsurlsessiond へ委譲、完了時に再起動 | DownloadManager / ForegroundService |
| 音声再生 | audio session + mediaserverd の assertion | ForegroundService (mediaPlayback) |
| VoIP 着信 | PushKit (着信時のみ起こされる) | FCM 高優先度メッセージ |
| 位置追跡 | locationd が監視、有意変化で起こす | ForegroundService (location) |
| 定期リフレッシュ | BGAppRefreshTask (dasd が時機決定) | WorkManager + App Standby Buckets |
| 長い処理 | BGProcessingTask (充電中・夜間) | WorkManager (充電・アイドル制約) |

- 共通の設計文法: **「アプリが走り続ける」を廃し、(a) 条件付き委譲、(b) イベントでの起床、(c) 充電中・アイドルへの追い出し**に分解する。ART の背景 dexopt([`../android/art-runtime-internals.md`](../android/art-runtime-internals.md))も iOS の写真解析も「充電中の夜」に走る——**「充電しつつ放置される夜間」がモバイル計算の新しい遊休資源**である点まで両者は収斂した。
- 差分は強制の向き: iOS は既定禁止＋列挙された委譲先、Android は既定許可から出発して制限を積み増した歴史(background execution limits, Android 8)。到達点は近いが、**Android は後方互換の負債を制限の複雑さ**(バケット・例外・OEM 独自 kill)として払い続けている。

### 4. 状態復元とプロセス死からの復帰

- 凍結解除(プロセス健在)が復帰の大半を占めるため、iOS アプリは「復帰＝メモリそのまま」に最適化されがちで、Jetsam kill 後の再起動が品質の穴になりやすい。`NSUserActivity`(現在の活動の宣言)は Handoff・Spotlight・scene 復元に共用され、「状態の外部化」の受け皿になっている。
- Android との構造差: Bundle は**システム側に退避**される(プロセス死と独立に生存)のに対し、iOS の復元アーカイブは**アプリのコンテナに書かれる**——輸送も保管もアプリの領分で、Binder 1MB 制限のような輸送制約はない代わりに、システムは内容に関知しない。「台帳の置き場」の思想差(発見事項 2)が状態保存にも一貫している。

### 5. ウォッチドッグと熱・電力の統治

- ウォッチドッグは起動・遷移の締切に加え、バックグラウンドでの CPU 濫用(凍結されるべきプロセスの走行)も監視する。例外コード(0x8badf00d)付きのクラッシュログはシステムが保全する。
- RunningBoard の台帳は熱制御にも接続し、thermal state の悪化でバックグラウンド実行権が縮む。GPU・ML 等の重い資源の利用可否が「プロセスの重要度台帳」から一元的に絞られる構造は、電力管理([`../operating-systems/power-management.md`](../operating-systems/power-management.md))の実装基盤でもある。

## セキュリティ考察 (Security Considerations)

- **実行権の台帳は攻撃面である**: assertion を偽って取得できれば「殺されないマルウェア」になるため、取得はエンタイトルメントと XPC 呼び出し元検証([`mach-ipc-xpc.md`](mach-ipc-xpc.md) の audit token)で守られる。第三者デーモンが証言する会計(発見事項 3)は、この面でも自己申告より頑健である。
- **バックグラウンド起床は追跡の道具になりうる**: silent push・位置の有意変化・BGTask は「ユーザが使っていない時にコードが走る」経路であり、スロットリング(silent push の頻度制限)と TCC(位置権限の常時/使用中区分)が濫用を絞る。Android の「バックグラウンド位置情報」権限の分離と同じ問題意識である。
- **凍結はフォレンジクスと防御の境界を動かす**: suspended プロセスのメモリは無傷で残るため、AFU 状態の端末からのメモリ抽出リスク([`security-architecture.md`](security-architecture.md))は「走っていないアプリ」にも及ぶ。機微データを扱うアプリが背面遷移でスナップショットを隠し(スクリーンショット保護)、鍵をメモリから消す(`didEnterBackground` での破棄)のは、この境界への応答である。
- **ライフサイクル通知は同一プロセス内の傍受面**: 拡張機能や注入されたライブラリがライフサイクル通知を購読して挙動を変える攻撃面は、コード署名([`security-architecture.md`](security-architecture.md))が実行コードを閉じていることで実質的に制限されている——ライフサイクルの安全性が署名モデルに依存する好例。

## 結論 (Conclusions)

- iOS のライフサイクルは「プロセス単位の契約」であり、その統治は**assertion という権利の複式簿記** (RunningBoard) で実装される。Android の「コンポーネント状態からの導出」と対比すると、同じ資源管理問題に対する**宣言的な権利付与 対 状態からの計算**という設計対が浮かぶ。
- 凍結の既定化・バックグラウンドの委譲・予測的スケジューリング (DAS) は一貫して「アプリの CPU 時間を OS の裁量下に置く」方向で、Android が制限の積み増しで到達した地点に iOS は初期設計で到達していた。一方、Android 11+ の cached apps freezer は両者の収斂を示す決定的な一歩である。
- 台帳の置き場の思想差——窓まで管理する iOS、画面遷移の歴史まで管理する Android——が、状態復元の強制度(推奨 対 契約)と UX の一貫性の差として観察できる。**プラットフォームがどこまでユーザモデルに介入するか**は、ライフサイクル設計の最も本質的な分岐である。
- ウォッチドッグ・熱・電力・セキュリティのすべてが RunningBoard の台帳に集約される構造は、「プロセス統治の一元化」の到達点であり、次書([`../operating-systems/power-management.md`](../operating-systems/power-management.md))で扱う電力管理の土台になる。

### 次の接続先 (Related Topics)

- [`../android/app-components-lifecycle.md`](../android/app-components-lifecycle.md) — 全節の対比対象。Activity 契約と二重台帳。
- [`platform-architecture.md`](platform-architecture.md) — launchd・Jetsam・凍結の全体像での位置づけ。
- [`mach-ipc-xpc.md`](mach-ipc-xpc.md) — assertion 取得・デーモン間協調の輸送層。
- [`core-animation-render-server.md`](core-animation-render-server.md) — 前面にいる間の描画の統治。
- [`../operating-systems/power-management.md`](../operating-systems/power-management.md) — 実行権の統治が仕える最終目的(電力)の横断分析。

## 参考文献 (References)

**一次資料 (Primary — Apple 公式)**

- Apple Developer, *Managing your app's life cycle* ／ *Scenes*. — [developer.apple.com/documentation/uikit/managing-your-app-s-life-cycle](https://developer.apple.com/documentation/uikit/managing-your-app-s-life-cycle) — Accessed: 2026-07-03。ライフサイクル状態と scene の規範仕様。
- Apple Developer, *Using background tasks to update your app* (BGTaskScheduler) ／ *Downloading files in the background* (NSURLSession). — [developer.apple.com/documentation/backgroundtasks](https://developer.apple.com/documentation/backgroundtasks) — Accessed: 2026-07-03。
- Apple WWDC 2019, *Advances in App Background Execution* ／ WWDC 2020, *Why is my app getting killed?*（Jetsam・ウォッチドッグ・背面 kill の公式分類）. — [developer.apple.com/videos](https://developer.apple.com/videos/) — Accessed: 2026-07-03。
- Apple Developer, *Addressing watchdog terminations*（0x8badf00d と締切の公式解説）. — [developer.apple.com/documentation/xcode/addressing-watchdog-terminations](https://developer.apple.com/documentation/xcode/addressing-watchdog-terminations) — Accessed: 2026-07-03。

**二次資料 (Secondary)**

- Levin, J. (2017–). **OS Internals* Vol. I*. Technologeeks Press. — SpringBoard/FrontBoard/BackBoard の分業と RunningBoard 以前のプロセス統治の解析。
- 各種セキュリティ研究・システム解析ブログによる RunningBoard/dasd の挙動解析（公式仕様非公開部分の補完。ログ (`os_log`) ベースの観察が主）。— Accessed: 2026-07-03。
