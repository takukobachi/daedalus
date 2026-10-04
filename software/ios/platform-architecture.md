# iOS プラットフォームアーキテクチャ (iOS Platform Internals)

## 概要 (Overview)

iOS は「XNU カーネル (Mach + BSD + IOKit) の上に、**署名とポリシで全実行コードを統制するユーザランド**を載せた OS」である。プロセスは launchd がオンデマンドで起こし、IPC は Mach ポート（の上の XPC）、メモリ逼迫は圧縮と **Jetsam** による kill で解決し、アプリは背面に回ると**凍結 (suspend)** される。すべての実行ページは署名検証され、サードパーティコードは JIT すら許されない。これらは Android と同じ制約（バッテリー・メモリ・相互不信のアプリ群）への、**「共有と隔離をカーネル機構で作る」(Android) とは対照的な「署名と強制ポリシで統制する」という別解**である。

本ドキュメントは iOS を、(1) カーネル (XNU の Mach/BSD 二重構造)、(2) プロセスモデル (launchd と凍結)、(3) 実行系 (dyld 共有キャッシュ・Objective-C/Swift・ARC)、(4) メモリ管理 (圧縮と Jetsam)、(5) セキュリティモデル (コード署名・サンドボックス・Secure Enclave) の順に読み解き、随所で [`../android/`](../android/) の対応機構と対比する。分析深度は Level 3。

## 方法論 (Methodology)

- Android 分析（[`../android/platform-architecture.md`](../android/platform-architecture.md)）と同じ切り口（プロセス・IPC・実行系・メモリ・セキュリティ）で層を揃え、**同じ制約への異なる解**として対比できるようにする。
- 一次資料は Apple Platform Security ガイド・Apple Developer ドキュメント・公開されている XNU ソース (apple-oss-distributions) に置く。iOS はソース非公開部分が大きいため、**公開ソース/公式文書で裏付く範囲**と、セキュリティ研究者の解析に依る範囲（Jetsam の閾値等）を区別して記す。
- バージョン依存の機構は導入時期を付記する。脱獄 (jailbreak) 手法自体には立ち入らず、公開された脆弱性分析は防御設計の教訓としてのみ扱う。

## 発見事項 (Findings)

1. **XNU は「マイクロカーネルの語彙を持つモノリシックカーネル」である。** タスク・スレッド・ポート・仮想メモリは Mach の抽象、プロセス・ファイル・ソケット・シグナルは BSD 層が提供し、両者は**同一カーネル空間で直結**している。ドライバ層 IOKit (C++) も同居する。メッセージパッシングの語彙を持ちながらアドレス空間分離の代償を払わない、実用主義のハイブリッド。
2. **launchd がプロセス生成の唯一の親であり、名前サービスでもある。** 全デーモン・アプリは launchd が spawn し、サービス名 → Mach ポートの解決 (bootstrap) も launchd が握る。**Zygote (fork による複製) に対する、宣言的なオンデマンド起動**という対照——iOS には Zygote に相当する fork 共有がなく、代わりに **dyld 共有キャッシュ**が「全プロセスで物理 1 部」を実現する。
3. **ARC は「GC を持たない」という決断である。** Objective-C/Swift の参照カウントはコンパイラが挿入する retain/release で、実行時のトレース GC が存在しない。ポーズゼロ・決定的解放・省メモリの代わりに、**循環参照の責任がプログラマ (weak) に移る**。ART の並行 GC（[`../android/art-runtime-internals.md`](../android/art-runtime-internals.md)）と正反対の割り切りで、どちらも「16ms を守る」同じ目的に到達している。
4. **アプリは殺される前に凍結される。** 背面に回ったアプリは（バックグラウンド実行権を持たない限り）スレッドごと停止され、CPU を消費しない。メモリ逼迫時は**圧縮 (memory compressor)** で粘り、それでも足りなければ **Jetsam** が優先度バンドの低い順に kill する。「cached プロセスは走り続ける」Android と「凍結して電力を確実にゼロにする」iOS——lmkd と Jetsam は同じ役割の兄弟だが、**電力への態度**が異なる。
5. **実行できるコードは署名で閉じている。** すべての実行可能ページはコード署名の検証対象で、サードパーティアプリは実行時にコードを生成できない（JIT 不可。JavaScriptCore の JIT は Apple 自身のエンタイトルメント限定）。**「何が走りうるか」を配布時点で確定させる**この一点が、iOS のセキュリティモデルの重心であり、アプリ配布 (App Store) と不可分である。
6. **権限の単位は UID ではなくエンタイトルメントとサンドボックスプロファイルである。** 全アプリは同一ユーザ (`mobile`) で走り、隔離は TrustedBSD MAC 上のサンドボックス（Seatbelt）とコンテナ（アプリごとのデータディレクトリ）が担う。**署名に焼き込まれたエンタイトルメント**が「このバイナリに何を許すか」の宣言で、Android の「カーネル UID 分離が土台」と対照的に、**ポリシエンジンが土台**にある。
7. **ハードウェアの信頼の根は Secure Enclave (SEP) に集約される。** Boot ROM からの署名検証チェーン、ファイル暗号化 (Data Protection) の鍵階層、生体認証の照合、パスコード試行の制限は、独立コアで別 OS (sepOS) を走らせる SEP が握る。Android の TEE/StrongBox と同じ文法だが、iOS は**全端末で同一のハードウェアを前提にできる**分、鍵階層とファイルシステムの統合が深い。

