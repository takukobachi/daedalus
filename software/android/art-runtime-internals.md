# ART 深掘り (ART Internals: DEX, Compilation Pipeline & GC)

## 概要 (Overview)

[`platform-architecture.md`](platform-architecture.md) は ART を「測ってからコンパイルする」ランタイムとして一段上から眺めた。本ドキュメントはその内部——**DEX 形式、dex2oat のコンパイルパイプライン、プロファイル誘導コンパイルの資材循環、GC の実装（並行コピー → userfaultfd ベースのマークコンパクト）、クラスローディングと JNI**——に降りる。

ART を深掘りする価値は、それが「JVM の移植」ではなく、**モバイルの制約（ストレージ・RAM・バッテリー・16ms のフレーム予算）から実行系を作り直した設計**である点にある: バイトコードはレジスタベース (DEX)、コンパイルはデバイス上の実測プロファイルに従い、GC は「スループットよりポーズ時間」を選び、コンパイル済みコードとヒープイメージはプロセス間で mmap 共有される。Zygote の COW 分岐・lmkd の「プロセス＝キャッシュ」と同じ設計文法が、ランタイムの内側でも貫かれている。分析深度は Level 3。

## 方法論 (Methodology)

- 「1 つのメソッドが辿る実行形態の遷移」（インタープリタ → JIT → プロファイル → AOT）を軸に、各段の成果物（DEX / プロファイル / OAT・VDEX / boot image）とその置き場・共有のされ方を追う。
- GC は「何を止めるか」を基準に世代を整理する: STW コンパクション → 並行コピー (Baker リードバリア) → userfaultfd マークコンパクト。各方式の代償（バリアのコードサイズ・スループット・ポーズ）を対比する。
- 一次資料は AOSP の ART 公式ドキュメントと `art/` ソースツリーに置き、バージョン依存の挙動は導入バージョンを付記する。数値（ヒープ既定値・閾値）はベンダ構成で変わりうることを明示する。
- JVM (HotSpot) との対比は設計判断の理由を示す範囲に留め、HotSpot 自体の詳説はしない。
- 有野 (2017) Ⅰ巻 第 7 章（Android 7.0 Nougat 基準）を機構レベルの基準線とし、現在の AOSP（ART Service/artd, nterp, Mainline 化）との差分を技術詳細 6 に表で明示する。書籍の記述は言い換えて引用し、現行の挙動は AOSP 公式文書・ソースで確認した。

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

### 成果物設計の根 — Clean/Dirty × Shared/Private

上表の「共有のされ方」列がなぜ設計の中心なのかは、ページを 2 軸で分類すると見える（有野 (2017) Ⅰ巻 7.3.2–7.3.4）。

| | Private | Shared |
|---|---|---|
| **Clean**（ファイル内容と一致） | 中: 捨てられるが 1 プロセス分 | **最良**: 捨てられ、かつ物理 1 部 |
| **Dirty**（書き換え済み・無名） | **最悪**: 捨てられず、プロセス数だけ増える | 中: COW 共有中は 1 部 |

