# 電力管理 (Power Management: From cpuidle to Doze & Background Task Scheduling)

## 概要 (Overview)

本リポジトリのモバイル OS 分析([`../android/`](../android/)・[`../ios/`](../ios/))は全編で「バッテリーからの逆算」を語ってきたが、電力そのものを主題にした層がなかった。本ドキュメントはその横串であり、**カーネルの省電力機構 (cpuidle・cpufreq/schedutil・EAS)、システムサスペンドと wakeup source、無線 (RRC) という最大の隠れ消費者、そしてその上に Android (wakelock・Doze・App Standby Buckets・WorkManager) と iOS (assertion・DAS・Low Power Mode) が築いた統治層**を、一つのスタックとして通しで読む。

電力管理の本質は「速くする」の逆問題ではない。**仕事量は同じでも、いつ・どこで・どの粒度でやるかで消費が桁で変わる**。電力管理は、遷移コスト(起きること自体の電力)と定常コスト(起きている間の電力)の会計であり、「まとめて起きて、深く眠る」がすべての層で反復される単一の答えである。ハードウェアの電力ドメイン([`../../hardware/hw-sw-interface/clock-power-reset.md`](../../hardware/hw-sw-interface/clock-power-reset.md))の真上、アプリのライフサイクル契約([`../android/app-components-lifecycle.md`](../android/app-components-lifecycle.md)・[`../ios/app-lifecycle-runningboard.md`](../ios/app-lifecycle-runningboard.md))の真下にあたる層。分析深度は Level 3。

## 方法論 (Methodology)

- 電力を「状態(どの深さで眠っているか)×時間」の積分として扱い、各機構を「眠りを深くする」「眠りを長くする(起床をまとめる)」「起きている時間あたりの仕事を効率化する」のどれに寄与するかで分類する。
- カーネル一般機構(Linux/XNU 共通の概念)→ 無線という支配的負荷 → プラットフォーム統治層(Android/iOS)の順に積み上げ、下の層の物理的制約が上の層の API 設計をどう規定するかを追う。
- 一次資料は Linux カーネルドキュメント・AOSP/Android Developers・Apple Developer の公式文書に置く。消費電力の絶対値は機種依存が激しいため、**桁と比の議論**に留め、特定機種の実測値は扱わない。

## 発見事項 (Findings)

