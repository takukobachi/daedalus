# プロセス・スレッドとスケジューリング (Process/Thread Scheduling)

## 概要 (Overview)

CPU は1つの物理コアにつき1時点で1つの実行流れ (hardware thread) しか走らせられない。しかし利用者から見れば、数百のプロセスが「同時に」動いているように見える。この錯覚を作るのが**スケジューリング (scheduling)** であり、OS が有限のコアを時間軸で多重化 (time-multiplexing) し、各実行主体に CPU を順番に貸し出す仕組みである。

スケジューラが扱う実行主体には2つの粒度がある。**プロセス (process)** はアドレス空間・ファイルディスクリプタ・権限を束ねた資源の容れ物であり、**スレッド (thread)** はその中の実行流れ（プログラムカウンタ・レジスタ・スタック）である。Linux の実装ではこの区別すら相対化され、スケジューリングの最小単位はプロセスでもスレッドでもなく**タスク (task)** という統一概念になっている。

本ドキュメントは、(1) プロセス/スレッドという抽象がどう状態機械として表現され、(2) **文脈切り替え (context switch)** がハードウェア（レジスタ・ページテーブル・TLB）をどう操作し、(3) **プリエンプション (preemption)** がタイマ割り込みでどう実現され、(4) スケジューリングアルゴリズムが round-robin から Linux の CFS / EEVDF へどう進化したか、(5) マルチコア・NUMA・電力・リアルタイムでそれぞれ何が問題になるか、を機構レベルで整理する。CPU 内部の仕組み（割り込み・仮想記憶・キャッシュコヒーレンス）は隣接する hardware/ ドキュメント群を土台として参照する。

## 方法論 (Methodology)

- スケジューリングを**ポリシー (policy)** と**機構 (mechanism)** に分けて論じる。「誰を次に走らせるか」（ポリシー = アルゴリズム）と「どうやって走らせ替えるか」（機構 = 文脈切り替え）は独立した問題であり、混同が理解の妨げになる。
- 評価軸は終始**スループット (throughput)・レイテンシ (latency)・公平性 (fairness)・予測可能性 (predictability)** の4つ。汎用 OS は前3者を、リアルタイム OS は最後の予測可能性を最優先する。この優先順位の違いがアルゴリズムの系譜を分岐させる。
- アルゴリズムは round-robin → 優先度 → MLFQ → 公平配分 → CFS → EEVDF と漸進的に拡張し、「各方式が前世代のどの欠陥を消すために生まれたか」を差分で説明する。
- 数値は一般的な x86-64 / ARM サーバ世代の桁感 (order of magnitude) であり、世代・実装で大きく変動する。絶対値より**相対コストの序列**を理解の基準とする。
- ハードウェア機構（タイマ割り込み・TLB flush・キャッシュ親和性）への接続を常に明示し、スケジューリングが純粋なソフトウェア問題ではなく**ハードウェア協調 (hw/sw co-design)** であることを示す。

## 発見事項 (Findings)

1. **プロセスとスレッドの境界は実装で溶ける。** 古典的には「プロセス = 資源 + 1スレッド」だが、Linux では `clone(2)` のフラグ集合（アドレス空間共有 `CLONE_VM`、FD 共有 `CLONE_FILES` 等）の濃淡として連続的に表現され、スケジューラはどちらも `task_struct` として等しく扱う。スレッドとは「資源共有の度合いが高いタスク」にすぎない。

2. **文脈切り替えの真のコストはレジスタ退避ではなく、キャッシュと TLB の冷却 (cold cache/TLB) である。** レジスタ保存復元は数十〜百ナノ秒の直接コストだが、切り替え後に作業セットが L1/L2 から追い出され、TLB がフラッシュされることによる**間接コスト**が支配的で、これは数マイクロ秒相当に達しうる（`../../hardware/cpu-architecture/virtual-memory-mmu.md`、`../../hardware/cpu-architecture/cache-coherence.md`）。

3. **プリエンプションはタイマ割り込みという「外部からの強制」で初めて成立する。** 協調的 (cooperative) スケジューリングは走行中タスクの自発的な譲渡に依存し、暴走タスクを止められない。周期的なタイマ割り込み（`../../hardware/cpu-architecture/interrupts-exceptions-traps.md`）がカーネルに制御を奪い返す機会を与え、これがプリエンプティブ・マルチタスクの物理的基盤である。

