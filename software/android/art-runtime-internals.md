# ART 深掘り (ART Internals: DEX, Compilation Pipeline & GC)

## 概要 (Overview)

[`platform-architecture.md`](platform-architecture.md) は ART を「測ってからコンパイルする」ランタイムとして一段上から眺めた。本ドキュメントはその内部——**DEX 形式、dex2oat のコンパイルパイプライン、プロファイル誘導コンパイルの資材循環、GC の実装（並行コピー → userfaultfd ベースのマークコンパクト）、クラスローディングと JNI**——に降りる。

ART を深掘りする価値は、それが「JVM の移植」ではなく、**モバイルの制約（ストレージ・RAM・バッテリー・16ms のフレーム予算）から実行系を作り直した設計**である点にある: バイトコードはレジスタベース (DEX)、コンパイルはデバイス上の実測プロファイルに従い、GC は「スループットよりポーズ時間」を選び、コンパイル済みコードとヒープイメージはプロセス間で mmap 共有される。Zygote の COW 分岐・lmkd の「プロセス＝キャッシュ」と同じ設計文法が、ランタイムの内側でも貫かれている。分析深度は Level 3。

## 方法論 (Methodology)

- 「1 つのメソッドが辿る実行形態の遷移」（インタープリタ → JIT → プロファイル → AOT）を軸に、各段の成果物（DEX / プロファイル / OAT・VDEX / boot image）とその置き場・共有のされ方を追う。
- GC は「何を止めるか」を基準に世代を整理する: STW コンパクション → 並行コピー (Baker リードバリア) → userfaultfd マークコンパクト。各方式の代償（バリアのコードサイズ・スループット・ポーズ）を対比する。
- 一次資料は AOSP の ART 公式ドキュメントと `art/` ソースツリーに置き、バージョン依存の挙動は導入バージョンを付記する。数値（ヒープ既定値・閾値）はベンダ構成で変わりうることを明示する。
- JVM (HotSpot) との対比は設計判断の理由を示す範囲に留め、HotSpot 自体の詳説はしない。

## 発見事項 (Findings)