## アーキテクチャ (Architecture)

### スタックの全体像

```
 ┌────────────────── アプリ (署名済みバンドル: Mach-O + リソース) ──────────────────┐
 │  各アプリ = サンドボックスプロファイル + コンテナ + エンタイトルメント             │
 ├─────────────── フレームワーク (Swift/ObjC API: UIKit, SwiftUI, Foundation …) ────┤
 │  実体の多くは XPC で背後のシステムデーモンに委譲 (mediaserverd, locationd …)      │
 ├─────────────── ランタイム & 実行基盤 ───────────────────────────────────────────┤
 │  dyld + 共有キャッシュ │ ObjC ランタイム/Swift │ libdispatch (GCD) │ libSystem    │
 ├─────────────── launchd (PID 1: 起動・監視・bootstrap 名前サービス) ──────────────┤
 ├─────────────── XNU カーネル ─────────────────────────────────────────────────────┤
 │  Mach (タスク/スレッド/ポート/VM/圧縮) + BSD (POSIX/VFS/ネット/サンドボックス MAC) │
 │  + IOKit/DriverKit (ドライバ)  + APFS                                            │
 ├─────────────── セキュリティ協調プロセッサ ───────────────────────────────────────┤
 │  Secure Enclave (sepOS): 鍵階層・生体照合・パスコードガード                        │
 └──────────────────────────────────────────────────────────────────────────────────┘
```

- **system_server に相当する単一巨大プロセスはない**。フレームワーク機能は多数の専用デーモンに分散し、XPC で会話する——「カーネルの外のカーネル」(Android) に対する「デーモンの連邦」。
- **DriverKit** (iOS/macOS 近年) はドライバをカーネル外のユーザ空間プロセスへ出す方向で、IOKit カーネル拡張の攻撃面を縮めている。

### ブートチェーンと更新

```
 Boot ROM (焼き込み・変更不能, 信頼の根)
   └─ 署名検証 → iBoot (ブートローダ)
        └─ 署名検証 → XNU カーネル + カーネルキャッシュ
             ├─ SSV: システムボリュームはハッシュ木ごと署名され読み取り専用マウント
             └─ launchd → デーモン群 → SpringBoard (ホーム画面)
 SEP は並行して自身の sepOS を Boot ROM から検証起動
```

- **署名済みシステムボリューム (SSV, iOS 15+)**: システム領域は APFS スナップショットのハッシュ木がまるごと署名され、実行時改竄はマウント時点で排除される——dm-verity（[`../android/platform-architecture.md`](../android/platform-architecture.md)）のファイルシステム統合版。
- **パーソナライズ更新**: OS 更新イメージは Apple のサーバが**端末固有 ID を含めて署名** (TSS) するため、正規署名済みの旧バージョンでも**他端末用・過去の署名**は使えない——ロールバック防止をサーバ側の署名発行で行う設計（Android の単調カウンタ方式との対照）。

## 技術詳細 (Technical Details)

### 1. XNU — Mach と BSD の分業

