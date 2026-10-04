# Android プラットフォームアーキテクチャ (Android Platform Internals)

## 概要 (Overview)

Android は「Linux カーネルの上に、**汎用 Linux とはまるで異なるユーザランド**を載せた OS」である。プロセスは `fork`+`exec` ではなく **Zygote** から分岐し、IPC はパイプやソケットではなく **Binder**、メモリ逼迫は swap ではなく**プロセスを殺すこと**で解決し、アプリは「終了」せず OS の裁量でいつでも消される。これらは奇抜さではなく、**バッテリー駆動・メモリ制約・相互に信頼しないアプリ群**という制約からの一貫した逆算である。

本ドキュメントは、Android を汎用 Linux との**差分**として読み解く: (1) プロセスモデル (Zygote と COW)、(2) IPC (Binder のケイパビリティ型 RPC)、(3) 実行系 (ART の AOT/JIT ハイブリッドと GC)、(4) メモリ管理 (lmkd と「プロセス＝キャッシュ」のライフサイクル)、(5) セキュリティモデル (UID サンドボックス・SELinux・Verified Boot)。本リポジトリの OS 層ドキュメント（[`../operating-systems/`](../operating-systems/)）の機構が、実プロダクトでどう組み合わされるかの**応用事例集**としても読める。分析深度は Level 3。

## 方法論 (Methodology)

- スタックを層（カーネル拡張 → ネイティブ層 → ランタイム → フレームワーク → アプリ）に分解し、各層で「汎用 Linux ならこうする／Android はこうする／なぜか」を対比する。
- 一次資料は AOSP 公式ドキュメント (source.android.com) と Android Developers の挙動仕様に置き、バージョン依存の機能は導入バージョンを付記する。
- 既存ドキュメントの概念（COW・ページキャッシュ・OOM・cgroup・セキュアブート）への対応を明示し、重複を避けて差分に集中する。
- 特定端末・ベンダ実装の差異には立ち入らず、AOSP の標準構成を対象とする。

## 発見事項 (Findings)