4. **公平性は「重み付き仮想時間 (weighted virtual time)」で定式化される。** Linux CFS は固定タイムスライスを捨て、各タスクが消費した CPU 時間を nice 値で重み付けした **vruntime** で管理し、最小 vruntime のタスクを赤黒木 (red-black tree) で O(log N) に選ぶ。後継の EEVDF はこれに**締切 (eligible deadline)** を加え、レイテンシ要求を公平性と両立させる。

5. **マルチコアではスケジューリングが「どのコアか」という配置問題に拡張される。** 単一の実行可能キューでは競合が爆発するため per-CPU 実行可能キュー (runqueue) に分割され、その代償として**負荷分散 (load balancing)** が必要になる。さらに NUMA（`../../hardware/cpu-architecture/cache-coherence.md`）と CPU 親和性 (affinity)、消費電力（`../../hardware/hw-sw-interface/clock-power-reset.md`）が配置判断に絡み、スケジューラは多目的最適化問題になる。

## アーキテクチャ (Architecture)

### プロセス状態機械 (Process State Machine)

各タスクは少数の状態のいずれかにあり、イベントで遷移する。中核は **実行中 (running)・実行可能 (ready/runnable)・待機 (blocked/waiting)** の3状態。

```
                  ┌──────────────────────────────────────┐
                  │              スケジューラが選択         │
                  │            (dispatch / context switch)  │
                  ▼                                        │
            ┌───────────┐    タイムスライス満了 /        ┌──┴────────┐
   fork ──▶ │  READY    │    高優先度タスクの出現 (preempt)│  RUNNING  │
   /clone   │ (runnable)│ ◀──────────────────────────────│ (on CPU)  │
            └───────────┘                                 └──┬─────┬──┘
                  ▲                                          │     │
       I/O完了 /  │                              I/O要求 /   │     │ exit()
       ロック解放 │            ┌───────────┐    sleep /       │     ▼
       (wakeup)   └────────────│ BLOCKED   │◀── ロック待ち ───┘  ┌────────┐
                               │ (waiting) │    (block)            │ZOMBIE/ │
                               └───────────┘                      │ DEAD   │
                                                                  └────────┘
```

- **READY → RUNNING (dispatch)**: スケジューラがそのタスクを選び、文脈切り替えで CPU を与える。ここで初めて実コードが走る。
- **RUNNING → READY (preempt)**: タイムスライス満了、または自分より優先すべきタスクが現れたとき、CPU を奪われ実行可能キューへ戻る。**自発的でない**点が後述プリエンプションの核心。
- **RUNNING → BLOCKED (block)**: I/O 発行・ロック獲得失敗・`sleep` で、待機事象が満たされるまで CPU を手放す。BLOCKED のタスクはスケジューラの選択対象から外れる（実行可能キューに居ない）。
- **BLOCKED → READY (wakeup)**: 待っていた事象（I/O 完了割り込み、ロック解放、シグナル）が来ると実行可能キューへ復帰する。
- **ZOMBIE**: `exit` 後、親が `wait` で終了ステータスを回収するまで `task_struct` の残骸が残る状態。資源は解放済みだが PID とステータスだけ残る。

Linux の対応では `TASK_RUNNING`（READY と RUNNING の両方を兼ねる）、`TASK_INTERRUPTIBLE` / `TASK_UNINTERRUPTIBLE`（BLOCKED の2種、シグナルで起きるか否か）、`EXIT_ZOMBIE` がこれらに当たる。

### プロセスとスレッドの構造 (Process vs Thread)

```
   プロセス A (アドレス空間・FD・権限を共有)        プロセス B
  ┌───────────────────────────────────┐      ┌──────────────┐
  │  ページテーブル / FD表 / シグナル設定  │      │  ページテーブル  │
  │  ┌─────────┐ ┌─────────┐ ┌────────┐ │      │  ┌─────────┐   │
  │  │ Thread1 │ │ Thread2 │ │Thread3 │ │      │  │ Thread1 │   │
  │  │ PC/レジ │ │ PC/レジ │ │PC/レジ │ │      │  │ PC/レジ │   │
  │  │ スタック │ │ スタック │ │スタック │ │      │  │ スタック │   │
  │  └─────────┘ └─────────┘ └────────┘ │      │  └─────────┘   │
  └───────────────────────────────────┘      └──────────────┘
   スレッド間は VM/FD 共有、スタックとレジスタのみ私有
```

- **スレッド私有 (per-thread)**: プログラムカウンタ、汎用レジスタ、スタック、スレッドローカルストレージ (TLS)。文脈切り替えで保存復元されるのはこの私有部分。
- **プロセス共有 (per-process)**: アドレス空間（ページテーブル）、ファイルディスクリプタ表、シグナルハンドラ、権限 (uid/gid)。
- 同一プロセス内スレッド間の切り替えはページテーブルを共有するため**TLB をフラッシュせずに済む**（後述する文脈切り替えコストの大きな差）。異プロセス間の切り替えはアドレス空間が変わるため TLB の汚染が避けられない。