- **なぜ Clean が効くのか**: フラッシュ（eMMC/UFS）は書き込みが遅く寿命も有限なため、Android は伝統的にディスクスワップを持たない。するとメモリ逼迫時にカーネルが**捨てられるのはファイルと同内容の Clean ページだけ**になる——捨てても次のアクセスでページフォルトから再読込できるからである。DEX/VDEX/OAT を**再配置なしでそのまま mmap して実行できる形式**にしてあるのは、コード領域全体を Clean に保つためであり、DEX を APK 内で**無圧縮**格納するのも、展開すればその先が Dirty になるからである。mmap は触れたページしか実メモリに載らないので、巨大なフレームワークの「使われないコード」はそもそもメモリを消費しない（同 7.3.3–7.3.4）。現在の Android はディスクスワップの代わりに zram（メモリ内圧縮スワップ）で Dirty 無名ページの一部を圧縮退避するが、「Clean は捨てるだけで済む」という非対称は変わらない——詳細は [`memory-lmkd.md`](memory-lmkd.md)。
- **なぜ Zygote のプリロードが元を取るのか**: fork 直後は全ページが COW で共有されるため、Zygote が**事前に**ロード・初期化したクラスは、書き換えられない限り全アプリで Shared のまま残る。書籍の算術例に倣えば、10MB を使うアプリ 3 つが 8MB を共有できれば実消費は 3×2+8=14MB で、全 Private の 30MB の半分以下になる。副次効果として、VM 初期化とクラスロードを毎回やり直さずに済む分**起動も速い**（同 7.3.4）。アプリ単位の起動経路は [`app-launch-end-to-end.md`](app-launch-end-to-end.md)。
- **JIT の成果物が「最悪象限」に落ちる理由**: JIT コードはプロセス内のコードキャッシュ（無名メモリ）に書かれるので Dirty かつ Private であり、しかもバイトコードより膨らむ。同じフレームワークメソッドを 10 プロセスが JIT すれば 10 部できる。AOT 済みの boot.oat が mmap＋fork 前ロードで Clean・Shared になるのと正反対であり、**「JIT は必要最小限、共通部分は AOT」という ART の配分はこの表から導かれる**（同 7.4.2, 7.4.3）。現在の JIT zygote（system_server/Zygote 内で JIT したコードを fork で配る）は、この欠点を部分的に Shared 側へ寄せる工夫と読める。
- 同じ理由で、クラスを列挙するリフレクション型ライブラリ（アノテーション走査など）は、使わないクラスまで触ってページインさせる——Clean で「まだ載っていない」はずの領域を実メモリに引き込むため、サーバ Java ほど歓迎されない（同 7.3.4 コラム）。

## 技術詳細 (Technical Details)

### 1. DEX 形式と nterp

- **レジスタベース**: 各メソッドは仮想レジスタ配列上で動き、JVM のオペランドスタック操作（push/pop の列）が 1 命令に畳まれる。命令数が減る分、インタープリタのディスパッチ回数が減る——インタープリタ滞在時間が長い ART の戦略と整合する。
- **小さな比較例**（有野 (2017) Ⅰ巻 7.2.2–7.2.4 の例を再構成）: `int r = add(3, 4)` は、スタック型 (JVM) では「3 を積む → 4 を積む → `invokestatic #n`（引数は暗黙にスタック上）→ 結果をローカルへ `istore`」の 4 命令、レジスタ型 (Dalvik) では「`const/4 v0, 3` → `const/4 v1, 4` → `invoke-static {v0, v1}, add` → `move-result v0`」の 4 命令で、単純な呼び出しでは差が出ない。差が出るのはループ等の実コードで、`for (i<n) res+=1` 相当ではスタック型が 12 命令（load/store の往復が要る）に対しレジスタ型は 7 命令、かつ Dalvik 命令は 16 ビット単位で揃い、オペランドにレジスタ番号を直接書ける。**命令数が少ない＝インタープリタが速い・コードが小さい**のがレジスタ型の利点、**値の生存区間がスタックの積み降ろしから機械的に読める＝JIT/AOT のレジスタ割付が容易**なのがスタック型の利点で、JIT を持たず RAM も乏しかった初期 Android がレジスタ型を選んだのは当時として必然、というのが書籍の整理である。Optimizing コンパイラが SSA 形式の HIR に変換してから割付けを行うのは、このスタック型の利点を解析で取り戻す現代的な解と読める。
- **DEX の 2 つの設計目的**（同 7.2.5）: (1) 文字列・型名などの定数を**複数クラスで共有**してサイズを削る（`Ljava/lang/Object;` のような文字列はほぼ全 .class が個別に持つ）、(2) **再配置なしで mmap して実行できる**形式にする。命令中の定数はすべて ID（固定幅インデックス）で参照されるので命令長が一定になりパースも速い。書籍が引く初期の公式比較では、無圧縮 jar 100% に対し圧縮 jar 50%、無圧縮 DEX 46% と、**圧縮なしで圧縮 jar 並み**に収まる——無圧縮であることが上記の Clean mmap の前提である。
- **統合定数プール**: 文字列・型・メソッド・フィールドの表を DEX ファイル単位で共有し、クラスごとの定数プール (JVM .class) の重複を排除する。64K メソッド参照上限（multidex の由来）はこの表のインデックス幅の帰結。
- **nterp** (ARM では Android 12+): Nougat で導入されたアセンブリ実装インタープリタ mterp（source.android.com *ART improvements*）の後継となるアーキテクチャ別アセンブリ実装。`art/runtime/interpreter/mterp/` は android-11.0.0_r1 で `nterp.cc` と x86_64 用の `x86_64ng/` のみを持ち、android-12.0.0_r1 で `arm64ng/`・`armng/` が加わる（android.googlesource.com/platform/art のタグ比較, Accessed: 2026-09-27）。JIT/AOT とスタックフレーム形式を揃え、形態遷移（インタープリタ⇄コンパイル済み）の境界コストを下げた。
- **dexlayout / DEX 最適化**: プロファイルに基づき「起動時に触るクラス・メソッド」を DEX 内で物理的に近接配置し、ページイン局所性を高める（[`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) のページキャッシュ効率への直接介入）。

### 2. dex2oat と Optimizing コンパイラ

- 単一の中間表現 **HIR**（SSA ベースのグラフ）上で、インライン化・境界チェック除去・ループ最適化・intrinsic 置換（`String.indexOf` 等をハンドコード実装へ）を行う標準的な最適化コンパイラ。JIT と AOT は**同一コンパイラの呼び出し方の違い**であり、生成コードの形式も共通。
- **JIT モード**: プロセス内のコードキャッシュ（実行可能ページ）に書く。容量は限られ、GC される（コールドになったコードは捨てる）。Zygote 起動後の system_server などには **JIT zygote**（Zygote 内で予熱し fork で配る）最適化がある。
- **AOT モード (dex2oat)**: `dex2oat` は独立プロセスとして走り、cgroup で背景優先度に絞られる（[`process-thread-model.md`](process-thread-model.md)、[`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) の cgroup 適用例）。生成物は上表の OAT/VDEX/ART。
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
- **ClassLoader 階層**: boot → system → アプリ (PathClassLoader) の委譲連鎖。dynamic feature / shared library は子ローダとして接がれ、「**どの DEX がどの名前空間に見えるか**」を階層が規定する。ネイティブライブラリ側にも対応する linker namespace があり、system/vendor の境界 (Treble) はクラスパスとリンカ名前空間の双方で貫かれる。