1. **アイドルこそが常態であり、省電力設計の主戦場は「何もしていない時」である。** スマートフォンの CPU は典型的な 1 日の大半をアイドルで過ごす。ゆえに cpuidle(C-state 選択)・サスペンド・起床頻度の削減が、ピーク効率の改善より総電力を支配する。「性能の議論はミリ秒を、電力の議論は時間を見る」。この時間スケールの違いが、この層の設計を性能工学から分離している。
2. **「眠りの深さ」には遷移コストがあり、選択は賭けである。** 深い C-state ほど定常消費は低いが、出入りに時間とエネルギーを払う。次の割り込みまでの時間を**予測して**深さを選ぶ cpuidle governor は、[`../../hardware/cpu/branch-prediction-speculation.md`](../../hardware/cpu/branch-prediction-speculation.md) と同じ「過去から未来を賭ける」構造を持つ。いわば、外れの代償が性能でなくエネルギーで払われる分岐予測である。
3. **DVFS の主導権はガバナーからスケジューラに移った。** 周波数を負荷の移動平均で追いかける旧ガバナー (ondemand) は、スケジューラが既に持つタスクごとの利用率 (PELT) を知らずに二重推定していた。**schedutil** は周波数決定をスケジューラの情報に直結し、**EAS (Energy Aware Scheduling)** はさらにエネルギーモデル(コアごとの消費曲線)を使って「このタスクをどのコアで走らせると系全体が最小消費か」を配置時に解く。big.LITTLE([`process-thread-scheduling.md`](process-thread-scheduling.md))は EAS があって初めて省電力装置になる。
4. **モバイルの Linux は「サスペンドが既定、起きるのが例外」に反転している。** Android は画面消灯後、積極的にシステム全体をサスペンドし (opportunistic suspend)、**wakeup source (wakelock)** を持つ主体がいる間だけ起きている。「誰かが起こしておく理由を登録する」この反転は、メインライン Linux に suspend blockers 論争 (2010) を引き起こした設計であり、実行権を assertion で会計する iOS の RunningBoard([`../ios/app-lifecycle-runningboard.md`](../ios/app-lifecycle-runningboard.md))と**同じ「複式簿記」の電力版**である。
5. **最大の隠れ消費者は CPU ではなく無線である。** セルラーモデムは RRC 状態機械(接続/待機)を持ち、1 パケット送るだけで高電力の接続状態に昇格し、**送信後も数秒〜十数秒の「テール」** をその状態で過ごす。ゆえに「小さな通信をパラパラ送る」ことは、総バイト数が同じでも桁で高くつく。これが、**通信のまとめ打ち (batching)** がプラットフォーム層の中心課題になる物理的根拠であり、プッシュ通知が「各アプリのポーリング」を「OS が一本だけ維持する接続 (FCM/APNs)」に集約した理由でもある。
6. **Android と iOS の統治層は「まとめて・条件付きで・予測して」に収斂した。** Doze(静止検出で深いアイドルに入り、メンテナンスウィンドウで一斉に起きる)と DAS(利用予測・充電・熱で BGTask の時機を決める)、App Standby Buckets(利用頻度でクォータ階層化)と assertion 期限、WorkManager の制約(充電中・Wi-Fi・アイドル)と BGProcessingTask。これらは語彙こそ違うが、「**アプリに時刻を選ばせない。条件を宣言させ、OS が最適時機に束ねる**」という同一の設計文法である。exact alarm が Android で特別権限になったのは、この文法の「例外の値段」を示す。
7. **「充電中の夜」がモバイル計算の遊休資源として制度化された。** ART の背景 dexopt([`../android/art-runtime-internals.md`](../android/art-runtime-internals.md))、iOS の写真解析・OS 更新・最適化充電、両者の ML 学習ジョブといった重い仕事は一斉に「充電・アイドル・(しばしば) Wi-Fi」の窓へ追い出された。バッテリー制約が生んだこの窓は、実質的に**各端末が毎晩持つバッチ処理クラスタ**であり、プラットフォーム設計の前提資源になっている。

## アーキテクチャ (Architecture)

### 電力スタック — 各層の担当

```
 [アプリ]        条件の宣言: WorkManager 制約 / BGTaskScheduler 要求
 ─────────────────────────────────────────────────────
 [統治層]        いつ起こすかの決定と会計:
   Android:      Doze / App Standby Buckets / JobScheduler / wakelock 帰属
   iOS:          RunningBoard assertion / dasd (予測) / Low Power Mode
 ─────────────────────────────────────────────────────
 [カーネル]      眠りの実装:
   cpuidle (C-state 選択) / cpufreq=schedutil (DVFS) / EAS (省エネ配置)
   システムサスペンド + wakeup source / タイマ集約
 ─────────────────────────────────────────────────────
 [SoC/デバイス]  電力ドメイン・クロックゲート・DVFS 段 / モデム RRC /
                 ディスプレイ (可変リフレッシュ) / ストレージ・センサハブ
                 ([`../../hardware/hw-sw-interface/clock-power-reset.md`](../../hardware/hw-sw-interface/clock-power-reset.md))
```

- **下の層ほど物理を、上の層ほど政策を扱う**。カーネルは「眠れる時に最も深く眠る」ことしかできない。**眠れる状況を作る**(起床をまとめ、無駄な常駐を止める)のは統治層の仕事であり、Android/iOS の差はほぼこの層に集中する。
- センサハブ(低電力コプロセッサ)は「歩数計測・持ち上げ検出のために AP を起こさない」ための委譲先で、「小さい仕事は小さいコアへ」の極限形。Doze の静止検出自体もここで走る。

### 起床の経済学 — なぜ「まとめる」が正義か

```
 バラバラの起床:  ▁█▁▁█▁▁█▁▁█▁   起床 4 回 × (遷移コスト + テール)
 まとめた起床:    ▁▁▁████▁▁▁▁▁   起床 1 回、深い眠りが長い
 同じ仕事量でも、遷移とテール (CPU の C-state 出入り・無線の RRC テール) の
 支払い回数が 1/4 になる。タイマ集約・Doze ウィンドウ・プッシュ集約は
 すべてこの図の実装である。
```