### M:N スレッディング (Threading Models)

ユーザレベルスレッドとカーネルスレッドの対応付けには3つのモデルがある。

```
   1:1 (kernel thread)        N:1 (user thread)         M:N (hybrid)
  U U U U                    U U U U                   U U U U U U
  │ │ │ │                    └─┬─┴─┘                    └┬┘ └┬┘ └┬┘
  K K K K                      K (1本)                   K   K   K
  各Uに1カーネルスレッド        全Uを1カーネルで多重化       M本のUをN本のKに写像
```

| モデル | 並列性 | ブロッキング | 切り替えコスト | 代表 |
|---|---|---|---|---|
| **1:1** | 真の並列（複数コア） | 1スレッドの I/O が他を止めない | カーネル介入が要る（重い） | Linux NPTL, Windows |
| **N:1** | 単一コアに制限 | 1つがブロックすると全停止 | ユーザ空間のみで極軽量 | 初期 green thread |
| **M:N** | 真の並列 | スケジューラアクティベーションで回避 | 軽量だが実装複雑 | Go goroutine, 旧 Solaris |

- **1:1** が現代の汎用 OS の標準。各ユーザスレッドがカーネルから見える `task_struct` を持ち、カーネルスケジューラが直接コアへ割り当てる。並列性とブロッキング独立性が得られる代わりに、生成と切り替えがシステムコール（`../../hardware/hw-sw-interface/abi-syscalls.md`）を伴う。
- **M:N** は多数の軽量スレッドを少数のカーネルスレッド上で多重化し、ユーザ空間スケジューラで切り替える。Go のランタイムが goroutine を OS スレッド (M) 上で多重化するのが典型。ブロッキングシステムコール時にカーネルスレッドを増やす「スケジューラアクティベーション (scheduler activation)」が成否の鍵で、ここの実装難度が歴史的に M:N を廃れさせ 1:1 を主流にした。

## 技術詳細 (Technical Details)

### 1. 文脈切り替えの機構とコスト (Context Switch)

文脈切り替えは「走行中タスク T_old の状態を保存し、次タスク T_new の状態を復元して制御を移す」操作。段階ごとに直接コストと間接コストがある。

```
// 例示 (illustrative)。プロセス間文脈切り替えの概念フロー。実コードではない。

1. T_old の CPU レジスタを T_old の kernel stack / task_struct へ退避  ← 直接(軽)
     汎用レジスタ, PC, SP, フラグ。FPU/SIMD は遅延退避(lazy)もある。
2. スケジューラがポリシーに従い T_new を選択 (pick_next_task)         ← 直接
3. アドレス空間切り替え: CR3 (x86) / TTBR (ARM) に T_new のページ
     テーブル基底を書く                                              ← TLB汚染の起点
       → TLB フラッシュ (ASID/PCID 非対応なら全フラッシュ)           ← 間接(重)
4. T_new のレジスタを復元し、T_new の PC へジャンプ                  ← 直接(軽)
5. 以降、T_new の作業セットを L1/L2 へ再充填するキャッシュミスの嵐    ← 間接(最重)
```

- **直接コスト**（レジスタ退避復元 + スケジューラ選択）は数十〜数百ナノ秒オーダ。FPU/SIMD レジスタ群は大きいため、使われるまで退避を遅延する **lazy FPU save** が一般的（`../../hardware/cpu-architecture/simd-vector.md` のレジスタ幅が直接効く）。
- **間接コスト**こそが本質。切り替え後、新タスクは自分のコードとデータでキャッシュを温め直す必要があり、これが数千サイクル＝マイクロ秒級に達しうる。さらにアドレス空間が変わると **TLB** の翻訳が無効化される（`../../hardware/cpu-architecture/virtual-memory-mmu.md`）。
- **TLB フラッシュの緩和**: x86 の **PCID (Process Context ID)** / ARM の **ASID (Address Space ID)** は、TLB エントリにアドレス空間タグを付け、CR3/TTBR を切り替えても全フラッシュを避ける。これにより頻繁な切り替えでも TLB が生き残り、間接コストを大幅に削る。同一プロセス内スレッド切り替えはそもそもアドレス空間不変なのでこの問題自体が生じない。