### 5. JNI — 境界のコスト構造

- Java 側オブジェクトはネイティブへ**生ポインタでは渡らない**。ローカル参照（呼び出しスコープ）・グローバル参照（明示解放）・弱グローバルの**参照表エントリ**として渡り、GC は表を root として辿り、移動時は表を書き換える——移動 GC と JNI が両立する唯一の接点。
- 境界コスト: 引数のマーシャリング・スレッド状態遷移（GC が「ネイティブ実行中」を待たなくてよいようにする runnable/native の遷移）・例外チェック。`@FastNative` は遷移を軽くし、`@CriticalNative` は参照受け渡し自体を禁じて生値のみにする——**速さは不変条件の放棄と引き換え**。
- 参照表のあふれ（ローカル参照のリーク）は即クラッシュであり、「ハンドルで持ち、スコープで返す」規律が要る。fd・Binder ハンドルと同じ「テーブル＋不透明ハンドル」文法である。

### 6. Dalvik → Lollipop AOT → Nougat ハイブリッド → 現在

書籍（有野 (2017) Ⅰ巻 7.1.2, 7.4, 7.5 および 1.2.4）は Android 7.0 Nougat を、AOT・JIT・profile guided コンパイル・イメージファイルを組み合わせたハイブリッド方式の完成形として描いた。その骨格はほぼそのまま現在も生きており、変わったのは**誰が・どこで・何の情報でコンパイルを駆動するか**の周辺である。