## 技術詳細 (Technical Details)

### 1. cpuidle と cpufreq — 眠りの深さと速さの選択

- **cpuidle**: アイドル進入時に governor (menu/teo) が「次の割り込みまでの時間」をタイマ情報と履歴から予測し、損益分岐(遷移コスト < 滞在時間の節約)を超える最深の C-state を選ぶ。予測が浅すぎれば無駄な消費、深すぎれば起床レイテンシと遷移エネルギーの損になる。そのため**タイマの正確さが省電力の前提**になり、だからこそカーネルはタイマ集約(近い期限の deferrable timer をまとめて発火)を行う。
- **cpufreq/schedutil**: PELT による走行中タスクの利用率から直接目標周波数を計算する。「レースして寝る (race-to-idle) か、ゆっくり走るか」は負荷の形に依存し、schedutil はブースト(急な立ち上がりへの即応)と減衰を scheduler のイベント粒度で行える。そのため、UI のバースト的負荷(タッチ→ 数フレームの重い仕事→アイドル)に、移動平均型より本質的に適合する。
- **EAS**: SoC ベンダが提供するエネルギーモデル(各コア種の周波数‐電力曲線)を使い、wakeup 時のコア選択を「性能十分な範囲で系の追加消費が最小の場所」で解く。前面/背面の cgroup 分離([`../android/platform-architecture.md`](../android/platform-architecture.md))と組み合わさり、「背景仕事は小さいコアでゆっくり」が機構として成立する。Apple Silicon の P/E 配置が QoS([`../ios/platform-architecture.md`](../ios/platform-architecture.md))を入力とするのと、入力の語彙(cgroup 対 QoS)だけが違う同型問題である。

### 2. サスペンドと wakeup source — 「起きている理由」の会計

- Android のシステムサスペンドは autosleep ベース: wakeup source が 0 になった時点でサスペンド試行、割り込み(モデム・センサハブ・RTC)で復帰する。**復帰時に「誰のせいで起きたか」を帰属させる**ことが電力デバッグの核心で、Battery Historian 等は wakelock の保持者・起床理由の時系列を可視化する。つまり「電力バグは犯人捜しの問題」である。
- ユーザ空間の wakelock (PowerManager.WakeLock) は「画面消灯後も処理を続けたい」の正規表現だが、保持リークは端末全体の眠りを奪う。これは、**一つのアプリの過失が系全体の定常消費を規定する**という、モバイル特有の外部性である。Android がバージョンを追うごとに wakelock の直接使用を JobScheduler/WorkManager への委譲で置き換えてきたのは、外部性を OS の会計下に置く運動である。
- iOS は対応物を assertion に一元化しており([`../ios/app-lifecycle-runningboard.md`](../ios/app-lifecycle-runningboard.md))、「起きている理由の台帳」がプロセス統治・Jetsam・電力で共用される。Android は wakelock(電力)と oom_adj(メモリ)の台帳が別系である。**台帳の統合度**が両 OS の統治層の見通しの差になっている。

### 3. 無線 — RRC テールとプッシュの経済

- セルラーの RRC 状態機械は、パケット発生で高電力の接続状態に昇格し、無通信が続いてはじめて段階的に降格する。この**テール時間**(ネットワーク事業者設定で数秒〜十数秒)の存在により、通信の電力は「バイト数」ではなく「**セッション回数 × (昇格＋テール)**」でほぼ決まる。小さなハートビートを独自に打つアプリが電池を大きく消耗させる機構的理由である。
- **プッシュの集約**: FCM/APNs は「OS が一本だけ維持する長寿命接続」に全アプリの着信を多重化する。各アプリのポーリング(N 本の接続 × 各テール)を 1 本にまとめる。**接続の共有こそ最大のネットワーク省電力**であり、サードパーティに独自常駐接続を許さない iOS・背景実行を制限する Android の政策は、この物理に根拠を持つ。
- Wi-Fi はセルラーよりバイト単価が桁で安いが、スキャンとビーコン受信の定常コストを持つ。「大きな転送は Wi-Fi/充電待ち」という WorkManager/BGProcessingTask の制約語彙は、この単価差の API 化である。

### 4. Android の統治層 — Doze・バケット・JobScheduler