1. **Android のサンドボックス単位は「Linux ユーザ」である。** アプリごとに一意の UID を割り当て、カーネルの由緒ある UID 分離をアプリ間の壁として使う。上に SELinux (MAC)・seccomp・権限モデルを重ねる多層構造で、**最も古い機構が最も土台**にある。
2. **アプリの起動が速いのは Zygote の COW 分岐だから。** 起動時に VM・コアライブラリ・共有リソースをプリロードした Zygote プロセスを常駐させ、全アプリを**そこから `fork`** する。ページは copy-on-write で共有され、起動コストとメモリの両方を節約する——[`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) の COW の、プラットフォーム設計レベルでの活用。
3. **Binder は「ファイルディスクリプタ的なケイパビリティ」を渡せる RPC である。** Binder ハンドルは偽造できない参照＝権限そのもので、カーネルドライバが送信元 UID/PID を保証する。「誰から来たか」をカーネルが証明する IPC が、権限チェックとサンドボックスの通用門になっている。
4. **ART は「測ってからコンパイルする」。** インストール時全 AOT（初期）→ インタープリタ＋JIT＋**プロファイル誘導の充電時 AOT**（Android 7+）へと進化した。ホットパスだけを最適化する判断をランタイムの実測に委ねる設計で、JIT/AOT の古典的トレードオフに対する運用込みの答えである。
5. **swap の代わりにプロセスを殺す。** ディスク swap は行わず（zram による圧縮 RAM swap は併用）、メモリ逼迫時は **lmkd** が重要度の低いプロセスから SIGKILL する。アプリのライフサイクル（onCreate/onStop…）が「いつ殺されてもよい」前提で設計されているため、**プロセスはキャッシュ**として扱える——`onDestroy` は保証されない、はこの帰結。
6. **Project Treble はカーネル/ベンダ層と OS 更新を分離した。** HAL を安定した Binder インタフェース (HIDL→AIDL) の背後に隔離し、SoC ベンダのコードを差し替えずに Android 本体を更新可能にした。組込みの宿痾「ベンダ依存で更新が止まる」への構造的な対処。
7. **セキュリティは「起動の検証」から「実行時の隔離」まで一本の鎖である。** Verified Boot (dm-verity) がブート段とシステムパーティションの改竄を拒み、その上で UID+SELinux+権限が実行時を隔離し、鍵は Keystore/StrongBox がハード側に隔離する。車載の HSM・セキュアブート（[`../../engineering/automotive/cybersecurity-iso21434.md`](../../engineering/automotive/cybersecurity-iso21434.md)）と同じ設計文法である。

## アーキテクチャ (Architecture)

### スタックの全体像

```
 ┌───────────────────────── アプリ (APK: DEX + リソース) ─────────────────────────┐
 │  各アプリ = 固有 UID + 固有プロセス (ART インスタンス) + SELinux ドメイン        │
 ├──────────────────── Android フレームワーク (Java/Kotlin API) ──────────────────┤
 │  ActivityManager / WindowManager / PackageManager … = system_server 内のサービス │
 ├─────────────── ランタイム & ネイティブ層 ──────────────────────────────────────┤
 │  ART (DEX 実行: AOT/JIT/GC) │ Bionic libc │ ネイティブデーモン (init, lmkd,     │
 │                             │             │  SurfaceFlinger, netd, vold …)      │
 ├─────────────── HAL (Treble 境界: 安定した AIDL/HIDL インタフェース) ────────────┤
 │  カメラ / オーディオ / センサ / 無線 … ベンダ実装 (vendor パーティション)         │
 ├─────────────── Linux カーネル + Android 拡張 ──────────────────────────────────┤
 │  Binder ドライバ │ ashmem/memfd │ ION/DMA-BUF │ wakelock │ cgroup (cpuset) …    │
 └─────────────────────────────────────────────────────────────────────────────┘
```

- **system_server** がフレームワークサービスの束を一手に担う巨大プロセスで（起動フェーズ・SystemServiceManager・Watchdog は [`system-server-services.md`](system-server-services.md)）、全アプリと Binder で会話する。Android の「カーネルの外のカーネル」。
- パーティションは役割で分離される: `boot`(カーネル)・`system`(AOSP)・`vendor`(SoC/ベンダ)・`userdata`。Treble 以降、`system` と `vendor` の境界が安定 ABI になった。

### プロセス生成 — Zygote モデル

```
 init (PID 1)
   └─ zygote 起動: ART 初期化、コアクラス・共有リソースをプリロード
        │  以後ずっと常駐し、ソケットで起動要求を待つ
        ├─ fork ──▶ system_server (フレームワークサービス群)
        ├─ fork ──▶ アプリ A (UID 10101)   ← プリロード済みページを COW 共有
        ├─ fork ──▶ アプリ B (UID 10102)   ← 同上
        └─ …
 fork 直後に: UID/GID 設定 → SELinux ドメイン遷移 → seccomp フィルタ適用 → アプリコードへ
```

- 汎用 Linux の `fork`+`exec` はアドレス空間を捨てて作り直すが、Zygote 分岐は**プリロード済みのアドレス空間をそのまま引き継ぐ**。数百ミリ秒級の VM 初期化が省け、共有ページ（フレームワーククラス・リソース）は全アプリで物理 1 部になる。
- 代償と対策: fork 元が単一なのでアドレス空間レイアウト (ASLR) が全アプリで似る——Zygote 起動ごとの乱数化に加え、要求時に事前 fork したプールから払い出す USAP など、分岐点の工夫が続く。**「速さのための共有」と「分離のための多様性」の緊張**がここにも現れる。

### Binder — ケイパビリティ型 IPC

```
 クライアント                     カーネル (binder ドライバ)              サーバ
 proxy.transact(code, data) ──▶  送信元 UID/PID を刻印し、宛先の        ──▶ onTransact(...)
                                 スレッドプールへ転送 (コピー1回)
        ◀────────────────────────  応答 ────────────────────────────────
 ハンドル = プロセスローカルな番号 ←→ ドライバ内のオブジェクト参照表で解決
 (偽造不能。渡されなければ呼べない = capability)
```

- **名前解決は ServiceManager**（ハンドル 0 の well-known オブジェクト）が担い、システムサービスはここに登録される。
- **送信元の真正性**: サーバは `getCallingUid()` で「カーネルが保証する呼び出し元」を取得して権限判定する。ソケットのような自己申告ではない——Android の権限モデルはこの一点に多くを負っている。
- 特性: 同期呼び出しが基本（`oneway` で非同期化）、トランザクションバッファは小さい（~1MB、大きなデータは fd/ashmem を**ハンドルとして**渡す）、**死亡通知 (linkToDeath)** で相手プロセスの消滅を購読できる——「相手がいつでも殺される」世界の必需品。
- [`../../hardware/hw-sw-interface/abi-syscalls.md`](../../hardware/hw-sw-interface/abi-syscalls.md) の視点では、Binder は「syscall 境界の上にもう一つ張られた、オブジェクト指向の特権境界」である。

## 技術詳細 (Technical Details)

### 1. ART — DEX・AOT/JIT ハイブリッド・GC

- **DEX**: Java/Kotlin は JVM バイトコードではなく、レジスタベースの DEX に変換されて APK に載る。
- **コンパイル戦略の変遷**: Dalvik (JIT のみ) → ART 初期 (インストール時に全 AOT: インストールが遅くストレージを食う) → **Android 7+ のハイブリッド**——初回はインタープリタ＋JIT で走らせ、ホットメソッドのプロファイルを記録し、**充電中・アイドル時に profile-guided AOT** (`speed-profile`) を行う。よく使う画面は速く、触らないコードはコンパイルすらされない。アプリ配布側はベースラインプロファイル (Baseline Profiles) で「初回からホットパスを AOT 済み」にできる。
- **GC**: 並行コピー GC (Concurrent Copying, Android 8+)。リージョンベースのヒープを、読み取りバリアを使って**アプリを止めずに**コンパクションする。Android 10+ で世代別化され、短命オブジェクト優先の回収になった。その後 Android 13 で userfaultfd ベースの並行マークコンパクト (CMC) が導入され、ART の Mainline 更新により対応カーネルの端末では既定 GC になった（[`art-runtime-internals.md`](art-runtime-internals.md) 参照）。UI スレッドを 16ms のフレーム予算内に保つことが GC 設計の支配的要求で、「スループットよりポーズ時間」という組込み寄りの選択である。

### 2. メモリ管理 — lmkd と「プロセス＝キャッシュ」

- **swap しない、殺す**: ディスク swap はフラッシュの寿命・レイテンシ・電力で割に合わない。代わりに (a) **zram**（圧縮 RAM swap）で匿名ページを RAM 内に圧縮退避し、(b) それでも足りなければ **lmkd** がプロセスを殺す。
- **重要度は ActivityManager が決める**: 前面 (foreground) > 可視 > サービス > **cached**（画面から消えたアプリ）の順で `oom_score_adj` を動的に更新し、lmkd は**低重要度から**殺す。cached プロセスは「次の起動を速くするためのキャッシュ」であり、殺されることが正常系——[`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) の OOM killer が「破綻処理」なのに対し、Android では**平常運転の仕組み**に昇格している。
- **逼迫の検知は PSI**: 現代の lmkd はカーネルの PSI (Pressure Stall Information) を監視して「回収で停滞し始めた」を早期検知して動く（旧カーネル lowmemorykiller の置き換え）。
- **アプリ側の契約**: ライフサイクルコールバック（onStop で状態保存、onDestroy は当てにしない）は、この「いつ殺されてもよい」メモリ管理と表裏一体の API 設計である。
- **CPU 側の対応物**: cgroup/cpuset で `top-app`（前面アプリ）に広いコア集合と高い優先度を与え、`background` を絞る。big.LITTLE の非対称コアへの配置もここで決まる——[`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) の cgroup 階層の実プロダクト適用（プロセス/スレッドと cgroup・task profile の対応は [`process-thread-model.md`](process-thread-model.md)）。

### 3. HAL と Project Treble — 更新可能性の構造改革

- Treble (Android 8) 以前、ベンダ実装は Android 本体と密結合で、OS 更新にはベンダの再作業が必須だった——更新が止まる構造的原因。
- Treble は **HAL を Binder ベースの安定インタフェース (HIDL、現在は AIDL) の背後に置き**、`system`（Google/AOSP 管理）と `vendor`（SoC ベンダ管理）をパーティションごと分離した。VTS (Vendor Test Suite) が境界の互換性を機械検証する。
- さらに **APEX** (Android 10+) がランタイムや個別モジュールを Play 経由で更新可能にし（Project Mainline）、「OS 全体の更新」を待たずにセキュリティ修正を配れるようにした。**更新可能性はアーキテクチャの問題である**——車載 SDV が同じ課題（[`../../engineering/automotive/in-vehicle-networks.md`](../../engineering/automotive/in-vehicle-networks.md) のゾーン化・中央集約）に向かっているのは偶然ではない。

### 4. 起動の検証 — Verified Boot

- ブート ROM → ブートローダ → `boot` パーティション（カーネル）→ `system` と、各段が次段の署名を検証する信頼の連鎖（[`../../hardware/hw-sw-interface/boot-initialization.md`](../../hardware/hw-sw-interface/boot-initialization.md) のセキュアブートの実装例）。
- システムパーティションは **dm-verity** でブロック単位のハッシュ木により**読むたびに**検証される——起動時 1 回の検証ではなく、実行中の改竄も検出する。
- **ロールバック保護**: 既知の脆弱性を持つ旧バージョンへの差し戻しを、単調カウンタで拒否する。鍵とカウンタの根はハードウェア（TEE/専用チップ）に置かれる。

### 5. アプリから見た全体像 — 1 タップの旅

```
 ホームでアイコンをタップ
 → Launcher が Binder で ActivityManager (system_server) に startActivity を要求
 → ActivityManager: 権限・対象解決 → プロセスが無ければ Zygote へ fork 要求
 → Zygote fork → UID/SELinux/seccomp 適用 → ART がプロファイル済み OAT を mmap
 → アプリの onCreate → WindowManager/SurfaceFlinger 経由で最初のフレーム描画
 (この間、lmkd が cached プロセスを殺して RAM を空けているかもしれない)
```

一つの UI 操作の裏で、本ドキュメントの全要素（Binder・Zygote・ART・lmkd・SELinux）が協調する。層ごとの理解を「1 本の流れ」に束ねて確認できる好例である。

## セキュリティ考察 (Security Considerations)

- **多層サンドボックス**: ①カーネル UID 分離（アプリ間はファイルシステム・プロセスレベルで他人）→ ② **SELinux** が「root でもできないこと」を規定（`untrusted_app` ドメイン、Android 5 以降全面 enforcing）→ ③ seccomp が使える syscall を制限 → ④ 権限モデル（install 時/実行時/署名権限、Android 6+ で実行時付与）がフレームワーク API を制御。**1 層の突破が即全損にならない**構成で、車載の多層防御（[`../../engineering/automotive/cybersecurity-iso21434.md`](../../engineering/automotive/cybersecurity-iso21434.md)）と同型。
- **メディアスタックの教訓 (Stagefright, 2015)**: MMS で届く動画の解析だけで RCE に至る脆弱性群は、「巨大な C++ パーサが高権限プロセスに同居する」ことの危険を実証した。以後 Android はメディア処理を**権限の異なる複数プロセスへ分割**し、コーデックを制約の強いサンドボックスに隔離した——攻撃面の分割という古典原則の大規模適用例。
- **鍵の隔離**: Keystore は鍵操作を TEE (TrustZone) 内で行い、鍵素材を Android 側に出さない。**StrongBox** (Android 9+) はさらに独立したセキュアエレメントへ隔離する。「OS が侵害されても鍵は出ない」を目指す点で、車載 HSM・航空宇宙の耐タンパ設計と同じ文法。
- **Binder はカーネル攻撃面でもある**: 全プロセスが直接叩けるカーネルドライバであるため、Binder 実装の UAF・競合は権限昇格の常連クラスである（[`../concurrency/lock-free-data-structures.md`](../concurrency/lock-free-data-structures.md) の「カーネル内競合＝昇格」の系譜）。到達可能性の高さゆえ、ここの監査・fuzzing は継続的に行われている。
- **アプリ間の残る経路**: UID 分離があっても、エクスポートされたコンポーネント・暗黙の Intent・ContentProvider の設定ミスは、アプリ自身が開ける穴として残る。プラットフォームの壁とアプリの設計責任は別物である。

## 結論 (Conclusions)

- Android は「バッテリー・メモリ・相互不信のアプリ」という制約から、プロセス生成 (Zygote/COW)・IPC (Binder)・実行 (ART プロファイル誘導)・メモリ (lmkd/プロセス＝キャッシュ) のすべてを汎用 Linux から作り替えた OS である。個々の判断は既存ドキュメントの機構（COW・OOM・cgroup・セキュアブート）の応用として読める。
- 中心にあるのは Binder で、IPC・名前解決・権限判定・死亡通知が一つの機構に集約され、UID サンドボックスと組み合わさって「カーネルが身元を保証する社会」を作っている。
- lmkd とライフサイクル API の対は「メモリ管理を API 契約に昇格させた」設計であり、OS とアプリの責務分界の好例。
- Treble/APEX は「更新できないこと」をアーキテクチャで解いた事例であり、Verified Boot〜StrongBox の鎖は組込みセキュリティの設計文法を大衆機に実装した到達点である。車載 (Android Automotive OS) への展開も含め、本リポジトリの hardware/software 両スタックの交点として今後の深掘りに向く領域である。

### 次の接続先 (Related Topics)

- [`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) — COW・OOM・PSI・zram の一般論。本書はその Android 流の運用。
- [`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) — cgroup/cpuset・優先度。top-app/background 分離の基盤。
- [`../../hardware/hw-sw-interface/boot-initialization.md`](../../hardware/hw-sw-interface/boot-initialization.md) — Verified Boot が挿入されるブート段の一般構造。
- [`../../engineering/automotive/cybersecurity-iso21434.md`](../../engineering/automotive/cybersecurity-iso21434.md) — HSM・セキュアブート・多層防御の車載版。Android Automotive OS で両者は合流する。
- [`../apis/authentication-tokens-specification.md`](../apis/authentication-tokens-specification.md) — アプリ層の認証。Keystore はそのトークン・鍵の置き場になる。
- [`boot-init-properties.md`](boot-init-properties.md) — ブートローダーから init・プロパティ・init.rc を経て Zygote に至る起動の詳細。
- [`memory-lmkd.md`](memory-lmkd.md) — 本書 §2「プロセス＝キャッシュ」の深掘り（カーネル回収・oom_score_adj・lmkd・freezer）。
- [`app-launch-end-to-end.md`](app-launch-end-to-end.md) — 本書 §5「1 タップの旅」をプロセス/スレッド単位に展開したもの。
- [`package-management.md`](package-management.md) — アプリ＝Linux ユーザの UID 割り当て・installd・packages.xml。
- [`process-thread-model.md`](process-thread-model.md) — Android のプロセス/スレッドと cgroup・優先度の対応（読む順序の第 1 層）。
- [`aidl-interfaces.md`](aidl-interfaces.md) — HAL/フレームワーク境界を定義する AIDL と stable AIDL の版管理。
- [`system-server-services.md`](system-server-services.md) — 「カーネルの外のカーネル」system_server の内部構成・起動フェーズ・Watchdog。

## 参考文献 (References)

**一次資料 (Primary — AOSP / Android 公式)**

- AOSP, *Architecture overview*. — [source.android.com/docs/core/architecture](https://source.android.com/docs/core/architecture) — Accessed: 2026-07-02。スタック構成・HAL・Treble の公式解説。
- AOSP, *Android Runtime (ART) and Dalvik*. — [source.android.com/docs/core/runtime](https://source.android.com/docs/core/runtime) — Accessed: 2026-07-02。DEX・コンパイル戦略・GC の公式解説。
- AOSP, *Low Memory Killer Daemon (lmkd)*. — [source.android.com/docs/core/perf/lmkd](https://source.android.com/docs/core/perf/lmkd) — Accessed: 2026-07-02。PSI ベースの逼迫検知とプロセス kill の公式解説。
- AOSP, *Security-Enhanced Linux in Android* ／ *Verified Boot*. — [SELinux](https://source.android.com/docs/security/features/selinux)、[Verified Boot](https://source.android.com/docs/security/features/verifiedboot) — Accessed: 2026-07-02。
- Android Developers, *Processes and app lifecycle*. — [developer.android.com](https://developer.android.com/guide/components/activities/process-lifecycle) — Accessed: 2026-07-02。プロセス重要度階層（foreground〜cached）の規範。
- Android Developers, *Android Interface Definition Language (AIDL)*. — [developer.android.com](https://developer.android.com/develop/background-work/services/aidl) — Accessed: 2026-07-02。Binder 上の RPC 定義の公式ガイド。

**二次資料 (Secondary)**

- Yaghmour, K. (2013). *Embedded Android*. O'Reilly. — init・Zygote・ネイティブ層の古典的詳説（バージョンは古いが構造の理解に有効）。
- Android Platform Team (2015–). *Stagefright 以後のメディアスタック分割に関する公式ブログ・セキュリティ告示*（Android Security Bulletins）. — [source.android.com/docs/security/bulletin](https://source.android.com/docs/security/bulletin) — Accessed: 2026-07-02。
- 有野和真 (2017).『Androidを支える技術〈I〉60fpsを達成するモダンなGUIシステム』技術評論社. ISBN 978-4-7741-8759-4 — 第1章 (GUI システムの三つの時代・Zygote・init.rc・プロパティの速習)、第7章 (Dalvik/ART とメモリ節約)。
- 有野和真 (2017).『Androidを支える技術〈II〉真のマルチタスクに挑んだモバイルOSの心臓部』技術評論社. ISBN 978-4-7741-8861-4 — 第2章 (インストールと UID)、第3章 (Low Memory Killer)、第5章 (Zygote と ActivityThread)、第6章 (ブートローダーと init)。Android 7.0 Nougat 基準。書籍内コードは著者による簡略化であり、現行 AOSP との対応は [`../../references/books/book-android-internals-arino.md`](../../references/books/book-android-internals-arino.md) を参照。