- **Mach 層**: タスク（アドレス空間＋ポート名前空間）、スレッド、ポート (IPC)、仮想メモリ（VM object、COW、ページャ）を提供する。カーネル内サブシステム同士も Mach メッセージの語彙で会話できるが、実体は関数呼び出しに最適化される。
- **BSD 層**: プロセス (PID)・ユーザ/グループ・ファイル記述子・VFS・ソケット・シグナル・そして **MAC フレームワーク（サンドボックスのフック点）**。1 つのユーザプロセスは「Mach タスク＋BSD プロセス」の二重の顔を持ち、syscall も Mach トラップと BSD syscall の二系統がある（[`../../hardware/hw-sw-interface/abi-syscalls.md`](../../hardware/hw-sw-interface/abi-syscalls.md) の境界が二重化している、と読める）。
- **スケジューリング**: Mach スレッドを QoS クラス（user-interactive / user-initiated / utility / background）で階層化し、**QoS は GCD のキュー・XPC メッセージ・ロック待ちを通じて伝播**する。Apple Silicon の P/E コア振り分けは QoS を主要な入力とする——[`../android/`](../android/) の cgroup/cpuset (top-app/background) に対応する機構が、カーネルパラメタでなく**API の型 (QoS)** として全フレームワークを貫いている点が iOS 的。優先度逆転は turnstile（優先度継承の一般機構）が緩和する。

### 2. プロセスモデル — launchd・XPC サービス・凍結

- **launchd**: 宣言 (plist) されたデーモン/エージェントを、**最初のメッセージが届いた時点で起動**する (on-demand)。クラッシュしたら次のメッセージで再起動——「サービスは死んでいてよい、ポートが生きていれば」という設計で、Binder の linkToDeath ＋手動再接続（[`../android/binder-internals.md`](../android/binder-internals.md)）に対し、**再起動を名前サービス側が透過的に肩代わり**する。
- **アプリのライフサイクル**: フォアグラウンド → バックグラウンド遷移で短い猶予（状態保存）ののち**全スレッド停止 (suspended)**。以後 CPU 時間はゼロで、メモリ上の姿だけが残る。バックグラウンド実行は「音声再生」「位置情報」等の**宣言された種別**と、BGTaskScheduler による OS 裁量のまとめ実行だけ——Android の「cached でも走りうる＋Doze で絞る」に対し、**既定が凍結で、例外を列挙する**ホワイトリスト構造。
- **プロセス生成**: fork+exec ではなく `posix_spawn` が標準（fork の COW 複製を経ない）。Zygote 型の fork 共有はなく、起動高速化は次項の dyld 共有キャッシュと、直近アプリのメモリ像の温存 (Jetsam が殺していなければ即再開) に依る。

### 3. 実行系 — dyld 共有キャッシュ・ObjC/Swift ランタイム・ARC

- **dyld 共有キャッシュ**: 全システムフレームワークを事前リンクして 1 つの巨大ファイルに焼き、全プロセスが同じ物理ページを mmap 共有する。シンボル解決・再配置はビルド時にほぼ済んでおり (chained fixups)、プロセス起動時のリンクコストと per-process メモリを同時に消す——**Zygote が実行時 fork でやること（ライブラリ・初期化済み状態の物理共有）を、ビルド時成果物でやる**。ART の boot image と最も近い親戚。
- **Objective-C ランタイム**: 全メソッド呼び出しが `objc_msgSend`（セレクタ→実装の動的解決、インラインキャッシュ付き）を通る遅延結合。カテゴリ・swizzling の柔軟性の源泉であり、同時に静的解析の難所。**Swift** は既定で静的ディスパッチ・値型を選び、ObjC 互換が必要な箇所だけ動的機構に落ちる——動的性を「既定」から「選択」に格下げした言語遷移。
- **ARC**: retain/release の挿入と最適化（冗長対の除去）はコンパイラの仕事で、実行時は参照カウント操作のみ。**GC スレッドもポーズも存在しない**。循環は weak/unowned で断つ責任がコード側にあり、autorelease pool がスコープ解放の粒度を制御する。ART との対比:

| 観点 | ART (トレース GC) | iOS (ARC) |
|---|---|---|
| ポーズ | 並行化で最小化（それでも存在） | 原理的にゼロ |
| 解放時期 | 不定（GC 次第） | 決定的（最終参照消滅時） |
| 循環参照 | GC が回収 | プログラマが weak で断つ |
| メモリ余裕 | ヒープに遊びが要る | 使用量 ≒ 生存オブジェクト |
| 実行時コスト | バリア/GC スレッド | 参照カウント原子操作 |

  低 RAM 端末を含む多様なハードに配る Android と、自社ハードのみに配る iOS の**製品戦略の差が、メモリ管理の哲学差**として現れている。