- **Doze** (Android 6+): 画面消灯・非充電・静止が続くと深いアイドルへ入り、ネットワーク遮断・wakelock 無効化・アラーム/ジョブの延期を行い、**メンテナンスウィンドウ**で一斉に解放して再び眠る。「まとめて起きる」の系全体への適用で、ウィンドウ間隔は時間とともに指数的に伸びる。ポケット内の移動を許す軽量版 (Doze light) が段階を作る。
- **App Standby Buckets** (Android 9+): アプリを利用実績で active / working set / frequent / rare / restricted に分類し、ジョブ・アラーム・FCM 高優先度メッセージのクォータを階層化する。**「使われないアプリは起きる権利ごと減る」**。これは利用予測を権利配分に写像する点で iOS の DAS と同じ発想だが、Android は分類を(挙動が観察可能な)クォータ制で公開し、iOS は時機決定を(観察困難な)予測モデルに委ねる。この**透明性と最適性のトレードオフ**が両者を分ける。
- **JobScheduler/WorkManager**: 「充電中・アイドル・ネットワーク種別」という条件宣言を受け、system_server が束ねて実行する。exact alarm(時刻指定)は Android 12+ で特別権限になり、「時刻を選ぶ自由」は目覚まし・カレンダー級の正当性を要求される。OEM 独自のアグレッシブ kill(ベンダごとの非標準挙動)がこの層の互換性問題として残るのは、統治層が AOSP の外で改変可能である Android の開放性の代償である。

### 5. iOS の統治層 — 凍結・DAS・Low Power Mode

- 既定凍結・委譲・DAS の予測スケジューリングは [`../ios/app-lifecycle-runningboard.md`](../ios/app-lifecycle-runningboard.md) で詳述した。電力の観点で重要なのは、**凍結が「背景 CPU の会計を原理的に不要にする」** ことであり、走れないものは測る必要がない。Android のバケット・クォータの複雑さは「走らせながら絞る」ことの複雑さであり、iOS は問題設定ごと消している。
- **Low Power Mode**: リフレッシュレート低下・背景リフレッシュ停止・メール取得の手動化・視覚効果削減を一括適用するユーザ主導の政策スイッチ。Android の Battery Saver と対応し、アプリには `ProcessInfo.isLowPowerModeEnabled` / `powerSaveMode` として見え、「品質を落として協力する」ことが API 契約に組み込まれている。
- **熱との統合**: thermal state の悪化は周波数上限 (DVFS) だけでなく、統治層の実行権(BGTask の抑制・ProMotion の上限)まで一気通貫で絞る。電力と熱は同じ台帳の両面であり、ここは [`../../hardware/hw-sw-interface/clock-power-reset.md`](../../hardware/hw-sw-interface/clock-power-reset.md) の熱設計がソフトウェア政策に貫通する経路である。

## セキュリティ考察 (Security Considerations)

- **省電力機構は可用性の攻撃面である**: 他アプリを restricted バケットに追い込む・wakelock を握って電池を枯らす(battery drain DoS)・高優先度プッシュの濫用は、いずれも「電力の外部性」を悪用する系譜で、クォータ・帰属会計・プッシュのスロットリングが防壁を兼ねる。**電力の会計はセキュリティの会計でもある**。
- **起床パターンは情報を漏らす**: いつ起きるか・どれだけ通信するかは、暗号化されていてもユーザ行動(起床時刻・アプリ使用)を推定させるサイドチャネルであり、まとめ打ち(バッチング)は省電力と同時にトラフィック解析への緩和にもなる。ただし、目的が違うため保証はない。
- **省電力とセキュリティ更新の緊張**: 「夜間・充電中に更新する」制度は、更新の適時性を充電習慣に依存させる。緊急パッチ(Rapid Security Response・Mainline)はこの窓を待たない別経路を持つ。これは、電力政策がセキュリティの配送計画に例外を要求する構図である。
- **Doze/凍結はマルウェアも眠らせる**: 背景実行の制限は、常駐型の監視・データ収集マルウェアの活動窓も同時に絞る。バックグラウンド統治は電力機構であると同時に、事実上の挙動制限サンドボックスとして機能している([`../android/security-model.md`](../android/security-model.md) の「同意経路」と補完的)。

## 結論 (Conclusions)

