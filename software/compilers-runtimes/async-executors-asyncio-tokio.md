# 非同期実行器の内部 — asyncio と Tokio (Async Executors: asyncio & Tokio Internals)

## 概要 (Overview)

[`language-implementations-cpython-go-rust.md`](language-implementations-cpython-go-rust.md) は非同期処理を「関数の色」問題への三つの答えとして設計レベルで比較した。本ドキュメントはその実行側の黒箱——**実行器 (executor / event loop)**——を開ける。`await` と書いたとき、誰がタスクを再開し、誰が I/O の完了に気づき、誰が「次に走るタスク」を選んでいるのか。Python の asyncio と Rust の Tokio を主対象に、Go は前掲書で扱った GMP (実行器がランタイムに溶けている形) を対照として参照する。

中心テーゼは、**実行器とは OS スケジューラのユーザ空間再演である**、ということ。レディキュー・wakeup・公平性・飢餓・ワークスティーリング——OS のスケジューラ ([`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md)) が解いてきた問題の一式が、コルーチン/Future という「再開可能な計算」の上でもう一度解かれる。ただし決定的な違いが一つある: **プリエンプションがない**。協調的世界では「タスクが手放すまで誰も走れない」が公理であり、実行器の設計の相当部分は、この公理の副作用 (1 個の重いタスクが全体を止める) をどう緩和するかに費やされる。

第二のテーゼは、**asyncio と Tokio は駆動方向が逆である**こと。asyncio は「準備できたコールバックを実行する」push 型の機械であり、コルーチンはその上に橋を架けた二階部分にすぎない。Rust の Future は「poll されるまで何もしない」pull 型であり、I/O の完了 (push) と poll (pull) を接続する蝶番が **Waker** である。この駆動方向の違いが、タスクの表現・キャンセルの哲学・性能特性のすべてに波及する。

本書は (1) asyncio のイベントループの一周、(2) Rust のタスク機構 (poll/Waker) と Tokio のワークスティーリング実行器、(3) 協調世界での公平性 (LIFO スロット・協調的バジェット)、(4) キャンセルの三哲学、(5) ブロッキング混入とその防衛、を一次資料 (PEP 3156, Rust RFC, Tokio 設計記事) を基に整理する。分析深度は Level 3。

## 方法論 (Methodology)

- 実行器を同じ 5 要素——タスクの表現・レディキュー・I/O 多重化との接続・タイマ・キャンセル——に分解して比較し、「asyncio 対 Tokio」を個別ツアーではなく同型問題の別解として読めるようにする。
- `../concurrency/` との棲み分け: メモリモデル・同期プリミティブの一般論は [`../concurrency/memory-models-and-synchronization.md`](../concurrency/memory-models-and-synchronization.md) に、OS のスケジューリングとイベント多重化 (epoll 等) は [`../operating-systems/`](../operating-systems/) に委ね、本書は**言語ランタイムの上に組まれた実行器そのもの**に集中する。
- 一次資料を優先する: asyncio は PEP 3156 と CPython `Lib/asyncio/`、Rust は RFC 2592 (task API) と std ドキュメント、Tokio は公式設計記事 (スケジューラ刷新・協調的 yield)。性能の主張は出典と計測主体を併記する (Tokio の数値は第一者計測)。
- Go は独立の節を立てず、各論点の対照として引く (実行器を言語ランタイムに吸収した設計は前掲書 GMP 節で詳述済み)。

## 発見事項 (Findings)

1. **asyncio の本体はコルーチンではなくコールバック機械である。** イベントループの核は「`call_soon` で積まれた Handle (コールバック) を FIFO で実行し、待ちがあれば selector (epoll/kqueue) で眠る」だけの単純な機械で、PEP 3156 はまずこのコールバック層を仕様化した。コルーチンは Task という**翻訳器**で二階に載る: Task は `coro.send()` で本体を「次の await まで」進め、返ってきた Future に完了コールバックを登録して自分を眠らせる。`await` の連鎖は、最下層では Future の done-callback に還元される——async/await は構文糖であるという説明は、asyncio では文字通り実装の記述である。
2. **Rust の Future は「何もしない」ことが仕様である。** Future は poll されない限り一切進まない遅延値であり、実行器の仕事は「wake されたタスクだけを poll し直す」こと。I/O リソースは準備未了なら `Pending` を返す前に **Waker を登録**し、完了時に `wake()` がタスクをレディキューへ戻す——push (I/O イベント) と pull (poll) の蝶番が Waker であり、RFC 2592 が固めたこの契約 (spurious wake 許容・最後の Waker のみ有効) が、実行器とリソースを疎結合にする。busy-poll なし・コールバック連鎖なしで「必要なタスクだけが走る」構造は、この契約の帰結である。
3. **Tokio のスケジューラは GMP の再発明であり、それは偶然ではない。** ワーカごとの固定長ローカルキュー (256)・グローバル注入キュー・空いたワーカが他から**半分盗む** work stealing・定期的 (61 tick ごと) なグローバルキュー確認による飢餓防止——2019 年の刷新で Tokio が到達した形は Go の GMP とほぼ同型である。同じ問題 (M 個のスレッドに大量の軽量タスクを公平かつキャッシュ効率よく分配する) は同じ答えに収斂する——[`stdlib-internals-cpython-go-rust.md`](stdlib-internals-cpython-go-rust.md) の SwissTable 収斂と同じ現象の、スケジューラ版である。
4. **LIFO スロットと協調的バジェットは「プリエンプションの不在」への対症療法である。** Tokio は直近に wake されたタスク 1 個だけを LIFO で優先する特別枠を持つ (メッセージパッシングの往復レイテンシをキャッシュが温かいうちに刈り取る) が、これは飢餓の種でもあるため盗み不可・連続使用に上限を課す。さらにタスクごとに操作回数の**バジェット (128)** を配り、使い切ったらリソース層が強制的に `Pending` を返して yield させる——協調世界に「時間切れ」を擬似的に持ち込む機構であり、テールレイテンシ改善の第一者計測が報告されている。asyncio には対応物がなく、重いコールバックの検出 (debug mode の遅延警告) という**観測**で代替する。
5. **キャンセルの哲学が三言語で正反対である。** asyncio は `CancelledError` をタスクの**次の await 地点に注入**する (例外として捕捉・掃除・拒否すらできる——3.8+ で BaseException 直系になり、うっかり握り潰しは減った)。Rust は Future を **drop するだけ**——poll されなくなった計算は存在ごと消え、後始末は Drop (RAII) が担う。Go は context という**値の規約**で「キャンセルされたか各自確認せよ」と伝搬する。注入・破棄・合意——強制力が強い順に、cancel-safety の問題の形も変わる: Rust では「途中まで進んだ操作が drop で消える」(`select!` でのメッセージ紛失が典型)、asyncio では「握り潰しと finally での再入」、Go では「確認を怠るタスクは止まらない」。
6. **構造化並行性は「タスクの所有権」の再発見である。** fire-and-forget したタスクはエラーの行き先も生存期間の管理者も失う——asyncio は「ループはタスクを弱参照でしか持たない」という有名な罠 (参照を持たないと GC がタスクを回収しうる) を抱え、Rust では JoinHandle の drop がタスクを切り離す。応答として両者は**親スコープがタスクを所有し、離脱時に合流を強制する**構造化並行性 (asyncio TaskGroup (3.11)、Tokio JoinSet、先行者は Trio の nursery) に収斂した。「値に所有者を」という Rust の中心思想が、タスクという動的な存在に輸入された形である。

## アーキテクチャ (Architecture)

### asyncio — イベントループの一周

```
        ┌─────────────────────────────────────────────────┐
        │ ① レディキュー (deque of Handle) を今ある分だけ実行  │
        │    Task.__step → coro.send() → 次の await で停止   │
        │    (実行中に積まれた分は次周へ — 飢餓防止)            │
        ▼                                                 │
   ② 直近タイマまでの残り時間を計算 (heapq の先頭)             │
        ▼                                                 │
   ③ selector.select(timeout) — epoll/kqueue で眠る         │
      I/O イベント → 対応するコールバックをレディキューへ        │
        ▼                                                 │
   ④ 期限が来たタイマをレディキューへ ──────────────────────────┘
```

- ループ自体はコールバックしか知らない。コルーチンとの接続は Task が担う: `await fut` で停止した Task は `fut.add_done_callback(self.__wakeup)` を登録し、完了時に `__wakeup` がレディキューに積まれて `coro.send()` が再開する。
- uvloop はこのループ層 (②③④と Handle 管理) を libuv + Cython で置換した互換実装であり、「二階 (コルーチン) はそのまま、一階だけ差し替え可能」という PEP 3156 の層設計の証明になっている。

### Tokio — ワークスティーリング実行器

```
 ワーカスレッド (通常 CPU 数)                     グローバル注入キュー
 ┌──────────────────────────────┐              (spawn 元がワーカ外のとき等)
 │ Worker 0: [LIFO slot][local q 256] │←── 61 tick ごとに必ず確認 (飢餓防止)
 │ Worker 1: [LIFO slot][local q 256] │
 └──────────────────────────────┘
   ・LIFO slot: 直近 wake の 1 タスク優先 (盗み不可・連続上限あり)
   ・local queue が空 → 他ワーカから半分盗む (work stealing)
   ・全員暇 → 1 ワーカが I/O ドライバ (mio: epoll/kqueue) とタイマホイールを poll
   ・タスク = poll されるまで動かない Future。wake() がキューへ戻す
```

- タイマは階層タイミングホイール (Varghese & Lauck) で O(1) 挿入・削除。I/O ドライバは専用スレッドではなく**暇なワーカが引き受ける**——スレッドを増やさず遅延を削る 2019 年刷新の設計判断である。

## 技術詳細 (Technical Details)

### 1. asyncio の解剖 — 二層構造の代償と利点

- **Handle と call_soon が通貨である**: `loop.call_soon(cb)` はスレッド非安全 (ループスレッド専用)、外から積むのは `call_soon_threadsafe` (self-pipe でループを起こす)。タイマは `call_at`/`call_later` が TimerHandle を heapq に積む。すべての「起きること」がこの通貨に両替される。
- **Task.__step の実体**: `coro.send(None)` は次の `await` で停止するまで**同期実行**される——つまり await と await の間は不可分であり、これが「協調的」の正確な意味である。yield された Future への done-callback 登録が「眠り」、その発火が「目覚め」で、プリエンプションはどこにも存在しない。
- **既知の罠が層設計から導出できる**: (a) ブロッキング呼び出しの混入は selector に到達する機会を奪い全タスクを止める (debug mode は 100ms 超のコールバックを警告)、(b) `run_in_executor` はブロッキング処理をスレッドプールに逃がす公式の脱出口、(c) タスクの弱参照問題——`create_task` の戻り値を握らないと GC がタスクごと回収しうる——は「ループはコールバック機械であり、タスクの所有者ではない」ことの帰結である (TaskGroup が所有者を提供する)。

### 2. Rust のタスク機構 — 契約としての poll

- **`poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<T>`** が全てである。`Pending` を返すなら、将来 `wake` される手筈を自分で整えるのが Future 側の義務 (整えず Pending を返せばタスクは永眠する——実行器はポーリングしない)。`Poll::Ready` 後の再 poll は契約違反。spurious wake は許容され、poll は冪等に書く。
- **Waker は vtable 手書きの動的ディスパッチ** (`RawWaker`): 任意の実行器が自分のタスク表現を Waker に包めるよう、trait object ではなく手動 vtable で定義される——std が実行器を同梱しない (実行器はエコシステムに委ねる) という Rust の決定の技術的表現であり、この unsafe 境界の健全性はエコシステム全体が相乗りする土台である。
- **タスクは 1 個のステートマシン**: `async fn` の変換 ([`language-implementations-cpython-go-rust.md`](language-implementations-cpython-go-rust.md) 参照) により、タスクの「スタック」は enum 1 個に平坦化され、Tokio のタスクはそれを 1 回のヒープ割り当てに収める。深い await の連鎖も追加割り当てなし——goroutine のスタック成長と対照的な、コンパイル時に大きさが決まる世界である。

### 3. 公平性・レイテンシ・飢餓 — 協調世界の統治

- **FIFO は公平、LIFO は速い**: レディキューの基本は FIFO (全員に順番が回る)。しかし req-rep パターンでは「いま wake したタスクを即走らせる」方が桁違いに速い (キャッシュ・遅延)。Tokio の LIFO スロットはこの二律の妥協で、Go スケジューラの runnext と同じ発想である——ここでも収斂。
- **バジェットの意味論**: Tokio の協調的バジェットは「リソース (channel 受信・ソケット読み等) が Ready を返せる回数」を 128 に制限し、超えたら人工的に Pending + self-wake させる。**awaitしていても yield しないループ** (常に Ready が返る受信ループ等) による飢餓を、リソース層で強制切断する仕組みであり、プリエンプションの「時間で切る」を「回数で切る」に翻訳したものである。
- **asyncio の統治は観測に寄る**: バジェット相当がないため、重いタスクは実測で見つけて (py-spy, debug mode) `await asyncio.sleep(0)` (明示的 yield) や `run_in_executor` で手動分割する。統治を機構で行うか運用で行うか——ここにも設計思想の差が出る。

### 4. キャンセルと構造化並行性

- **asyncio**: `task.cancel()` → 次の await で `CancelledError` 注入。`finally`/async context manager が掃除を担うが、掃除の中でさらに await できる (そして再キャンセルされうる) ため、キャンセル処理は本質的に再入的である。`asyncio.timeout()`/`TaskGroup` (3.11) は「スコープを出るとき全子タスクの合流/取消を保証する」構造化の道具。
- **Tokio/Rust**: キャンセル = drop。同期的・即時・拒否不能で、後始末は各リソースの Drop 実装に分散する。**cancel-safety** は API の文書化事項になる (`tokio::select!` で分岐が捨てられたとき、途中状態が失われない操作か?)——「受信済みメッセージが drop で消えた」が典型事故で、`CancellationToken` や JoinSet が構造化側の道具である。async Drop (drop の中で await したい) が言語に無いことが、Rust 側の現在進行形の欠落として残る。
- **対照としての Go**: context は「キャンセルを知る手段」であって「止める力」を持たない——確認しない goroutine は止まらず、リークする。強制力の弱さと引き換えに、途中状態が勝手に消える事故は構造的に起きない。

### 5. 観測の入口 (Observability)

| 疑問 | asyncio | Tokio |
|---|---|---|
| いま何のタスクがあるか | `asyncio.all_tasks()`, aiomonitor | tokio-console (タスク一覧・poll 時間・wake 回数) |
| ループ/ワーカを塞いだ犯人 | debug mode の遅延コールバック警告, py-spy | tokio-console の busy 時間, `tracing` スパン |
| タスクリーク | 「Task was destroyed but it is pending!」警告 | JoinSet/JoinHandle の未合流, console の孤児タスク |
| レイテンシの尖り | ループ遅延の計測 (call_soon の遅延) | バジェット/LIFO の効果は metrics (`tokio_runtime_metrics`) |

## セキュリティ考察 (Security Considerations)

- **実行器の可用性は「最も重い 1 タスク」に律速される。** プリエンプションがない以上、CPU-bound な 1 リクエスト (巨大 JSON のパース、圧縮、正規表現のバックトラック爆発) が全接続を止める——[`stdlib-internals-cpython-go-rust.md`](stdlib-internals-cpython-go-rust.md) の HashDoS と同型の「計算量を握られる」攻撃面が、協調的実行器では**プロセス全体の停止**に増幅される。防衛は入力サイズの制限・タイムアウト・重い処理の隔離 (spawn_blocking / run_in_executor) の多層で行う。
- **非同期はコネクション保持を安価にする——攻撃者にとっても。** slowloris 型 (大量の遅い接続を維持する) 攻撃に対し、スレッドサーバは早々に枯渇することで「自衛」していたが、async サーバは 10 万接続を平然と抱えてメモリで死ぬ。**タイムアウトと同時接続数の上限を明示するのは async では必須の設計事項**であり、tokio/asyncio がタイムアウトを一級 API (asyncio.timeout, tokio::time::timeout) にしたのはこのためである。
- **無制限の spawn とキューはメモリ DoS の形である。** 「リクエストごとに spawn」「無限バッファのチャネル」は、背圧 (backpressure) の欠如として攻撃可能になる。有界チャネル・セマフォによる同時実行数制限が防衛の基本形で、これは [`../messaging-streaming/`](../messaging-streaming/) の背圧設計と同じ問題の縮小版である。
- **キャンセル経路はエラー処理の死角になりやすい。** asyncio の CancelledError の握り潰し (歴史的に `except Exception` で捕まった時代の名残)、Rust の drop による途中状態の消失、Go の context 無視——キャンセルは「正常でも異常でもない第三の終了」であり、リソースリーク・ロック保持・不変条件の破れがこの経路に溜まる。構造化並行性 (TaskGroup/JoinSet) の採用は、この死角を型とスコープで塞ぐセキュリティ対策としても読める。

## 結論 (Conclusions)

- 実行器は OS スケジューラのユーザ空間再演であり、Tokio と Go が独立に同じ形 (ローカルキュー + 半分盗み + 特別枠 + 定期的なグローバル確認) に到達したことは、この問題の解が設計者の好みではなく問題の構造で決まることを示す。
- asyncio (push のコールバック機械にコルーチンの橋) と Rust/Tokio (pull の poll 契約と Waker の蝶番) は駆動方向が逆であり、その差はタスク表現 (動的な Future 連鎖 vs 平坦化された 1 ステートマシン)・性能・キャンセル哲学 (注入 vs 破棄) まで一貫して波及する。
- プリエンプションの不在が協調的実行器の原罪であり、LIFO スロット・協調的バジェット・遅延警告はすべてその緩和策である。可用性の観点では、この原罪は「1 タスクの計算量が全体を人質に取る」攻撃面として現れる。
- 構造化並行性への収斂 (nursery → TaskGroup / JoinSet) は、「タスクにも所有者を」という所有権思想の輸出であり、エラーの行き先・生存期間・キャンセルの死角という三つの問題を一つのスコープ規律で解く——前掲書の支払時期テーゼで言えば、実行時の規約だったタスク管理を構文とスコープに前倒しする動きである。

### 次の接続先 (Related Topics)

- [`language-implementations-cpython-go-rust.md`](language-implementations-cpython-go-rust.md) — 本書の親。async 設計の三答と「関数の色」問題。
- [`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) — 再演元の OS スケジューラと epoll 等の I/O 多重化。
- [`../concurrency/memory-models-and-synchronization.md`](../concurrency/memory-models-and-synchronization.md) — wake/poll の下にある同期と可視性の一般論。
- [`managed-runtimes-gc-jit.md`](managed-runtimes-gc-jit.md) — 「停止を薄く塗る」「観測で統治する」という同型の設計原理。
- [`../messaging-streaming/`](../messaging-streaming/) — 背圧・有界キューの一般論。

## 参考文献 (References)

**一次資料 (Primary — 仕様・設計文書・論文)**

- [PEP 3156 — Asynchronous IO Support Rebooted: the "asyncio" Module](https://peps.python.org/pep-3156/) — コールバック層とコルーチン層の二層設計の原典。Accessed: 2026-08-15。
- [Rust RFC 2592 — futures](https://rust-lang.github.io/rfcs/2592-futures.html) — poll/Waker 契約の安定化文書。Accessed: 2026-08-15。
- Varghese, G., & Lauck, T. (1987). Hashed and Hierarchical Timing Wheels. *SOSP '87*. DOI: 10.1145/41457.37504 — Tokio のタイマ機構の原典。
- CPython ソース — [`Lib/asyncio/base_events.py`](https://github.com/python/cpython/blob/main/Lib/asyncio/base_events.py)、[`Lib/asyncio/tasks.py`](https://github.com/python/cpython/blob/main/Lib/asyncio/tasks.py) — Accessed: 2026-08-15。

**二次資料 (Secondary — 第一者解説・実装ドキュメント)**

- Lerche, C. (2019). [Making the Tokio scheduler 10x faster](https://tokio.rs/blog/2019-10-13-scheduler) — ワークスティーリング実行器の設計判断 (数値は第一者計測)。Accessed: 2026-08-15。
- Tokio Blog (2020). [Reducing tail latencies with automatic cooperative task yielding](https://tokio.rs/blog/2020-04-preemption) — 協調的バジェットの導入解説。Accessed: 2026-08-15。
- Smith, N. J. (2018). [Notes on structured concurrency, or: Go statement considered harmful](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) — 構造化並行性 (nursery) の原典エッセイ。Accessed: 2026-08-15。
- MagicStack (2016). [uvloop: Blazing fast Python networking](https://magic.io/blog/uvloop-blazing-fast-python-networking/) — ループ層差し替えの実証。Accessed: 2026-08-15。
- [Tokio ドキュメント — tokio::select! と cancel safety](https://docs.rs/tokio/latest/tokio/macro.select.html)、[tokio-console](https://github.com/tokio-rs/console) — Accessed: 2026-08-15。