### 4. メモリ管理 — 圧縮と Jetsam

- **swap しない、圧縮して、殺す**: 匿名ページはカーネルの **memory compressor** が RAM 内圧縮する（Android の zram に対応。ただし iOS はディスク swap を一切持たない）。
- **Jetsam**: プロセスは優先度バンド（アイドル < バックグラウンド < フォアグラウンド < システム）に並び、逼迫時は低バンドから kill。加えて**プロセスごとのメモリ上限 (highwater mark)** があり、超過は単独でも kill 対象——「フットプリントの予算制」が明示的で、アプリは `didReceiveMemoryWarning` / `os_proc_available_memory` で自衛する。lmkd の PSI 駆動（システム全体の逼迫度）に対し、Jetsam は**per-process 予算**の色が濃い。
- **凍結との合わせ技**: suspended アプリは圧縮の格好の対象で（触られないページは圧縮したまま）、「凍結 → 圧縮 → それでも足りなければ kill」の三段が、Android の「cached → zram → lmkd kill」に正確に対応する。

### 5. アプリから見た全体像 — 1 タップの旅

```
 ホームでアイコンをタップ (SpringBoard がホーム画面アプリ)
 → SpringBoard が FrontBoard/RunningBoard 経由で起動を要求
 → launchd が posix_spawn: 署名・エンタイトルメント検証 → サンドボックス適用
 → dyld: 共有キャッシュへアタッチ、アプリ本体を chained fixups で束縛
 → UIApplicationMain → 最初のフレームを Core Animation / レンダーサーバへ提出
 (この間、Jetsam がアイドルバンドのアプリを刈ってメモリを空けているかもしれない)
```

- 描画も「アプリが提出し、別プロセス (レンダーサーバ) が合成する」構造で、[`../android/graphics-pipeline.md`](../android/graphics-pipeline.md) の SurfaceFlinger に対応する。Core Animation のレイヤツリー転送という差分はあるが、「制御と画素の分離・VSYNC 同期・別プロセス合成」の骨格は共通である。

## セキュリティ考察 (Security Considerations)

- **コード署名＋実行時強制**: 実行可能ページは署名ハッシュと突き合わせて検証され、`W^X` は例外なし（JIT 許可エンタイトルメントを除く）。「実行されるコードの集合が配布時に閉じる」ため、攻撃は ROP/JOP 等のコード再利用へ追い込まれ、それを **PAC (ポインタ認証, arm64e)** が潰しにいく——ハードウェア（[`../../hardware/cpu-architecture/`](../../hardware/cpu-architecture/)）と配布モデルが噛み合った多層防御。
- **サンドボックスと TCC**: 全アプリ共通のサンドボックスプロファイル (container) がファイル・IPC 到達性・syscall 面を絞り、**TCC** が連絡先・写真・マイク等への実行時同意を管理する。Android の「UID 分離＋SELinux＋実行時権限」と役割対応するが、iOS は**同一 UID 上のポリシ隔離**である点が構造的差異。
- **Secure Enclave と Data Protection**: ファイル鍵は「クラス鍵 ← パスコード絡みの鍵 ← SEP 内 UID 鍵」の階層で包まれ、ロック中はクラス（完全保護等）に応じて鍵自体が消える。パスコード試行制限・生体照合も SEP 内で完結し、**AP (アプリケーションプロセッサ) が侵害されても鍵素材が出ない**ことを目指す——Keystore/StrongBox と同じ文法の、より深い FS 統合版。
- **カーネル自衛の系譜**: KTRR（カーネルテキストの不変化）→ PPL（ページテーブル操作の特権分離）→ **SPTM/TXM (iOS 17+, ページテーブル管理と署名検証をカーネルより高い権限層へ分離)** と、「カーネルが取られても最後の不変条件を守る」層が増え続けている。Mach ポートのライフサイクルバグや IOKit がカーネル攻略の主経路であり続けたことへの構造的応答（詳細は [`mach-ipc-xpc.md`](mach-ipc-xpc.md)）。
- **メッセージ解析の隔離 — BlastDoor**: ゼロクリック攻撃 (iMessage 経由の Pegasus 等) を受け、メッセージの解析を強サンドボックスの専用プロセス (BlastDoor, iOS 14+) に隔離し、共有キャッシュのアドレスランダム化再生成等を導入した。Android の Stagefright 後のメディアスタック分割と**同じ教訓・同じ処方**であり、「パーサは隔離せよ」がプラットフォーム横断の定石であることを示す。
- **Lockdown モード**: 標的型攻撃対象者向けに、攻撃面（メッセージ添付・JIT 付きブラウジング・有線接続）を**機能ごと落とす**運用モード (iOS 16+)。「利便性と攻撃面はトレードオフであり、選ばせる」という設計思想の明示。