- 電力管理は「遷移コストとテールの会計」に還元できる: cpuidle の深さ選択・タイマ集約・RRC テール・Doze のウィンドウ・プッシュの接続共有は、すべて「**起きる回数を減らし、起きたらまとめて働き、深く眠る**」という単一原理の各層での実装である。
- 主導権は一貫して上へ移ってきた: 周波数はガバナーからスケジューラへ (schedutil/EAS)、起床の時刻はアプリから OS へ (JobScheduler/DAS)、常駐の権利は自己申告から会計台帳へ (wakelock 帰属/assertion)。「**アプリに時刻と場所を選ばせない**」ことがモバイル電力設計の到達点である。
- Android と iOS の差は物理層にはなく、統治層の哲学にある: 走らせながらクォータで絞る(透明・複雑・OEM 分裂の余地)か、止めて例外を列挙する(単純・不透明・委譲必須)か。両者とも「充電中の夜」という新しい遊休資源の制度化で合流した。
- この層は本リポジトリの縦の糸を束ねる: ハードの電力ドメインと熱設計が下から制約し、スケジューラと仮想記憶が実装を提供し、ライフサイクル契約とセキュリティ統治が上で政策を執行する。**バッテリーは、モバイルスタックの全層を貫く唯一の設計変数**である。本リポジトリの Android/iOS 分析が繰り返し観察してきた事実の、機構側からの裏付けが本書である。

### 次の接続先 (Related Topics)

- [`process-thread-scheduling.md`](process-thread-scheduling.md) — PELT・cgroup・big.LITTLE。schedutil/EAS の基盤。
- [`../../hardware/hw-sw-interface/clock-power-reset.md`](../../hardware/hw-sw-interface/clock-power-reset.md) — クロックゲート・電力ドメイン・DVFS のハードウェア実装。
- [`../android/app-components-lifecycle.md`](../android/app-components-lifecycle.md) ／ [`../ios/app-lifecycle-runningboard.md`](../ios/app-lifecycle-runningboard.md) — 統治層がアプリに見せる契約の両 OS 版。
- [`../android/graphics-pipeline.md`](../android/graphics-pipeline.md) ／ [`../ios/core-animation-render-server.md`](../ios/core-animation-render-server.md) — ディスプレイという第二の大消費者の制御(可変リフレッシュ)。
- [`../networking/tcp-transport-congestion.md`](../networking/tcp-transport-congestion.md) — 接続維持・ハートビートの輸送層。RRC テールはその電力面の制約。

## 参考文献 (References)

**一次資料 (Primary — カーネル / プラットフォーム公式)**

- Linux Kernel documentation, *CPU Idle Time Management* ／ *Energy Aware Scheduling* ／ *Energy Model*. — [docs.kernel.org/admin-guide/pm/](https://docs.kernel.org/admin-guide/pm/cpuidle.html)、[docs.kernel.org/scheduler/sched-energy.html](https://docs.kernel.org/scheduler/sched-energy.html) — Accessed: 2026-07-03。
- Android Developers, *Optimize for Doze and App Standby* ／ *App Standby Buckets* ／ *Schedule tasks with WorkManager*. — [developer.android.com/training/monitoring-device-state/doze-standby](https://developer.android.com/training/monitoring-device-state/doze-standby) — Accessed: 2026-07-03。
- AOSP, *Power management*（wakeup source・電力会計の公式解説）. — [source.android.com/docs/core/power](https://source.android.com/docs/core/power) — Accessed: 2026-07-03。
- Apple Developer, *Energy Efficiency Guide for iOS Apps*（テール・まとめ打ち・Low Power Mode の公式規範）. — [developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/) — Accessed: 2026-07-03。

**二次資料 (Secondary)**

- LWN.net (2010–). *suspend blockers 論争の一連の報道と技術分析*（opportunistic suspend のメインライン議論）. — [lwn.net](https://lwn.net/Articles/416690/) — Accessed: 2026-07-03。
- 3GPP TS 36.331 ほか RRC 状態機械の標準仕様(セルラーのテール挙動の規範)、および Huang, J. et al. (2012). A Close Examination of Performance and Power Characteristics of 4G LTE Networks. *MobiSys '12*. DOI: 10.1145/2307636.2307658 — RRC テールの電力影響の実測研究。