この間接コストの存在が「切り替えは安くない」という事実の根拠であり、スケジューラが**タイムスライスを短くしすぎない**理由（切り替えオーバヘッドが実仕事を食う）と、**キャッシュ親和性 (affinity)** を保とうとする理由の両方を説明する。

### 2. プリエンプションとタイマ割り込み (Preemption & Timer Interrupt)

協調的マルチタスクでは、走行中タスクが `yield` するまで CPU は奪えない。これでは無限ループ1つでシステムが固まる。**プリエンプティブ・マルチタスク**は外部の強制力を必要とし、それがタイマ割り込みである。

```
// 例示 (illustrative)。周期タイマによるプリエンプションの流れ。

   タイマ(HPET/APIC timer/ARM generic timer) が周期的に割り込み発火
        │  (例: 250Hz〜1000Hz、tickless なら可変)
        ▼
   CPU が現タスクを中断しカーネルの割り込みハンドラへ (特権遷移)
        │   ← ../../hardware/cpu-architecture/interrupts-exceptions-traps.md
        ▼
   scheduler_tick(): 走行中タスクの実行時間を加算、vruntime 更新
        │
        ▼
   need_resched フラグ判定: より相応しいタスクが居るか?
        │ Yes                           │ No
        ▼                               ▼
   割り込みリターン直前に schedule()    そのまま現タスクへリターン
   → 文脈切り替えで別タスクへ
```

- **ティック方式 (tick-based)**: 固定周波数（歴史的に 100/250/1000Hz）でタイマ割り込みを受け、その都度スケジューリング判断をする。周波数が高いほど応答が細かいが、割り込みオーバヘッドと電力消費が増える。
- **ティックレス (tickless / `NO_HZ`)**: アイドル時や単一タスク走行時に周期割り込みを止め、必要な時刻にだけワンショットタイマを仕掛ける。アイドルコアを起こさないため**電力**に有利（`../../hardware/hw-sw-interface/clock-power-reset.md` の C-state 滞在を延ばせる）。
- **カーネルプリエンプション**: ユーザ空間だけでなくカーネル実行中もプリエンプト可能にするか（`PREEMPT` 系設定）でレイテンシ特性が変わる。`PREEMPT_RT` は割り込みハンドラまでスレッド化し、ほぼ全域をプリエンプタブルにしてリアルタイム性を高める。
- システムコール境界（`../../hardware/hw-sw-interface/abi-syscalls.md`）やシステムコール戻り・割り込み戻りは、`need_resched` を確認する自然なチェックポイントになる。

### 3. スケジューリングアルゴリズムの系譜 (Algorithm Lineage)

各方式は前世代の欠陥を埋めるために登場した。

| 方式 | 中核アイデア | 解決したこと | 残った欠陥 |
|---|---|---|---|
| **FCFS/FIFO** | 到着順に最後まで | 単純・飢餓なし | 短ジョブが長ジョブを待つ (convoy effect) |
| **SJF/SRTF** | 最短ジョブ優先 | 平均待ち時間を最小化 | 実行時間が未知・長ジョブが飢える |
| **Round-Robin** | 固定タイムスライスで巡回 | 応答性・プリエンプション導入 | スライス長の調整が困難、優先度なし |
| **Priority** | 優先度の高い順 | 重要タスクを優遇 | 低優先度の**飢餓 (starvation)**、優先度逆転 |
| **MLFQ** | 多段キュー + 動的昇降格 | 対話/バッチを自動判別 | パラメータ調整が職人芸、gaming 可能 |
| **Fair-share** | グループ/ユーザ単位で公平配分 | 利用者間の公平性 | グループ階層の管理コスト |
| **CFS** | 重み付き仮想時間で完全公平 | スライス長問題と優先度を統一 | レイテンシ要求の表現が間接的 |
| **EEVDF** | 仮想時間 + eligible deadline | 公平性とレイテンシを直交化 | 実装の新しさ・調整経験の蓄積中 |

**MLFQ (Multi-Level Feedback Queue)** は歴史的に重要。タスクを複数の優先度キューに置き、CPU を使い切ったタスクを下位キューへ降格、I/O で頻繁に手放す対話的タスクを上位に保つことで、「対話的か計算主体か」を**実行時間という観測量から推定**する。定期的な優先度ブースト (priority boost) で飢餓と gaming（わざと I/O して優先度を保つ攻撃）を緩和する。古い Windows / 旧 Linux O(1) スケジューラの思想的祖先である。

### 4. Linux CFS — vruntime と赤黒木 (Completely Fair Scheduler)