1. **DEX がレジスタベースなのはメモリのためである。** スタックマシン (JVM) より命令数が減り、複数クラスの定数プール（文字列・型・メソッド表）を 1 ファイルに統合して重複を消す。「バイトコードの形式」がインタープリタ速度・mmap 共有・ストレージの三方を同時に決めている。
2. **ART のコンパイルは「どこで・いつ・何を」の三軸で分散している。** 初回実行はインタープリタ (nterp)、ホットメソッドは端末上の JIT、蓄積プロファイルは充電中の background dexopt で AOT (`speed-profile`)、さらに Play 経由のクラウドプロファイル (Cloud Profiles) とアプリ同梱の Baseline Profile が「初回からホットパス AOT 済み」を先回りで実現する。**コンパイルはインストール時の一回のイベントではなく、端末群をまたぐ継続的な資材循環**である。
3. **コンパイル済みコードは「第二の共有ライブラリ」である。** boot image (`boot.art`/`boot.oat`) はコアクラスの**初期化済みヒープごと** mmap され、Zygote 経由で全アプリが物理 1 部を COW 共有する。AOT の成果物設計が、プロセスモデル (Zygote) とメモリ管理 (COW) に直結している。
4. **並行コピー GC の本体は「読むたびに直す」リードバリアである。** Baker スタイルのリードバリアが「移動済みオブジェクトへの古い参照」を読み取り時に修正するため、**アプリを走らせたままリージョンをコンパクション**できる。ポーズはミリ秒未満に抑えられるが、代償として**全参照読み取りにバリアコードが載る**（コードサイズ・スループット税）。
5. **Android 13 の userfaultfd GC は「バリアの税をカーネルに肩代わりさせた」。** ページフォルトを userspace で受ける userfaultfd により、移動中ページへのアクセスだけを捕捉して直す並行マークコンパクトへ移行。リードバリア一式がコードから消え、**コードサイズ・スループット・断片化のすべてが改善**した——「ハードウェア/カーネル機構で言語ランタイムの不変条件を守らせる」という [`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) 的な解法。
6. **クラスローディングは「検証をいつ済ませるか」の設計である。** DEX の検証結果は VDEX に永続化され、再インストール・更新時の再検証を省く。ClassLoader 階層はアプリの依存分割 (dynamic feature) と Treble 的な境界 (system/vendor のクラスパス分離) の両方を支える。
7. **JNI は「速いが、ランタイムの不変条件の外」である。** ローカル/グローバル参照表を介した参照管理は GC の移動と両立するための間接層であり、`@FastNative`/`@CriticalNative` はその間接層を削る代わりに制約（GC との相互作用禁止等）を課す。**「参照はハンドルで持つ」規律**は Binder のハンドル翻訳（[`binder-internals.md`](binder-internals.md)）と同型の設計文法。

## アーキテクチャ (Architecture)

### 1 メソッドの一生 — 実行形態の遷移

```
 APK インストール
   └─ dex2oat (install 時): verify のみ (+同梱 Baseline Profile があれば AOT)
        │
 初回実行:  nterp (アセンブリ実装インタープリタ) で走る
        │      ホットネスカウンタが閾値超過
        ├──▶ JIT コンパイル (プロセス内, コードキャッシュへ)
        │      ProfileSaver がホットメソッド/クラスを .prof に記録
        │
 充電中・アイドル (background dexopt):
        └──▶ dex2oat --compiler-filter=speed-profile
               プロファイル記載メソッドだけ AOT → OAT/VDEX/ART ファイル更新
 次回起動: AOT 済みコードを mmap して即実行（JIT 予熱が不要）
```

- **実行形態は常に混在**する: 同一プロセス内で AOT 済み・JIT 済み・インタープリタ実行のメソッドが呼び合い、デオプティマイゼーション（デバッガ接続・プロファイル無効化）で AOT/JIT からインタープリタへ**戻る**こともできる。
- コンパイルフィルタの主要段: `verify`（検証のみ・実行はインタープリタ/JIT）→ `speed-profile`（プロファイル記載分を AOT）→ `speed`（ほぼ全メソッド AOT、システムアプリ向け）。**既定が speed-profile であること**が「測ってからコンパイルする」の実装形である。

### 成果物の地図 — 何がどこに置かれ、誰と共有されるか

| 成果物 | 中身 | 共有のされ方 |
|---|---|---|
| DEX (APK 内) | レジスタベースのバイトコード＋統合定数プール | mmap、全プロセスで物理 1 部 |
| VDEX | 検証済み DEX（検証結果の永続化） | 同上。再検証の省略が目的 |
| OAT/ODEX | AOT 済みネイティブコード | mmap、実行可能・読み取り専用 |
| ART (.art) | 事前初期化済みオブジェクト（文字列・クラス）のヒープイメージ | mmap + COW |
| boot.art / boot.oat | コアライブラリ版の上記（boot image） | Zygote 経由で**全アプリ**が COW 共有 |
| .prof | ホットメソッド/クラスの集合 | 端末内 background dexopt ＋ Play へ集約 (Cloud Profiles) |

- boot image は「フレームワーククラスのロード・検証・初期化・JIT 予熱」を**ビルド時に済ませてスナップショット化**したものと読める。Zygote のプリロード（[`platform-architecture.md`](platform-architecture.md)）は、この mmap 済みイメージの上でさらにアプリ共通分を温める第二段である。

## 技術詳細 (Technical Details)

### 1. DEX 形式と nterp

- **レジスタベース**: 各メソッドは仮想レジスタ配列上で動き、JVM のオペランドスタック操作（push/pop の列）が 1 命令に畳まれる。命令数が減る分、インタープリタのディスパッチ回数が減る——インタープリタ滞在時間が長い ART の戦略と整合する。
- **統合定数プール**: 文字列・型・メソッド・フィールドの表を DEX ファイル単位で共有し、クラスごとの定数プール (JVM .class) の重複を排除する。64K メソッド参照上限（multidex の由来）はこの表のインデックス幅の帰結。
- **nterp** (Android 12+): C++ の switch インタープリタ (mterp 以前) を置き換えるアーキテクチャ別アセンブリ実装。JIT/AOT とスタックフレーム形式を揃え、形態遷移（インタープリタ⇄コンパイル済み）の境界コストを下げた。
- **dexlayout / DEX 最適化**: プロファイルに基づき「起動時に触るクラス・メソッド」を DEX 内で物理的に近接配置し、ページイン局所性を高める（[`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) のページキャッシュ効率への直接介入）。

### 2. dex2oat と Optimizing コンパイラ