| 機構 | Nougat (有野 2017) | 現在（Android 14〜16 世代） | 変化の理由 |
|---|---|---|---|
| バイトコード | Dalvik バイトコード（レジスタ型）＋ DEX。dx が .class から変換（Jack は実験段階） | 形式は同じ。変換は **d8**（dex 化）＋ **R8**（縮小・最適化）に統合、Jack は廃止 | 実行側の形式を変えずツールチェーンだけ進化させる方が互換性コストが低い |
| 64K 上限 | メソッド ID が 16 ビット（0〜0xffff）。5.0 から複数 DEX をネイティブ対応、4.4 以前はライブラリ頼み | 上限自体は不変。**API 21+ はネイティブ multidex が既定**、R8 の縮小で上限に届きにくくする方針が第一推奨 | ファイル形式の ID 幅は互換性上変えられないので、回避策側（複数 DEX・縮小）が標準化した |
| インタープリタ | switch（実体は関数ポインタ表）型の C++ 実装と、アセンブリ実装の mterp | **nterp**（ARM は Android 12〜、x86_64 は 11 から）: JIT/AOT とフレーム形式を揃えたアセンブリ実装 | 「まず解釈実行」の時間が長い以上、解釈⇄コンパイル済みの遷移コストを下げる価値が大きい |
| JIT の単位 | Dalvik（2.2〜4.4）は**トレース JIT**（実行パスの断片）→ N で**メソッド単位 JIT** に | メソッド単位 JIT を継続（OSR・JIT zygote 等を追加） | メソッド単位なら JIT と AOT（dex2oat）で同じコンパイラモジュールを共有でき、プロファイルも「メソッド集合」として AOT に渡せる。書籍はこれをトレース JIT を捨てた主因と推測 |
| AOT の時期 | 5.0〜6.x は**インストール時にフル AOT**。N ではブートクラスパスと `coreApp` 等だけを事前 AOT、他は初回 JIT | OTA・初回ブート=`verify`、背景=`speed-profile`。インストール時は Android 13 以前の推奨値が `pm.dexopt.install=speed-profile`（同梱／クラウドのプロファイルがあれば該当分だけ AOT） | フル AOT は OTA のたびに全アプリ再コンパイル（数十分級）を強い、月例セキュリティ更新と両立しなかった。使われないアプリまでコンパイルする無駄も大きい |
| プロファイル採取 | アプリプロセスの **ProfileSaver** スレッドが JIT コードキャッシュからメソッド一覧を周期保存（`/data/misc/profiles/cur/<user>/<pkg>/primary.prof`） | 同じ骨格。クラス・起動時メソッド・ホット度など情報量が拡張され、dexlayout や .art 生成にも使用 | 採取点を JIT に同居させると追加コストがほぼゼロで済む |
| 背景コンパイルの駆動 | system_server の **BackgroundDexOptService**（JobScheduler: アイドル＋充電中、1 日 1 回）→ installd に `merge_profiles` → **profman** が cur を ref にマージし差分が閾値超なら → installd が **dex2oat** `--profile-file` | Android 14〜は **ART Service**（`libartservice`、ART モジュール内）が dexopt を所管し、特権作業（ファイル操作・dex2oat 起動、および **profman** の起動: `art/artd/artd.cc` の `mergeProfiles`・`isProfileUsable`・`copyAndRewriteProfile` が `GetProfman()` で得た profman を `art_exec` 経由で実行, main @ android.googlesource.com, Accessed: 2026-09-27）は **artd** デーモンが担う。bg-dexopt は依然「1 日 1 回、アイドル＋充電中」で、熱状態 moderate 以上や使用再開で中断 | dexopt ロジックを OS 本体（PackageManager/installd）から更新可能な ART モジュールへ移し、Play 経由で改善を配るため |
| コンパイル理由とフィルタ | install・bg-dexopt・first-boot 等の理由ごとに `pm.dexopt.*` プロパティ | 理由は first-boot / boot-after-ota / boot-after-mainline-update / install（＋install-fast 等の派生）/ bg-dexopt / inactive / cmdline 等。既定は bg-dexopt=`speed-profile`、boot 系=`verify`、他アプリから使われる共有ライブラリ=`speed`、長期未使用は `inactive` で `verify` へ**降格** | 「測ってから必要分だけ」を理由ごとに細分化し、ストレージ回収（降格）まで同じ枠組みで扱う |
| プロファイルの出所 | 端末内の実行のみ（初回は必ず JIT で温める） | **Cloud Profiles**（Play が多数端末の実測を集約し、Android 9+ にインストール時配布）＋ **Baseline Profiles**（開発者が `assets/dexopt/baseline.prof` を同梱、初回起動から約 30% 高速化と公称） | 「一度使わないと速くならない」ハイブリッドの弱点を、プロファイルを端末の外から先回り供給して消す |
| アプリイメージ (.art) | N で一般アプリにも導入。クラスをロードし自明な初期化まで済ませたヒープを保存し、起動時に mmap して初期化を省略。oat と相互に絶対アドレス参照 | プロファイル駆動（`speed-profile`）でプロファイル記載クラスについて生成される点は同じ。boot image 側も「ART モジュール分」と「フレームワーク分」(boot image extension) に分かれ、ART モジュール更新後は `odrefresh` が ART APEX 外の boot classpath（framework.jar 等）と system_server jar を端末上で再コンパイルする構成になった | JIT 回帰で遅くなった起動時間を AOT 時代並みに戻すための仕掛けだった（書籍の推測）が、起動最適化の標準部品として定着 |
| ART 自体の更新 | OS に固定（OTA でしか変わらない） | **ART は Mainline（APEX）モジュール**（Android 12〜）。Google Play システムアップデートで ART・コンパイラ・GC を更新 | ランタイムのセキュリティ修正・最適化を端末メーカーの OTA 周期から切り離す |