CFS（カーネル 2.6.23〜）は「固定タイムスライス」という概念を捨てた。理想的には N 個の実行可能タスクが各々 1/N の CPU を**連続的に**得る（理想的マルチタスクプロセッサ）。これを近似するため、各タスクの**仮想実行時間 (vruntime)** を追跡する。

```
// 例示 (illustrative)。vruntime の概念。実コードではない。

vruntime += delta_exec * (NICE_0_WEIGHT / task_weight)

  delta_exec    : 実際に走った実時間
  task_weight   : nice値から決まる重み (nice -20 が最重, +19 が最軽)
  ⇒ 重い(高優先)タスクほど vruntime の進みが遅く、より多く走れる
```

- スケジューラは常に**最小 vruntime のタスク**を選ぶ。これを高速に行うため、実行可能タスクを **vruntime をキーとした赤黒木 (red-black tree)** で保持し、最左ノード（最小 vruntime）を O(log N) で取り出す。挿入・削除も O(log N)。
- nice 値は**重み (weight)** に変換され、vruntime の進む速度を変える。優先度は「絶対的な順番」ではなく「公平な取り分の比率」として表現されるため、低優先度タスクも必ずいつかは走る ＝ **飢餓が原理的に起きにくい**。これが Priority 方式の最大の欠陥を消した点。
- タイムスライスは固定ではなく、`sched_latency`（全タスクを1巡させる目標期間）を実行可能タスク数で按分して動的に決まる。タスクが増えるとスライスは縮むが、`min_granularity` で下限を設け、切り替えオーバヘッド（前述の文脈切り替えコスト）が支配的にならないよう抑える。
- 新規・起床タスクの vruntime は、走り続けていたタスクに対して**不当に小さくならない**よう、実行可能キューの最小 vruntime 付近に補正される。これがないと、長く眠っていたタスクが起きた瞬間 vruntime が極小で CPU を独占してしまう。

### 5. EEVDF — CFS の後継 (Earliest Eligible Virtual Deadline First)

CFS は公平だが、「低レイテンシが欲しい」という要求を nice 値で間接的にしか表現できなかった。**EEVDF**（カーネル 6.6 で CFS を置換）は公平性とレイテンシを分離する。

- **eligibility（資格）**: あるタスクが「これまでに得るべきだった公平な取り分」を消費し尽くしていない（lag ≥ 0）うちは eligible。取りすぎたタスクは一時的に ineligible となり、他に追い越させる。これが公平性を保証する。
- **virtual deadline（仮想締切）**: 各タスクに要求スライス（`sched_runtime` 的なリクエスト）に基づく仮想締切を与え、eligible なタスクの中から**最も締切が早いもの**を選ぶ。短いスライスを要求するレイテンシ重視タスクは締切が早くなり、優先的に走れる。
- 結果として、「公平な総取り分」（lag = 0 を目指す）と「いつ走らせるか」（締切）が**直交した2軸**になる。CFS では一体だった2つの目的を分離したことが本質的な改善で、対話的ワークロードの応答性が nice 値の手当てなしに改善する。

### 6. リアルタイムスケジューリング (Real-Time Scheduling)

汎用スケジューラがスループットと平均的公平性を狙うのに対し、リアルタイムは**締切を守る予測可能性**を最優先する。Linux では実時間ポリシーが CFS/EEVDF より常に優先される。

- **`SCHED_FIFO`**: 固定優先度、タイムスライスなし。同優先度内では自発的に手放すまで走り続ける。最高優先度の実時間タスクが CPU を握る。
- **`SCHED_RR`**: `SCHED_FIFO` に固定タイムスライスを足し、同優先度内を round-robin で回す。
- **`SCHED_DEADLINE` (EDF + CBS)**: タスクごとに (実行時間, 締切, 周期) を宣言し、**Earliest Deadline First (EDF)** で最も締切が近いタスクを選ぶ。Constant Bandwidth Server (CBS) が宣言した実行時間を超えるタスクを抑え込み、他タスクへの影響を隔離する。固定優先度より理論上の CPU 利用率上限が高い（単一プロセッサで EDF はスケジュール可能性が利用率 ≤ 100% まで）。

**優先度逆転 (Priority Inversion) と継承 (Inheritance)**:

```
// 例示 (illustrative)。古典的な優先度逆転。

  優先度: H(高) > M(中) > L(低)
  L が共有ロック R を保持中
  H が R を要求 → L の解放待ちでブロック
  ここで M（Rと無関係）が実行可能になると、M が L をプリエンプト
  → L は走れず R を放せない → H は M より高いのに M に間接的に待たされる(逆転)
```