- 単一の中間表現 **HIR**（SSA ベースのグラフ）上で、インライン化・境界チェック除去・ループ最適化・intrinsic 置換（`String.indexOf` 等をハンドコード実装へ）を行う標準的な最適化コンパイラ。JIT と AOT は**同一コンパイラの呼び出し方の違い**であり、生成コードの形式も共通。
- **JIT モード**: プロセス内のコードキャッシュ（実行可能ページ）に書く。容量は限られ、GC される（コールドになったコードは捨てる）。Zygote 起動後の system_server などには **JIT zygote**（Zygote 内で予熱し fork で配る）最適化がある。
- **AOT モード (dex2oat)**: `dex2oat` は独立プロセスとして走り、cgroup で背景優先度に絞られる（[`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) の cgroup 適用例）。生成物は上表の OAT/VDEX/ART。
- **プロファイルの中身**: 「ホットメソッド集合」に加え「起動時ロードクラス集合」を含み、AOT 対象選定と dexlayout・.art イメージの事前初期化対象の両方に使われる。

### 3. GC — 並行コピーから userfaultfd マークコンパクトへ

- **ヒープ構造**: リージョンベースのメインヒープ＋大オブジェクト空間 (LOS)＋イメージ空間 (.art、ほぼ回収対象外)。割り付けはリージョン内バンプポインタ＋スレッドローカルバッファ (TLAB) で、malloc より数命令の桁で軽い。
- **並行コピー CC (Android 8+)**: マーク・退避 (evacuation)・参照修正をアプリスレッドと並行に行う。核は **Baker リードバリア**——参照ロード時に「from 空間を指していたら to 空間の新住所へ直す」チェックを全参照読み取りに埋め込む。ポーズは flip（全スレッドのルート切替）の瞬間だけで、ミリ秒未満。**16ms フレーム予算の中で GC が起きても落ちない**ことが設計の支配的要求。
- **世代別化 (Android 10+)**: 若い世代だけの minor CC を高頻度・低コストで回し、full CC を稀にする。「短命オブジェクト多数」というアプリの実態（UI のイベント・一時オブジェクト）への標準的応答。
- **userfaultfd CMC (Android 13+)**: コンパクション中のページを `userfaultfd` で保護し、アプリが触った瞬間にフォルトを userspace GC スレッドが受けて**そのページだけ先に修復**する。リードバリアが不要になり、(a) 全コードからバリア命令が消えてコードサイズ・スループットが改善、(b) 退避コピーではなく**その場スライドのコンパクト**で断片化とメモリ使用が改善。バリア方式 10 年の後に「カーネル機構で置き換える」判断がされた点が示唆的。
- **背景遷移との連携**: アプリが背面に回ると、ヒープを詰めてフットプリントを縮める GC が走る——lmkd に殺される前に「キャッシュとしての居住コスト」を自ら下げる、プラットフォーム協調の一部。

### 4. クラスローディング・検証・ClassLoader 階層

- **ClassLinker** が DEX からのクラス解決・リンク・初期化を担う。ロード済みクラスはクラステーブルに入り、boot image のクラスは**最初から解決済み**の状態で mmap される。
- **検証 (verification)**: バイトコードの型健全性検査。結果は VDEX へ永続化され、OTA 更新時は boot classpath の変化に応じて要否が再判定される。「検証をいつ払うか」が install/boot/初回実行のレイテンシ配分を決める。
- **ClassLoader 階層**: boot → system → アプリ (PathClassLoader) の委譲連鎖。dynamic feature / shared library は子ローダとして接がれ、**「どの DEX がどの名前空間に見えるか」**を階層が規定する。ネイティブライブラリ側にも対応する linker namespace があり、system/vendor の境界 (Treble) はクラスパスとリンカ名前空間の双方で貫かれる。

### 5. JNI — 境界のコスト構造

- Java 側オブジェクトはネイティブへ**生ポインタでは渡らない**。ローカル参照（呼び出しスコープ）・グローバル参照（明示解放）・弱グローバルの**参照表エントリ**として渡り、GC は表を root として辿り、移動時は表を書き換える——移動 GC と JNI が両立する唯一の接点。
- 境界コスト: 引数のマーシャリング・スレッド状態遷移（GC が「ネイティブ実行中」を待たなくてよいようにする runnable/native の遷移）・例外チェック。`@FastNative` は遷移を軽くし、`@CriticalNative` は参照受け渡し自体を禁じて生値のみにする——**速さは不変条件の放棄と引き換え**。
- 参照表のあふれ（ローカル参照のリーク）は即クラッシュであり、「ハンドルで持ち、スコープで返す」規律が要る。fd・Binder ハンドルと同じ「テーブル＋不透明ハンドル」文法である。

## セキュリティ考察 (Security Considerations)

- **DEX/OAT は攻撃者入力である**: サイドロードされた APK の DEX は任意に細工できる。検証器 (verifier) は型混同（verifier を騙して `int` を参照として扱わせる類）に対する第一防壁であり、verifier のバグはサンドボックス内コード実行からの脱出起点になりうる。OAT/VDEX の**ロード時整合性**（dm-verity 圏外の `/data` に置かれる）も、ブート検証の連鎖（[`platform-architecture.md`](platform-architecture.md) の Verified Boot）から見ると相対的に弱い環である。
- **JIT コードキャッシュは W^X の例外地帯**: 実行可能ページへの書き込みが定常的に起きるため、ART は書込用/実行用の二重マッピング分離等でコードキャッシュ改竄の難度を上げている。プロファイル (.prof) の細工が「どのメソッドが AOT されるか」を歪める余地も、資材循環（Cloud Profiles 集約側の検証）で扱われるべき面。
- **JNI 境界は型安全の崖**: 参照表の誤用・生ポインタの持ち出し・GC 移動の失念は、Java 側の型健全性をネイティブ側の UAF に変換する。プラットフォームが JNI 依存を減らし Rust 化を進める動機の一部（[`binder-internals.md`](binder-internals.md) の Rust バックエンドと同じ系譜）。
- **デバッグ機構の両義性**: デオプティマイゼーションとメソッド差し替え（インストルメンテーション）は正規のデバッグ経路だが、`debuggable` フラグの誤設定や run-as の穴はアプリ改竄・資格情報窃取の経路になる。リリースビルドで JDWP が閉じていることはサンドボックス境界の一部である。

## 結論 (Conclusions)

- ART は「バイトコード形式 (DEX)・コンパイル時期 (JIT+背景 AOT)・成果物の共有 (boot image/COW)・GC (ポーズ最優先)」のすべてをモバイル制約から逆算した実行系であり、個々の判断は Zygote・lmkd・Treble と同じ設計文法——**共有で節約し、実測で最適化し、いつ殺されてもよい形に保つ**——の現れである。
- コンパイルは端末内で完結するイベントではなく、**プロファイルが端末⇄クラウドを循環する継続プロセス**になった。実行系の性能が「配布インフラの設計」に依存する段階に入っている。
- GC の系譜（STW → リードバリア並行コピー → userfaultfd マークコンパクト)は、「言語ランタイムの不変条件をコード埋め込みで守るか、カーネル/HW 機構で守るか」というトレードオフの実験場であり、後者への移行が現時点の答えである。
- 検証器・JIT キャッシュ・JNI という「型安全の境界」は、そのままセキュリティ境界である。実行系の深掘りは Android のサンドボックス理解（多層防御のどの層が実行系に依存しているか）と不可分である。

### 次の接続先 (Related Topics)

- [`platform-architecture.md`](platform-architecture.md) — ART を含む全体像（Zygote・lmkd・Verified Boot）。boot image と COW 分岐の関係。
- [`binder-internals.md`](binder-internals.md) — 「テーブル＋不透明ハンドル」文法の IPC 版。Rust 化の動機の共通性。
- [`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) — mmap・COW・userfaultfd・ページイン局所性の一般論。本書はその実行系での活用。
- [`../concurrency/memory-models-and-synchronization.md`](../concurrency/memory-models-and-synchronization.md) — リードバリア・並行 GC が依拠する順序保証の一般論。