- **変わらなかったもの**: バイトコード形式（DEX）、「成果物は mmap して Clean に保つ」方針、「全体ではなくホットな少数だけをコンパイルする」戦略、プロファイルのマージ閾値で再コンパイル要否を決める構造は、書籍の記述がそのまま当てはまる。JIT に戻ったことは、実行時さえ速ければコンパイルに時間をかけてよいわけではないという結論を意味し、コンパイルが容易なレジスタ型の意義はまだある、という書籍の見立ても、ART が DEX を維持している事実と整合する。
- **注意（書籍の限界）**: 書籍時点では、プロファイルはコンパイルするか否かの判定にしか使われていないと明記されていたが、現在はアプリイメージに載せるクラスの選定や DEX レイアウト（dexlayout）などにも使われる。また ART Service の内部 API は Android バージョンごとに変わっており、上表の理由名・既定値は AOSP 公式の *ART Service configuration* と `ReasonMapping.java` を基準にした（端末ベンダは `pm.dexopt.*` で上書き可能）。

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
- [`../concurrency/memory-models-synchronization.md`](../concurrency/memory-models-synchronization.md) — リードバリア・並行 GC が依拠する順序保証の一般論。
- [`memory-lmkd.md`](memory-lmkd.md) — Clean/Dirty・Shared/Private の分類が lmkd・zram・PSS 計測でどう使われるか。本書の「成果物を Clean に保つ」設計の受け皿。
- [`app-launch-end-to-end.md`](app-launch-end-to-end.md) — Zygote fork → `handleBindApplication`（ProfileSaver 起動点）→ 初回フレームまで。AOT/JIT/アプリイメージが起動時間のどこに効くか。
- [`package-management.md`](package-management.md) — インストール時 dexopt の起点（PackageManagerService → ART Service/artd）と APK・split・DEX の配置。
- [`process-thread-model.md`](process-thread-model.md) — dex2oat・GC スレッドの cgroup/優先度配置。

## 参考文献 (References)

**一次資料 (Primary — AOSP / ソース)**