- **優先度継承 (priority inheritance)**: ロック保持者 L の優先度を、待っている最高優先度タスク H まで一時的に引き上げる。これで M は L をプリエンプトできず、L は素早く R を放し、H が進める。Linux の `PI-futex` / RT-mutex が実装。
- **優先度上限 (priority ceiling)**: ロックに「そのロックを取りうる最高優先度」を割り当て、保持中は保持者をその上限まで引き上げる代替策。
- 1997年の Mars Pathfinder のリセット連発は、この優先度逆転（VxWorks 上で発生）と継承の有効化で解決した有名事例。リアルタイム性が「平均」ではなく「最悪ケース」の問題であることを示す。

### 7. マルチコアでの負荷分散と親和性 (Load Balancing & Affinity)

単一のグローバル実行可能キューは、全コアが1つのロックとキャッシュラインを奪い合うため**シリアライズ点**になる（`../../hardware/cpu-architecture/cache-coherence.md` のホットライン競合と同じ構造）。よって現代カーネルは **per-CPU runqueue** を持つ。その代償が負荷分散と親和性の管理である。

- **負荷分散 (load balancing)**: コア間でタスク数（正確には負荷の重み）が偏ると、忙しいコアから暇なコアへタスクを移す (migration)。Linux は **scheduling domain** という階層（SMT 兄弟 → 同一 LLC コア群 → 同一 NUMA ノード → ノード跨ぎ）でこれを行い、**階層が深いほど移動の閾値を高くする**。遠いコアへの移動ほどキャッシュ・TLB を捨てるコストが高いからである。
- **CPU 親和性 (affinity)**: `sched_setaffinity(2)` / `taskset` でタスクを特定コア集合に固定する。キャッシュの温かさを保ち、リアルタイムタスクを専用コアに隔離する。**ソフト親和性**（スケジューラが極力同じコアに戻そうとする傾向）と、**ハード親和性**（明示的なマスク）がある。
- **SMT (Hyper-Threading) の考慮**: 1物理コアの2論理コアは実行資源を共有するため、2タスクを別物理コアに散らす方が（負荷が低いうちは）速い。スケジューリングドメインの最下層が SMT 兄弟を表し、まず兄弟を避けて分散する。

### 8. NUMA 親和性 (NUMA Awareness)

複数ソケット構成ではメモリがノードに分散し、リモートノードのメモリアクセスはローカルより遅い（`../../hardware/cpu-architecture/cache-coherence.md` の ccNUMA 注を参照）。スケジューラはタスクを**そのデータが在るノード**に置こうとする。

- **データとスレッドの同居**: タスクのメモリが NUMA ノード0に first-touch で確保されているなら、そのタスクをノード0のコアで走らせる方がメモリレイテンシが小さい。リモートノードへ migration すると、以後すべてのメモリアクセスが跨ノードになり高くつく。
- **Linux の NUMA balancing (`AutoNUMA`)**: ページに定期的にアクセス障害 (NUMA hinting fault) を仕掛けて「どのノードのどのタスクがそのページに触っているか」を観測し、(a) ページをタスクのノードへ移す (page migration)、(b) タスクをページのノードへ移す、のいずれかで同居を改善する。前述の負荷分散とは**ローカリティを優先するか負荷均等を優先するか**で時に対立し、スケジューラはその折衷を取る。
- NUMA を無視した素朴な負荷分散は、ローカリティを壊してメモリレイテンシで損をしうる。「空いているコアへ移す」が常に正解ではない。

### 9. スケジューリングと電力 (Scheduling & Power)

スケジューリングは性能だけでなく消費電力も決める（`../../hardware/hw-sw-interface/clock-power-reset.md`）。

- **タスク集約 vs 分散**: 軽負荷時に少数コアへタスクを**集約 (race-to-idle / consolidation)** し、空いたコアを深い C-state（クロック停止・電源遮断）に落とすと省電力。逆に均等分散は各コアを浅いアイドルに留め、電力を食う。どちらが得かは負荷と復帰レイテンシのトレードオフ。
- **DVFS との協調**: 走行中タスクの要求に応じて周波数・電圧を上下させる **DVFS** と、スケジューラの配置判断は連動する。Linux の **`schedutil`** ガバナは、スケジューラが見積もる CPU 利用率 (PELT: Per-Entity Load Tracking) を直接周波数選択に渡し、別々の推定をやめて一元化した。
- **非対称コア (big.LITTLE / P-core+E-core)**: 高性能コアと高効率コアが混在する構成では、スケジューラがタスクの重さに応じてコア種別を選ぶ（**Energy Aware Scheduling, EAS** / `SCHED_UTIL`）。軽い対話タスクは E-core、重い計算は P-core、というエネルギー効率の見積もりに基づく配置を行う。
- ティックレス（前述）は、アイドルコアを周期割り込みで起こさないことで C-state 滞在を延ばす、スケジューリングと電力の接点そのものである。