## 参考文献 (References)

**一次資料 (Primary — AOSP / ソース)**

- AOSP, *Android Runtime (ART) and Dalvik* ／ *Configuring ART*（コンパイルフィルタ・background dexopt の公式解説）. — [source.android.com/docs/core/runtime](https://source.android.com/docs/core/runtime) — Accessed: 2026-07-03。
- AOSP, *Implement ART just-in-time compiler*. — [source.android.com/docs/core/runtime/jit-compiler](https://source.android.com/docs/core/runtime/jit-compiler) — Accessed: 2026-07-03。JIT・ProfileSaver・speed-profile の循環の公式解説。
- AOSP `art/` ソースツリー（ClassLinker, ConcurrentCopying, MarkCompact, nterp 実装）. — [android.googlesource.com/platform/art](https://android.googlesource.com/platform/art/) — Accessed: 2026-07-03。
- Android Developers, *Baseline Profiles overview*. — [developer.android.com/topic/performance/baselineprofiles/overview](https://developer.android.com/topic/performance/baselineprofiles/overview) — Accessed: 2026-07-03。同梱プロファイルと Cloud Profiles の規範。
- Android Developers Blog / AOSP (2023). *Concurrent Mark Compact (userfaultfd) GC の導入解説*（Android 13 リリースノート・ART モジュール更新）. — [source.android.com/docs/core/runtime/gc-debug](https://source.android.com/docs/core/runtime/gc-debug) — Accessed: 2026-07-03。

**二次資料 (Secondary)**

- Google ART チームによる各種カンファレンス発表（Google I/O・Android Dev Summit の ART セッション: nterp, userfaultfd GC, Mainline 化）。— [youtube.com/@AndroidDevelopers](https://www.youtube.com/@AndroidDevelopers) — Accessed: 2026-07-03。
- JVM (HotSpot) との比較参照として: Lindholm, T. et al. (2015). *The Java Virtual Machine Specification, Java SE 8 Edition*. Oracle. — スタックマシン・クラスファイル定数プールとの対比の基準。