## 結論 (Conclusions)

- iOS は Android と同じ制約に対し、「**配布時に閉じる**」ことを軸に組まれた OS である: 署名で実行コードを閉じ、エンタイトルメントで権限を閉じ、凍結でバックグラウンドを閉じ、SEP で鍵を閉じる。Android の「カーネル機構 (UID/COW/cgroup) で開いたまま制御する」文法との対比は、同一問題に対する設計空間の広さを示す最良の教材である。
- 一方で収斂も著しい: ゼロコピー合成・圧縮メモリ＋優先度 kill・ハードウェア鍵隔離・パーサ隔離・ブート署名チェーンは**両者が独立に同じ答え**に達した領域であり、モバイル OS の「正解の形」がここに現れている。
- ARC 対 GC、Zygote 対 dyld 共有キャッシュ、lmkd 対 Jetsam はいずれも「同じ目的・逆の手段」の好例で、自社ハード限定という iOS の前提（ハード多様性を捨てて統合を取る）が技術判断の随所に浸透している。
- カーネル自衛 (PAC/PPL/SPTM) と攻撃面隔離 (BlastDoor/Lockdown) の進化は、「侵害を前提に被害を層で限定する」現代モバイルセキュリティの最前線であり、[`mach-ipc-xpc.md`](mach-ipc-xpc.md) で深掘りする IPC 層はその主戦場である。

### 次の接続先 (Related Topics)

- [`mach-ipc-xpc.md`](mach-ipc-xpc.md) — Mach ポート・XPC の深掘り。Binder 深掘りの iOS 対応物。
- [`../android/platform-architecture.md`](../android/platform-architecture.md) — 同じ切り口の Android 版。全節が対比対象。
- [`../android/art-runtime-internals.md`](../android/art-runtime-internals.md) — ARC の対極にあるトレース GC の実装詳細。
- [`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) — COW・ページ圧縮・OOM の一般論。圧縮＋Jetsam はその iOS 流。
- [`../../hardware/hw-sw-interface/boot-initialization.md`](../../hardware/hw-sw-interface/boot-initialization.md) — Boot ROM からの検証チェーンの一般構造。

## 参考文献 (References)

**一次資料 (Primary — Apple 公式 / 公開ソース)**

- Apple, *Apple Platform Security*（ブートチェーン・SEP・Data Protection・コード署名・サンドボックス・BlastDoor・Lockdown の公式体系）. — [support.apple.com/guide/security](https://support.apple.com/guide/security/welcome/web) — Accessed: 2026-07-03。
- Apple, XNU カーネル公開ソース (apple-oss-distributions/xnu: Mach/BSD 層、memorystatus (Jetsam)、vm 圧縮). — [github.com/apple-oss-distributions/xnu](https://github.com/apple-oss-distributions/xnu) — Accessed: 2026-07-03。
- Apple Developer, *Energy Efficiency Guide for iOS Apps* ／ *Prioritize Work with Quality of Service Classes*. — [developer.apple.com](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/) — Accessed: 2026-07-03。QoS・バックグラウンド実行種別の規範。
- Apple, dyld 公開ソース（共有キャッシュ・chained fixups）. — [github.com/apple-oss-distributions/dyld](https://github.com/apple-oss-distributions/dyld) — Accessed: 2026-07-03。

**二次資料 (Secondary)**

- Levin, J. (2017–). **OS Internals* (Vol. I–III)*. Technologeeks Press. — XNU・launchd・Jetsam・セキュリティ機構の体系的解析（非公式解析の事実上の標準文献）。
- Google Project Zero (2021). *A Look at iMessage in iOS 14*（BlastDoor の第三者分析）. — [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/2021/01/a-look-at-imessage-in-ios-14.html) — Accessed: 2026-07-03。
- Apple WWDC セッション（dyld・GCD/QoS・BGTaskScheduler・メモリフットプリント関連の各回）. — [developer.apple.com/videos](https://developer.apple.com/videos/) — Accessed: 2026-07-03。
