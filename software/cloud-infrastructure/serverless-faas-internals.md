# サーバーレス実行基盤の内部 (Serverless & FaaS Platform Internals)

## 概要 (Overview)

**FaaS (Function as a Service)** は「コードを渡せば、リクエストが来たときだけ実行され、実行したミリ秒ぶんだけ課金される」という抽象である。AWS Lambda、Azure Functions、Google Cloud Functions、そしてオープン実装の OpenWhisk / Knative がこの抽象を提供する。利用者から見えるのは関数とイベントだけだが、その裏側は「**相互に信頼しない何百万ものテナントのコードを、アイドルコストほぼゼロで、ミリ秒単位に多重化する**」という、仮想化・スケジューリング・ストレージ・分散システムの総力戦である。

本ドキュメントの中心テーゼは、**FaaS が売っている商品は計算ではなく「アイドルの消滅」であり、コールドスタート (cold start) はその代金である**、ということ。常時稼働のサーバは平均利用率 10–20% でも 100% の料金がかかるが、FaaS は統計多重化 (statistical multiplexing) によってアイドル時間を他テナントに転売する。その代償として「最初のリクエストが来た瞬間に実行環境が存在しない」という問題が生まれ、プラットフォームの技術史はほぼ全て**コールドスタートの分解と各段の撃破**として読める。分析対象は AWS Lambda を中心に、一次資料（Firecracker NSDI '20、コンテナロード ATC '23、ワークロード特性 ATC '20）で機構を追う。分析深度は Level 3。

## 方法論 (Methodology)