## セキュリティ考察 (Security Considerations)

スケジューリングは資源配分機構であると同時に、その配分の**観測可能性**と**枯渇可能性**がセキュリティ問題を生む。

- **スケジューラ・サイドチャネル / タイミングリーク**: 攻撃者プロセスは自分の走行が遅延した量から、同居タスクの活動を推定できる。SMT で物理コアを共有する場合、被害者の実行ユニット占有が攻撃者のスループットに直接漏れる（PortSmash 系）。文脈切り替え頻度・キャッシュ温度（`../../hardware/cpu-architecture/cache-coherence.md` のクロスコア・タイミングチャネル）と組み合わさると、秘密依存の分岐や実行時間が漏洩経路になる。緩和は**コア・スケジューリング (core scheduling)**（信頼境界を跨ぐタスクを同一物理コアの SMT 兄弟に同時配置しない）と SMT 無効化、定数時間実装。

- **cgroup / CPU quota による隔離**: `cgroup v2` の `cpu.max`（周期内の上限実行時間）と `cpu.weight`（相対配分）は、テナント／コンテナごとに CPU を会計し、1グループが全体を食い潰すのを防ぐ。CFS bandwidth controller が、quota を使い切ったグループを周期境界までスロットルする。これがコンテナ環境でのマルチテナント隔離の土台である。ただし quota の周期境界でのバースト的スロットリングがレイテンシ尖りを生む副作用は既知の注意点。

- **fork bomb と資源制限**: `:(){ :|:& };:` のように再帰的にプロセスを増殖させる **fork bomb** は、PID とタスク構造体・実行可能キューを枯渇させ、正規タスクが走れなくする可用性攻撃（DoS）である。防御は多層: `RLIMIT_NPROC`（ユーザあたりプロセス数上限）、`cgroup` の `pids.max`（グループあたりプロセス数）、`kernel.threads-max`。いずれも「スケジューリング以前に、実行主体の生成数を会計し上限を課す」アプローチで、スケジューラ自体ではなく**入口での資源制限**が要であることを示す。

- **優先度を悪用した飢餓 / DoS**: 攻撃者が高優先度（実時間ポリシー）を取得できると、`SCHED_FIFO` の最高優先度で無限ループを回し他タスクを完全に飢えさせうる。Linux は非特権ユーザの実時間優先度取得を `RLIMIT_RTPRIO` で制限し、`sched_rt_runtime_us` で実時間タスクの総 CPU 占有に上限（既定で 1秒中 0.95秒）を課して、暴走実時間タスクからシステムを救えるようにしている。

- **公平性 gaming**: MLFQ 系で「短時間 I/O を挟んで優先度を高く保つ」攻撃や、CFS で sleep/wake を繰り返して vruntime を低く保とうとする試みは、観測ベースの優先度推定に共通する弱点。前述の vruntime 起床補正や優先度ブーストは、性能機構であると同時にこうした gaming への防御でもある。

教訓: スケジューラは「誰がいつどれだけ CPU を得るか」を決めるため、その決定は (1) 観測されればサイドチャネル、(2) 偏らせれば DoS になる。隔離（cgroup・affinity・core scheduling）と会計（quota・rlimit）が、性能と引き換えに安全を買う基本手段である。

## 結論 (Conclusions)

- スケジューリングは「有限コアの時間多重化」によって並行実行の錯覚を作る OS の中核機構であり、ポリシー（誰を走らせるか）と機構（文脈切り替え）に分離して理解すべきである。
- プロセスとスレッドの区別は実装で連続化し（Linux の `task_struct`/`clone`）、M:N は実装難度ゆえ 1:1 が汎用 OS の標準に落ち着いた。
- 文脈切り替えの真のコストはレジスタ退避ではなく、キャッシュ冷却と TLB フラッシュの間接コストであり、PCID/ASID・親和性・タイムスライス下限はすべてこのコストを抑える工夫である（`../../hardware/cpu-architecture/virtual-memory-mmu.md`、`../../hardware/cpu-architecture/cache-coherence.md`）。
- プリエンプションはタイマ割り込みという外部強制で初めて成立し（`../../hardware/cpu-architecture/interrupts-exceptions-traps.md`）、ティックレス化を通じて電力（`../../hardware/hw-sw-interface/clock-power-reset.md`）とも接続する。
- アルゴリズムは「固定スライス＋優先度」の欠陥（スライス調整難・飢餓）を、重み付き仮想時間（CFS の vruntime + 赤黒木）で乗り越え、EEVDF が公平性とレイテンシを直交化した。リアルタイムは平均でなく最悪ケースの問題であり、EDF と優先度継承が予測可能性を支える。
- マルチコアではスケジューリングが配置問題に拡張され、負荷分散・親和性・NUMA ローカリティ・電力が多目的に競合する。セキュリティ面では、配分の観測可能性（サイドチャネル）と枯渇可能性（fork bomb/飢餓）が脅威となり、隔離と会計（cgroup quota・rlimit・core scheduling）が防御の要である。