- AOSP, *Android Runtime (ART) and Dalvik* ／ *Configuring ART*（コンパイルフィルタ・background dexopt の公式解説）. — [source.android.com/docs/core/runtime](https://source.android.com/docs/core/runtime) — Accessed: 2026-07-03。
- AOSP, *Implement ART just-in-time compiler*. — [source.android.com/docs/core/runtime/jit-compiler](https://source.android.com/docs/core/runtime/jit-compiler) — Accessed: 2026-07-03。JIT・ProfileSaver・speed-profile の循環の公式解説。
- AOSP `art/` ソースツリー（ClassLinker, ConcurrentCopying, MarkCompact, nterp 実装）. — [android.googlesource.com/platform/art](https://android.googlesource.com/platform/art/) — Accessed: 2026-07-03。
- Android Developers, *Baseline Profiles overview*. — [developer.android.com/topic/performance/baselineprofiles/overview](https://developer.android.com/topic/performance/baselineprofiles/overview) — Accessed: 2026-07-03。同梱プロファイルと Cloud Profiles の規範。
- Android Developers Blog / AOSP (2023). *Concurrent Mark Compact (userfaultfd) GC の導入解説*（Android 13 リリースノート・ART モジュール更新）. — [source.android.com/docs/core/runtime/gc-debug](https://source.android.com/docs/core/runtime/gc-debug) — Accessed: 2026-07-03。

- AOSP, *ART Service configuration*（Android 14〜の dexopt 理由・既定コンパイラフィルタ・背景ジョブ条件）. — [source.android.com/docs/core/runtime/configure/art-service](https://source.android.com/docs/core/runtime/configure/art-service) — Accessed: 2026-09-27。
- AOSP, *Package manager configuration*（Android 13 以前の `pm.dexopt.*` 既定値）. — [source.android.com/docs/core/runtime/configure/package-manager](https://source.android.com/docs/core/runtime/configure/package-manager) — Accessed: 2026-09-27。
- AOSP `art/artd/README.md`（artd = system_server に無い特権で dex2oat 起動・ファイル操作を行う shim）および `art/libartservice/service/java/com/android/server/art/ReasonMapping.java`（コンパイル理由の定義）. — [android.googlesource.com/platform/art/+/refs/heads/main/artd](https://android.googlesource.com/platform/art/+/refs/heads/main/artd/) — Accessed: 2026-09-27。
- Android Developers, *Enable multidex for apps with over 64K methods*（API 21+ のネイティブ multidex、R8 による縮小の推奨）. — [developer.android.com/build/multidex](https://developer.android.com/build/multidex) — Accessed: 2026-09-27。

**二次資料 (Secondary)**

- 有野和真 (2017).『Androidを支える技術〈I〉60fpsを達成するモダンなGUIシステム』技術評論社. ISBN 978-4-7741-8759-4. — 第 7 章「バイトコード実行環境 Dalvik と ART」（7.2 Dalvik バイトコードと dex、7.3 メモリ節約の工夫と Zygote、7.4 JIT/AOT の変遷、7.5 Nougat の ProfileSaver・BackgroundDexOptService・profman・dex2oat・.art）および 1.2.4。Android 7.0 (android-7.0.0_r6)・2016 年 12 月時点の記述であり、コード片は著者による簡略化擬似コード。本ドキュメントでは Nougat の基準線として使用。
- AOSP, `art/odrefresh/README.md`（ART APEX 更新後に boot classpath extension と system_server の成果物を再生成するツール）. — [android.googlesource.com/platform/art/+/refs/heads/main/odrefresh/README.md](https://android.googlesource.com/platform/art/+/refs/heads/main/odrefresh/README.md) — Accessed: 2026-09-27。
- Android Developers Blog (2023). *Latest ARTwork on hundreds of millions of devices*（ART の Google Play システムアップデート配信が Android 12 以降の全端末に及ぶこと、ART 13 更新で起動最大 30% 改善）. — [android-developers.googleblog.com](https://android-developers.googleblog.com/2023/08/latest-artwork-on-hundreds-of-millions-of-devices.html) — Accessed: 2026-09-27。ART の Mainline 化時期の一次確認。
- XDA Developers (2021). *Google will make the Android Runtime (ART) a Mainline module in Android 12*. — [xda-developers.com](https://www.xda-developers.com/google-android-runtime-art-mainline-module-android-12/) — Accessed: 2026-09-27。ART の Mainline 化時期の確認用（一次資料は AOSP Mainline 文書）。

- Google ART チームによる各種カンファレンス発表（Google I/O・Android Dev Summit の ART セッション: nterp, userfaultfd GC, Mainline 化）。— [youtube.com/@AndroidDevelopers](https://www.youtube.com/@AndroidDevelopers) — Accessed: 2026-07-03。
- JVM (HotSpot) との比較参照として: Lindholm, T. et al. (2015). *The Java Virtual Machine Specification, Java SE 8 Edition*. Oracle. — スタックマシン・クラスファイル定数プールとの対比の基準。