- コールドスタートを**配置決定 → microVM 起動 → ランタイム初期化 → コード取得**の 4 段パイプラインに分解し、各段への対策（スケジューリング、microVM、スナップショット復元、遅延ロード）を独立に評価する。
- 隔離機構そのもの（VT-x, Firecracker, gVisor, コンテナ in VM）は [`../virtualization/hardware-virtualization-containers.md`](../virtualization/hardware-virtualization-containers.md) が既に扱っており、本書では再導出せず「FaaS がなぜその点を選んだか」だけを論じる。
- クラウド内部の記述はベンダー発表に依存する部分が多い。査読付き論文 (NSDI/ATC) を**一次資料**、AWS 文書・講演を**ベンダー提示 (vendor-presented)** と明示的に区別し、可能な箇所はオープン実装 (OpenWhisk/Knative) と独立測定 (ATC '18) で相互検証する。
- 数値は桁感 (order of magnitude) であり、ランタイム・メモリ設定・リージョンで容易に 1 桁動く。

## 発見事項 (Findings)

1. **FaaS の商品は「アイドルの消滅」であり、コールドスタートはその代金である。** サーバの購入単位（台・時間）と負荷の発生単位（リクエスト・ミリ秒）のミスマッチが利用率を下げる。FaaS は課金単位を実行時間そのもの（Lambda は 1ms 単位）まで細分化し、アイドルをプラットフォーム側が引き取って統計多重化で埋める。埋められるのは「使われていない実行環境をいつでも回収できる」からであり、回収した環境が再び必要になった瞬間のペナルティがコールドスタートである。アイドルコストとコールドスタートは同じ契約の表裏で、どちらかだけを消すことはできない。

2. **隔離の答えは「コンテナ in microVM」の製品化である。** 相互不信のテナント間の境界に共有カーネルのコンテナは薄すぎ、従来型 VM は密度が足りない——[`../virtualization/hardware-virtualization-containers.md`](../virtualization/hardware-virtualization-containers.md) の帰結そのままに、Lambda は 1 ワーカー上に数千の Firecracker microVM を詰め、各 microVM の中でテナントの関数をコンテナ的機構 (cgroup/namespace/seccomp) で包む。Firecracker の起動 100ms 強・メモリオーバーヘッド数 MiB（ベンダー実証値, NSDI '20）は「1ms 課金と多テナント密度を両立できる最小の硬い境界」として要件から逆算された数字である。

3. **コールドスタートは単一の遅延ではなく 4 段のパイプラインであり、各段は別の技術で削られる。** (a) 配置決定はスケジューラの応答時間の問題、(b) microVM 起動は Firecracker がデバイスモデル最小化で解いた問題、(c) ランタイム＋ユーザ初期化はスナップショット復元 (SnapStart, Catalyzer 系) が償却する問題、(d) コード・イメージ取得は遅延ロードと重複排除の問題。「コールドスタートが遅い」という一枚岩の観察は工学的には無意味で、どの段が支配的かはパッケージ形式（zip か 10 GiB コンテナイメージか）とランタイム（JVM か軽量ランタイムか）で全く変わる。

4. **イメージ配布は「ダウンロードするな、ページインせよ」で解かれた。** コンテナイメージ対応（最大 10 GiB）を素朴に実装すると、起動のたびに GiB 級の pull が走る。Lambda の実装 (ATC '23) はイメージをフラットな ext4 ブロックデバイス像に変換して 512 KiB チャンクに割り、**実際に読まれたチャンクだけ**をワーカーローカル → AZ 共有キャッシュ → S3 の三層から供給する。起動に必要なバイトはイメージ全体のごく一部という観察が前提にある。さらにチャンクを**収束暗号化 (convergent encryption)**（平文のハッシュから鍵を導出）することで、テナント間で平文を見せ合うことなく同一チャンクの重複排除が成立する——ベースイメージの大半は全テナントで共通、という分布の歪みを容量とレイテンシの両方に換金する設計である。

5. **スナップショット復元は初期化コストを「関数バージョンごとに 1 回」へ償却するが、クローンは一意性を破る。** JVM 起動＋フレームワーク初期化のような数秒級の init は、初期化済み microVM のスナップショットを取って復元 (resume) すれば省略できる (SnapStart、系譜としては Catalyzer ASPLOS '20)。だが同一スナップショットから何百ものインスタンスを複製すると、**乱数生成器の内部状態・UUID・時刻・TLS 接続がすべて同一の状態から再開する**。ユーザ空間 PRNG が同じ列を吐けば暗号は壊れる。復元検知 (vmgenid による再シード)、ランタイムフック (beforeCheckpoint/afterRestore)、接続の張り直しなど、「スナップショット安全性 (snapshot safety)」はサーバーレス時代に新しく追加された正しさの公理である。

6. **ワークロードは極端に歪んでおり、keep-warm は予測問題である。** Azure Functions の実トレース (ATC '20) では、呼び出しの大半をごく少数のアプリが占め、大多数の関数はまれにしか呼ばれない（数時間〜数日に 1 回の裾が厚い）。全関数を温めておけばアイドルが復活して経済が壊れ、全て冷やせばコールドスタートだらけになる。同論文のハイブリッドヒストグラム政策——関数ごとの呼び出し間隔の分布を学習し、pre-warm 時刻と keep-alive 長を関数別に決める——が示すように、実行環境の生存管理はキャッシュ置換問題（どの環境を追い出すか＝どのテナントに代金を払わせるか）である。

7. **「1 サンドボックス = 同時 1 リクエスト」という選択が、スケーリング・課金・プログラミングモデルの全てを規定する。** Lambda の実行環境は同時に 1 リクエストしか処理しない。ゆえに同時実行数 = 生きているサンドボックス数となり、スケールアウトの単位・課金の単位・隔離の単位が一致する。アプリケーションコードはリクエスト間の並行性を考えなくてよい（プロセス内共有状態への同時アクセスがない）代わりに、並行 1000 リクエストは 1000 環境を要求する。呼び出しの合間、環境は **freeze**（cgroup freezer 相当でCPU を剥奪）され、課金もCPU も「実行していない時間には存在しない」——この実行モデルの帰結として、バックグラウンドスレッドは凍結中に進まない。

## アーキテクチャ (Architecture)

### 呼び出し経路 — コントロールプレーンとデータプレーン

```
            同期 invoke
 Client ──────────────▶ ┌────────────┐   ワーカー割当の照会    ┌──────────────────┐
                        │ Frontend / │ ─────────────────────▶ │ Placement /      │
                        │ Router     │ ◀───────────────────── │ Worker Manager   │
                        └─────┬──────┘   (どのワーカーの        │  (在庫管理・配置) │
                              │           どのサンドボックスか)  └──────────────────┘
                     ②以降は割当を保持し ─┐                        制御プレーン
                       warm 環境へ直行    │                    ─────────────────
                              ▼          │                       データプレーン
      ┌─ Worker (ベアメタル EC2) ────────┴───────────────────────────┐
      │  ┌─ Firecracker microVM (関数 A, テナント X) ──────────────┐  │
      │  │  guest kernel + ランタイム + 関数コード                  │  │
      │  │  (内側で cgroup/namespace/seccomp によりさらに絞る)      │  │
      │  └────────────────────────────────────────────────────────┘  │
      │  ┌─ microVM (関数 B, テナント Y) ─┐ ┌─ microVM (…) ─┐ × 数千 │
      │  └────────────────────────────────┘ └───────────────┘        │
      └──────────────────────────────────────────────────────────────┘
```

- **コールドパス**: 割当が存在しないとき、Worker Manager が容量在庫から配置を決め、ワーカー上に microVM とサンドボックスを新設する。上記 4 段パイプラインが全部走る。
- **ウォームパス**: 既存サンドボックスへの sticky routing。実行が終わった環境は freeze され、次の呼び出しで thaw される。
- 配置は bin packing であると同時に**相関回避**である：同一関数・同一テナントの環境を同じワーカーに集中させると、そのテナントのバーストがワーカーを焼き、故障の爆発半径も揃ってしまう。負荷の相関が低いテナント同士を混ぜるほど統計多重化の効率は上がる（ベンダー提示）。
- この図の Frontend / Worker Manager / Placement は次章 [`cloud-control-planes.md`](cloud-control-planes.md) のコントロールプレーン設計原則（セル分割・静的安定性）に従って作られている側であり、本書は個々のワーカー上の機構に集中する。

### コールドスタートの 4 段と対策の対応

| 段 | 内容 | 支配的なケース | 対策 |
|---|---|---|---|
| ① 配置決定 | 割当照会・容量確保 | 常に一定の小さな税 | 在庫の事前確保、割当のキャッシュ |
| ② microVM 起動 | ゲストカーネルブート | （既に小さい） | Firecracker: 最小デバイスモデル、~125ms・<5MiB（ベンダー実証値） |
| ③ ランタイム/ユーザ init | JVM 起動、DI コンテナ構築等 | JVM/.NET 系で数秒級 | スナップショット復元 (SnapStart / Catalyzer 系) |
| ④ コード・イメージ取得 | zip 展開 or イメージ pull | コンテナイメージ (〜10GiB) | チャンク化＋遅延ページイン＋収束暗号 dedup (ATC '23) |

## 技術詳細 (Technical Details)

### 1. Firecracker の位置——要件からの逆算

Firecracker (NSDI '20) は KVM 上の最小 VMM であり、機構は [`../virtualization/hardware-virtualization-containers.md`](../virtualization/hardware-virtualization-containers.md) で扱った。FaaS 側から見るべきは要件の向きである：(a) 相互不信テナント間の境界はハードウェア仮想化でなければならない（共有カーネルの syscall 表面は広すぎる）、(b) 1 ワーカーに数千環境を詰めるにはゲストあたりのメモリオーバーヘッドが数 MiB でなければならない、(c) コールドスタート予算から起動は 100ms 級でなければならない、(d) デバイスは virtio-net/virtio-block/シリアルだけでよい（FaaS のゲストは GPU もUSB も要らない——当初の設計時点）。汎用 VMM (QEMU) から要らない自由度を全部捨てると、セキュリティ（攻撃面縮小）と密度（オーバーヘッド削減）が同時に手に入る、という「制約の勝利」が Firecracker の本質である。jailer によるVMM プロセス自体の封じ込め（chroot・namespace・seccomp・非特権ユーザ）で、「microVM を破っても次の壁がある」多層構造を成す。

### 2. コンテナイメージの遅延ロード (ATC '23)

Lambda のコンテナイメージ対応の実装は、イメージ配布問題をストレージ問題に変換した好例である。

- **平坦化**: OCI イメージのレイヤ構造を、起動時に重ねる (overlayfs) のではなく、**事前に 1 枚の ext4 ブロックデバイス像へ焼き込む**。決定的 (deterministic) なシリアライズにより、同じ内容からは同じバイト列が生成され、後段の重複排除が効くようになる。
- **チャンク化と遅延供給**: デバイス像を 512 KiB チャンクに分割し、ゲストのブロック読み出しに応じて**オンデマンドに**供給する。起動時に読まれるのはイメージのごく一部であり、「全部ダウンロードしてから起動」は「読まれたぶんだけページイン」に置き換わる。
- **三層キャッシュ**: ワーカーローカルキャッシュ → AZ 単位の共有キャッシュ → S3（起源）。人気イメージのチャンクはローカルにあり、テールは S3 まで落ちる。共有キャッシュは [`../object-storage/object-storage-architecture.md`](../object-storage/object-storage-architecture.md) が扱う不変データの倉庫の設計そのものである。
- **収束暗号化 (convergent encryption)**: チャンク鍵を平文チャンクのハッシュから導出する。同一平文 → 同一暗号文となりテナント横断の重複排除が可能になる一方、プラットフォームは鍵を知るテナントの平文しか読めない。「共有ベースイメージ（OS・ランタイム層）はほぼ全員同じ」という分布の歪みが、この設計の全利得の源泉である。既知のトレードオフとして、収束暗号は「平文を推測できる相手には内容の存在確認 (confirmation attack) を許す」——公開ベースイメージが大半という前提でこの弱点を許容している。

### 3. スナップショット復元と一意性回復

初期化済み VM のメモリ＋デバイス状態を保存し、呼び出し時に復元する方式 (Catalyzer ASPLOS '20、AWS SnapStart) は、③の init コストを関数バージョンあたり 1 回に償却する。復元は全メモリのロードを待たず、アクセスされたページからオンデマンドに読む (lazy restore) ことで起動をさらに縮められる。問題は**クローンの意味論**である：

- **乱数**: ユーザ空間 PRNG（OpenSSL の乱数プール、言語ランタイムの seed 済み生成器）はスナップショット時点の内部状態ごと複製される。対策は復元検知と再シード——仮想ハードウェアの **VM Generation ID (vmgenid)** が復元で変わり、ゲストカーネルがこれを検知して CSPRNG へエントロピーを再注入する (Linux では `virt: vmgenid` ドライバ以降)。ユーザ空間側にはランタイムフック（SnapStart の `beforeCheckpoint`/`afterRestore` 等）で再初期化の機会を与える。
- **時刻・期限**: スナップショット時点の時計・キャッシュ済みトークンの期限・DNS TTL は復元時点では嘘になっている。
- **接続**: TCP 接続・TLS セッションは複製できない（シーケンス番号・セッション鍵を共有するクローンが多数生まれる）。復元後の張り直しが必須。
- 一般化すると、「プロセスは一度だけ生まれ、状態は単調に進む」という暗黙の公理がスナップショットで壊れる。fork 安全性 (fork-safety) の VM 版であり、ライブラリエコシステム全体に「復元安全 (snapshot-safe) か」という新しい審査基準を課す。

### 4. keep-warm・ワークロード特性・スケーリング

- **実測に基づく政策** (ATC '20, Azure トレース): 呼び出し間隔の分布は関数ごとに桁で異なり、大半の関数はまれにしか呼ばれない。固定 keep-alive（例: 一律 N 分）は、頻繁な関数には冗長、まれな関数には無力。関数ごとの間隔ヒストグラムから pre-warm と keep-alive を決めるハイブリッド政策は、コールドスタート率とメモリ浪費を同時に改善できることを示した——実行環境プールの管理は本質的にキャッシュ置換である。
- **同時実行とスロットリング**: 「1 サンドボックス = 1 リクエスト」ゆえ、同時実行数の上限はサンドボックス在庫の上限であり、アカウント・関数単位のクォータはアドミッション制御（[`cloud-control-planes.md`](cloud-control-planes.md)）として前段で施行される。バーストは在庫（既設 microVM）と新設速度の競争になり、予約済み容量 (provisioned concurrency) は「statistical multiplexing から自分を外す」オプション、すなわちアイドル料金の再導入である——発見事項 1 の表裏一体性はここにも現れる。
- **独立測定の視点**: プラットフォーム内部は公開されないが、外側からの系統的測定 (ATC '18) は VM の共置・再利用・スケーリング挙動を関数側から推定できることを示した。コールドスタート率や環境再利用は観測可能な外形であり、ベンダー提示値の独立検証手段になる。

### 5. オープン実装との対比

| 軸 | AWS Lambda | Knative (K8s 上) | OpenWhisk |
|---|---|---|---|
| 隔離境界 | Firecracker microVM（テナント間はHW 仮想化） | Pod（= 運用者の信頼域。多テナントは別途） | コンテナ (Docker) |
| コールド対策 | 遅延ロード＋スナップショット＋在庫 | scale-from-zero (Activator が保留) | プール済みコンテナ (prewarm stem cell) |
| スケール単位 | サンドボックス (同時 1 リクエスト) | Pod (同時 N, concurrency 設定) | コンテナ |
| 状態 | 外部サービスへ強制 | 同左 | 同左 |

オープン実装がテナント境界にハードウェア仮想化を置かないのは、想定信頼モデルが「同一組織内」だからである。裏返せば、Lambda 型のパブリック FaaS の設計の大部分は**相互不信**という一語から導出されている。

## セキュリティ考察 (Security Considerations)

- **多層の隔離スタック**: 関数コード → ランタイムサンドボックス (cgroup/namespace/seccomp) → microVM (KVM/Firecracker) → jailer で封じた VMM プロセス → ワーカー。単層の強度ではなく層の直列で守る。テナント間の境界に必ずハードウェア仮想化が挟まる点が要であり、共有カーネルの権限昇格バグ 1 発では隣のテナントに届かない（[`../virtualization/hardware-virtualization-containers.md`](../virtualization/hardware-virtualization-containers.md) セキュリティ考察と同型）。
- **サイドチャネルと共置**: 同一物理ホスト上の他テナントとの共置は投機実行系チャネル（Spectre 級）の前提条件になる。緩和はホスト側での対称マルチスレッディングの扱い・コアスケジューリング・カーネル緩和策に依存し、密度（経済性）と直接衝突する。スナップショット由来の乱数複製（技術詳細 3）は暗号鍵・nonce の再利用に直結するため、復元時再シードは可用性機能ではなく**暗号学的必須要件**である。
- **実行ロールと爆発半径**: 各関数には IAM ロール（[`cloud-iam-authorization.md`](cloud-iam-authorization.md)）が紐づき、漏えい時の被害は「その関数に与えた権限」に限定される——関数単位の最小権限は、モノリスのプロセス単位権限より細かい境界を引ける点で構造的に有利。一方、イベント駆動ゆえに**入力面が広い**（キュー・ストレージイベント・HTTP）：イベントデータのインジェクションと、依存ライブラリ経由のサプライチェーン（[`../security/software-supply-chain-security.md`](../security/software-supply-chain-security.md)）が現実的な主戦場になる。
- **凍結された環境の残留状態**: warm 環境は呼び出し間で `/tmp` やグローバル変数を保持する。同一関数・同一テナント内の再利用に限定されるため他テナントへは漏れないが、「リクエストごとにまっさら」という誤解に基づくコードは、前リクエストの秘密（キャッシュされたトークン等）を後続リクエストに露出しうる。

## 結論 (Conclusions)

- FaaS は「アイドルの転売」を成立させるための機構群である。課金をミリ秒に細分化するには回収可能な実行環境が要り、回収するからコールドスタートが生まれ、コールドスタートを削るために microVM・遅延ロード・スナップショット・keep-warm 予測が積み上がった。経済モデルと技術スタックは因果で繋がった一本の鎖である。
- コールドスタートは 4 段パイプラインとして分解して初めて工学の対象になる。各段の対策——スケジューリング、Firecracker、スナップショット復元、チャンク化遅延ロード——は独立に進化しており、「サーバーレスは遅い/速い」という総括に意味はない。
- スナップショット復元と収束暗号 dedup は、いずれも「分布の歪み」（init は毎回同じ・ベースイメージは皆同じ）を性能に換金する技術である。同時に、クローンの一意性回復という新しい正しさの公理を生んだ——プラットフォームの進化がアプリケーションの正しさの定義を書き換えた例として記憶されるべきである。
- 実行モデルの単純さ（1 サンドボックス 1 リクエスト、freeze/thaw、状態の外部化）は制約ではなく、スケール単位・課金単位・隔離単位を一致させるための設計である。この一致こそが「同時実行 1000 までコードを 1 行も変えずにスケールする」という FaaS の約束の実体である。

### 次の接続先 (Related Topics)

- [`../virtualization/hardware-virtualization-containers.md`](../virtualization/hardware-virtualization-containers.md) — Firecracker・コンテナ in VM・隔離/密度/互換性のトライアングル。本書の実行基盤の機構面。
- [`cloud-control-planes.md`](cloud-control-planes.md) — 本書の Frontend / Worker Manager / Placement を支える設計原則（セル・静的安定性・アドミッション制御）。
- [`../networking/virtual-networks-overlay.md`](../networking/virtual-networks-overlay.md) — サンドボックスに ENI/VPC 接続を与える側の機構（Nitro オフロード, マッピングサービス)。
- [`../object-storage/object-storage-architecture.md`](../object-storage/object-storage-architecture.md) — イメージチャンクの起源ストアと共有キャッシュの設計原理。
- [`../../foundations/mathematics/probability-statistics/markov-chains-queueing.md`](../../foundations/mathematics/probability-statistics/markov-chains-queueing.md) — 統計多重化・到着過程・待ち行列という本書の経済側の数理。

## 参考文献 (References)

**一次資料 (Primary — 査読付き論文)**

- Agache, A., Brooker, M., Iordache, A., Liguori, A., Neugebauer, R., Piwonka, P., & Popa, D.-M. (2020). Firecracker: Lightweight Virtualization for Serverless Applications. *NSDI '20*, 419–434. — microVM の設計・起動時間/オーバーヘッドの実証値、jailer の多層防御。
- Brooker, M., Danilov, M., Greenwood, C., & Piwonka, P. (2023). On-demand Container Loading in AWS Lambda. *USENIX ATC '23*. — イメージ平坦化・512KiB チャンク・三層キャッシュ・収束暗号 dedup の一次記述。
- Shahrad, M., Fonseca, R., Goiri, Í., et al. (2020). Serverless in the Wild: Characterizing and Optimizing the Serverless Workload at a Large Cloud Provider. *USENIX ATC '20*. — Azure Functions 実トレースによる呼び出し分布の歪みとハイブリッド keep-warm 政策。
- Du, D., Yu, T., Xia, Y., Zang, B., et al. (2020). Catalyzer: Sub-millisecond Startup for Serverless Computing with Initialization-less Booting. *ASPLOS '20*, 467–481. DOI: 10.1145/3373376.3378512 — スナップショット復元による init 償却の独立実装（gVisor 基盤）。
- Wang, L., Li, M., Zhang, Y., Ristenpart, T., & Swift, M. (2018). Peeking Behind the Curtains of Serverless Platforms. *USENIX ATC '18*. — 外形測定によるプラットフォーム挙動（共置・再利用・スケーリング）の独立推定。

**ベンダー提示 (Vendor-presented)**

- AWS. *Security Overview of AWS Lambda* (whitepaper). [docs.aws.amazon.com/whitepapers/latest/security-overview-aws-lambda/](https://docs.aws.amazon.com/whitepapers/latest/security-overview-aws-lambda/security-overview-aws-lambda.html) — Accessed: 2026-07-30。実行環境の隔離スタックと freeze/thaw の一次（ベンダー）記述。
- AWS Lambda Developer Guide: Execution environment / SnapStart. [docs.aws.amazon.com/lambda/latest/dg/](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html) — Accessed: 2026-07-30。1 環境同時 1 リクエスト、ランタイムフック、復元時の一意性に関する規範記述。

**二次資料 (Secondary)**

- Linux カーネル文書: VM Generation ID ドライバ (`drivers/virt/vmgenid.c`, Linux 5.18+) — スナップショット復元検知と CSPRNG 再シードの実装。[docs.kernel.org](https://docs.kernel.org/) — Accessed: 2026-07-30。
- Knative / Apache OpenWhisk プロジェクト文書 — [knative.dev/docs/](https://knative.dev/docs/), [openwhisk.apache.org](https://openwhisk.apache.org/) — Accessed: 2026-07-30。オープン実装側の scale-from-zero・prewarm 設計の相互検証用。