### 次の接続先 (Related Topics)

- `../../hardware/cpu-architecture/interrupts-exceptions-traps.md` — タイマ割り込みと特権遷移。プリエンプションとシステムコール戻りのスケジューリング契機を提供する。
- `../../hardware/cpu-architecture/virtual-memory-mmu.md` — ページテーブルと TLB。文脈切り替えの間接コストと PCID/ASID による緩和の前提。
- `../../hardware/cpu-architecture/cache-coherence.md` — NUMA / ccNUMA とクロスコア・タイミングチャネル。負荷分散のローカリティ判断とサイドチャネルの基盤。
- `../../hardware/hw-sw-interface/abi-syscalls.md` — システムコール境界。スケジューリングのチェックポイントとブロッキング遷移の発生点。
- `../../hardware/hw-sw-interface/clock-power-reset.md` — クロック・電力・C-state/DVFS。ティックレスと省電力スケジューリングの接点。

## 参考文献 (References)

- Silberschatz, A., Galvin, P. B., & Gagne, G. (2018). *Operating System Concepts* (10th ed.). Wiley. — プロセス状態機械・スケジューリングアルゴリズムの古典的整理。
- Arpaci-Dusseau, R. H., & Arpaci-Dusseau, A. C. (2018). *Operating Systems: Three Easy Pieces*. Arpaci-Dusseau Books. — MLFQ・比例配分・CFS の教育的解説。[Online](https://pages.cs.wisc.edu/~remzi/OSTEP/) — Accessed: 2026-06-29.
- Liu, C. L., & Layland, J. W. (1973). Scheduling Algorithms for Multiprogramming in a Hard-Real-Time Environment. *Journal of the ACM*, 20(1), 46–61. DOI:10.1145/321738.321743 — RM/EDF とスケジュール可能性の基礎。
- Sha, L., Rajkumar, R., & Lehoczky, J. P. (1990). Priority Inheritance Protocols: An Approach to Real-Time Synchronization. *IEEE Transactions on Computers*, 39(9), 1175–1185. DOI:10.1109/12.57058 — 優先度継承・優先度上限の定式化。
- Stoica, I., et al. (1996). A Proportional Share Resource Allocation Algorithm for Real-Time, Time-Shared Systems. *Proc. RTSS*. — EEVDF（Earliest Eligible Virtual Deadline First）の原論文。
- [Linux kernel documentation — CFS Scheduler](https://docs.kernel.org/scheduler/sched-design-CFS.html) — Accessed: 2026-06-29. vruntime と赤黒木の設計。
- [Linux kernel documentation — Deadline Task Scheduling (SCHED_DEADLINE)](https://docs.kernel.org/scheduler/sched-deadline.html) — Accessed: 2026-06-29. EDF + CBS の実装。
- [An EEVDF CPU scheduler for Linux (LWN.net)](https://lwn.net/Articles/925371/) — Accessed: 2026-06-29. CFS から EEVDF への移行解説。
- [Core scheduling (Linux kernel documentation)](https://docs.kernel.org/admin-guide/hw-vuln/core-scheduling.html) — Accessed: 2026-06-29. SMT サイドチャネル緩和としてのコア・スケジューリング。
- [Control Group v2 — CPU controller (Linux kernel documentation)](https://docs.kernel.org/admin-guide/cgroup-v2.html) — Accessed: 2026-06-29. `cpu.max` / `cpu.weight` による CPU 会計と隔離。
- Reeves, G. E. (1997). *What really happened on Mars?* — Mars Pathfinder の優先度逆転事例の一次解説。[Online](http://www.cs.cornell.edu/courses/cs614/1999sp/papers/pathfinder.html) — Accessed: 2026-06-29.
