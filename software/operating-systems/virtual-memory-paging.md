# OS の仮想記憶とページング (Virtual Memory & Paging: The OS Side)

## 概要 (Overview)

[`../../hardware/cpu/virtual-memory-mmu.md`](../../hardware/cpu/virtual-memory-mmu.md) は、アドレス変換の**機構**——ページテーブル・TLB・ページウォーク——を扱った。本ドキュメントはその真上、**OS がこの機構を使って何をするか**を主題とする: プロセスにアドレス空間という幻想を与え、物理メモリをファイルキャッシュと分け合い、足りなければ回収し、それでも足りなければプロセスを殺す——**ポリシと機構の層**である。

鍵になる転換は、ページフォルトを「エラー」ではなく **OS への主要な入口**として見ること。`fork` が一瞬で返るのも、`mmap` した 1TB のファイルが即座に「見える」のも、`malloc` が実メモリを消費しないのも、すべて「**実際に触られるまで何もしない (lazy)**」戦略の帰結であり、その決済がページフォルトで行われる。

本ドキュメントは、(1) アドレス空間の管理 (VMA)、(2) ページの供給——demand paging・COW・ページキャッシュ、(3) ページの回収——LRU 近似・writeback・swap、(4) 破綻時の振る舞い——overcommit と OOM killer、を Linux を参照実装として整理する。分析深度は Level 3。

## 方法論 (Methodology)

- 仮想記憶を「**供給**（フォルトでページを与える）・**共有**（ページキャッシュ・COW）・**回収**（LRU 近似・writeback・swap）」の3機構に分解し、それぞれの機構とポリシを分離して示す。
- ページフォルトの分類（minor/major、anon/file-backed）を軸に、コードパスと性能・永続性への含意を対応づける。
- 数値は桁感 (order of magnitude) を示すものとし、Linux のバージョン・設定で変わる点は明示する。
- 下の層（ページテーブル・TLB）は既存ドキュメントに委ね、本書は OS のデータ構造と判断に集中する。

## 発見事項 (Findings)

1. **仮想記憶の実体は「約束の台帳」である。** `mmap`/`brk` が更新するのは VMA（仮想アドレス範囲の予約リスト）だけで、物理ページもページテーブルエントリも作らない。**約束が守られるのは触られた瞬間**——ページフォルトが VMA を照会し、正当なら物理ページを用立て、不当なら SIGSEGV を送る。「segfault とはVMA にない番地への参照」である。
2. **`fork` の速さは COW の産物。** 親子はページテーブルを共有せずとも物理ページを共有し、全ページを**書き込み禁止**にしておく。どちらかが書いた瞬間のフォルトで初めて複製する (copy-on-write)。「fork してすぐ exec」する典型パターンでは複製がほぼ発生しない。
3. **ページキャッシュはファイル I/O と仮想記憶の合流点である。** `read`/`write` も `mmap` も同じページキャッシュを経由する（unified page cache）。「空きメモリ」に見えない大半のメモリはこのキャッシュであり、**回収可能なので実質空き**——`free` コマンドの available 列の意味はこれ。
4. **正確な LRU は高すぎるので、全ては近似である。** 全メモリアクセスの追跡は不可能なため、OS はハードの参照ビットを周期的に採取して「最近使われたか」を粗く推定する。Linux の active/inactive 2 リストから世代別の MGLRU への進化は、この**近似の精度とコストの改善史**である。
5. **dirty page の書き戻しが、性能と永続性の境界を作る。** `write` の完了はキャッシュへの書き込みに過ぎず、永続化は writeback（背景・閾値駆動）か `fsync`（明示）まで起きない。データベースが WAL と `fsync` に神経質になる理由（[`../databases/storage-engines.md`](../databases/storage-engines.md)）は、この層の遅延書き込みにある。
6. **`malloc` の成功はメモリの存在を意味しない (overcommit)。** Linux は既定で約束を過剰発行し、決済（初アクセス）時に物理が尽きていれば **OOM killer** がプロセスを選んで殺す。「確保できたのに使ったら死ぬ」は仕様であり、データベースやコンテナ運用はこの前提で設計されている。

## アーキテクチャ (Architecture)

### プロセスのアドレス空間と VMA

```
 プロセスの仮想アドレス空間 (x86-64 Linux の典型)
 ┌────────────┬──────┬──────────┬─────────┬──────────────┬───────┐
 │ text/rodata │ data │  heap →  │ mmap 領域 │ ← stack      │ kernel │
 │ (ELF, 共有) │ (RW) │ (brk)    │ (ファイル/匿名/共有ライブラリ) │ (不可視) │
 └────────────┴──────┴──────────┴─────────┴──────────────┴───────┘
   各範囲 = VMA (vm_area_struct): [start, end) × 権限(rwx) × 背後 (file or anon)
```

- VMA は「この範囲は正当で、背後は何か（ファイルか匿名メモリか）、権限は何か」を記述する台帳。ページテーブル（[`../../hardware/cpu/virtual-memory-mmu.md`](../../hardware/cpu/virtual-memory-mmu.md)）はその**キャッシュされた実現**にすぎず、いつでも VMA から再構築できる。
- `mmap` は VMA を1つ挿入するだけの O(log n) 操作。だから巨大ファイルのマップが即座に返る。

### ページフォルトの分類と処理パス

```
                     ページフォルト (MMU が例外を上げる)
                                  │
              VMA になし ────────▶ SIGSEGV (本物の不正参照)
              VMA にあり・権限違反 ─▶ 書込 to RO: COW 判定 or SIGSEGV
                                  │ VMA にあり・PTE 未在
                 ┌────────────────┴───────────────┐
             匿名 (anon)                      ファイル背後 (file-backed)
                 │                                │
        zero page / 新規ページ割当        ページキャッシュにある? ── yes → PTE を張るだけ
        (minor fault)                            │ no                 (minor fault)
                                          ディスクから読む (major fault)
                                          — ミリ秒級。readahead で先読み
```

- **minor fault**（物理ページは既にある、PTE を張るだけ）はマイクロ秒級で日常茶飯事。**major fault**（ディスク I/O を伴う）は数桁遅く、性能問題の指標になる（`ps` の MINFL/MAJFL、`perf stat` の page-faults）。

### メモリの全体像 — 誰が物理ページを持っているか

```
 物理メモリ
 ├── カーネル自身 (固定・回収不可)
 ├── 匿名ページ (anon): ヒープ・スタック。捨てられない → 逃がすには swap が要る
 ├── ページキャッシュ (file): clean なら即捨てられる / dirty は書き戻してから
 └── 空き
      回収の優先順位はこの「捨てるコスト」の差から決まる
```

## 技術詳細 (Technical Details)

### 1. demand paging と COW — 遅延の経済学

- **demand paging**: 実行ファイルのコードも `mmap` 領域も、初アクセスまでロードしない。起動が速く、触らない部分のメモリを一切消費しない。代償は「初回アクセスの遅延が不定」——リアルタイム用途では `mlock`/`mlockall` で先に決済しておく（[`process-thread-scheduling.md`](process-thread-scheduling.md) のリアルタイムの文脈）。
- **COW の連鎖**: `fork` 後の親子、`mmap(MAP_PRIVATE)` のファイル、重複排除 (KSM) ——「共有しておいて、書く者が複製コストを払う」は仮想記憶の至る所で使われる。**読み共有はタダ、書き込みがコスト**という非対称は、キャッシュコヒーレンス（[`../../hardware/cpu/cache-coherence.md`](../../hardware/cpu/cache-coherence.md)）と同じ構造で層を変えて再現している。

### 2. ページキャッシュ — read/write と mmap の統一

- `read(fd, buf, n)` = ページキャッシュへ読み込み（なければディスクから）→ ユーザバッファへ**コピー**。`mmap` = 同じキャッシュページを**直接マップ**（コピーなし、ただしフォルトと TLB のコスト）。どちらが速いかはアクセスパターン次第で、「mmap は常に速い」は俗説である。
- **readahead**: 逐次アクセスを検出すると先読み窓を広げ、major fault を隠す。ランダムアクセスには効かない——ストレージエンジンが自前のバッファプール＋`O_DIRECT` を選ぶ理由の一つ（[`../databases/storage-engines.md`](../databases/storage-engines.md) の「二重キャッシュ回避」）。
- **dirty と writeback**: 書き込みはページを dirty にするだけで返る。書き戻しは (a) 背景スレッドが古い dirty を刷く、(b) dirty 比率が閾値（`vm.dirty_background_ratio` → 非同期、`vm.dirty_ratio` → 書き手を同期的に待たせる）を超える、(c) `fsync`/`fdatasync`、で起きる。**大量書き込みが突然固まる**現象の多くは (b) の同期スロットリングである。

### 3. 回収 — LRU 近似の系譜

物理が逼迫すると、kswapd（背景）または直接回収（アロケータが自ら、レイテンシに直撃）が動く。

- **古典: active/inactive 2リスト**: 新規ページは inactive へ。参照ビット（ハードが PTE に立てる）を採取して昇格/降格し、inactive の末尾から回収する。「2回触られたら active」という**再参照の証拠**で、一度きりの大量読み（バックアップ等）がワーキングセットを追い出すのを防ぐ。
- **MGLRU (Multi-Gen LRU, Linux 6.1〜)**: 2リストを複数世代に細分し、ページテーブル走査ベースの採取で精度とコストを改善。回収品質はデスクトップの体感からサーバのテールレイテンシまで直結するため、この層の改良は今も現役の開発対象である。
- **swap**: 匿名ページの逃がし先。「swap は悪」ではなく、**触られない匿名ページを追い出してキャッシュに物理を回す**ための弁 (`vm.swappiness` が anon vs file の回収比率を制御)。swap なし運用は「匿名ページを一切回収できない」構成であり、OOM を近づける側面もある。zswap/zram は圧縮でこのコストを軽減する。
- 理論的背景は Denning の**ワーキングセットモデル** (1968): プロセスが直近 τ で触ったページ集合を保てばスラッシングしない。実装はどれも、これを追跡可能なコストで近似する試みである。

### 4. overcommit と OOM killer — 約束の破綻処理

- Linux の既定 (`vm.overcommit_memory=0`) は、確保要求を物理＋swap の合計を超えて受け付ける。根拠は「ほとんどのプログラムは確保した分を全部触らない」（fork の COW、疎な配列、余裕を持った malloc）。
- 決済不能——フォルト時に物理も swap も尽きた——になると **OOM killer** が「メモリ使用量・oom_score_adj」等のスコアでプロセスを選んで SIGKILL する。**殺されるのは犯人ではなく、殺すと一番楽になるプロセス**である点に注意。
- cgroup (memory controller) はこの破綻を**コンテナ単位に閉じ込める**: 制限超過はそのグループ内の OOM で処理され、隣人に波及しない。Kubernetes の memory limit の実体。
- 厳格会計 (`overcommit_memory=2`) を選べば malloc が正直に失敗するが、fork ヘビーなワークロードは COW の見かけ上の予約で先に破綻する——**どちらの嘘を選ぶか**という設計判断である。

### 5. ヒュージページの OS 側 — THP と khugepaged

TLB カバレッジの問題（[`../../hardware/cpu/virtual-memory-mmu.md`](../../hardware/cpu/virtual-memory-mmu.md)）への OS 側の答えが **THP (Transparent Huge Pages)**: フォルト時に 2MB で割り当てる、または khugepaged が背景で 4KB×512 を昇格させる。

- 効能は TLB ミスとページウォークの削減。代償は (a) 内部断片化（2MB の一部しか使わない）、(b) 昇格・分割・コンパクションの CPU/レイテンシ、(c) 過去の実装では回収との相互作用によるレイテンシスパイク。データベースが「THP を無効にせよ」と長く案内してきたのはこの (b)(c) が原因で、**「透過」な最適化がワークロードによっては負ける**実例として覚えておく価値がある。

### 6. 観測の入口 (Observability)

| 疑問 | 道具 |
|---|---|
| フォルトはどれだけ起きているか | `perf stat -e page-faults,major-faults`、`ps -o min_flt,maj_flt` |
| メモリは誰が持っているか | `/proc/meminfo`（Cached/Dirty/AnonPages）、`smem`、`/proc/<pid>/smaps` |
| 回収は逼迫しているか | `vmstat`（si/so, kswapd）、PSI (`/proc/pressure/memory`) |
| キャッシュに何が載っているか | `mincore(2)`、`fincore` |

PSI (Pressure Stall Information) は「メモリ待ちで止まった時間の割合」を直接示す指標で、回収の逼迫を OOM の手前で検知できる——モダンな容量管理の起点。

## セキュリティ考察 (Security Considerations)

- **Dirty COW (CVE-2016-5195)**: COW の書き込み処理と `madvise` の競合により、**読み取り専用マップへの書き込みが本体に到達**する競合状態バグ。読み取り専用で root 所有のファイル（setuid バイナリ等）を書き換えられ、権限昇格が成立した。仮想記憶の中核パスの**データ競合**が直ちに最重要脆弱性になる実例であり、並行性の正しさ（[`../concurrency/memory-models-synchronization.md`](../concurrency/memory-models-synchronization.md)）と仮想記憶が交差する地点。
- **ページキャッシュはサイドチャネルになる**: キャッシュ在否の時間差（`mincore` や計時で観測）から、他プロセスがどのファイルのどこを触ったかが漏れる。共有＝観測面という構造はハードのキャッシュ攻撃（[`../../hardware/cpu/cache-memory-hierarchy.md`](../../hardware/cpu/cache-memory-hierarchy.md)）と同型で、`mincore` の挙動制限など OS 側の対処が続いている。
- **zero page 保証**: 新規ページは必ずゼロ埋めで渡される。これが破れると他プロセスの残骸（鍵・パスワード）が読める——「未初期化メモリの開示」はカーネルにとって恒常的な脆弱性クラスである。ユーザ空間でも、秘密を置いたページは `mlock`（swap への漏出防止）と明示ゼロ化 (`explicit_bzero`) で扱う。
- **OOM は可用性の攻撃面**: メモリを食い尽くす入力（解凍爆弾、巨大リクエスト）で OOM killer に無関係なプロセスを殺させうる。cgroup による隔離と入力サイズの上限が防御になる。

## 結論 (Conclusions)

- OS の仮想記憶は「約束 (VMA) と決済 (フォルト)」のシステムである。fork・mmap・malloc の速さはすべて遅延評価の産物であり、その請求書はページフォルトとして届く。
- 物理メモリの実態は匿名ページとページキャッシュの綱引きであり、回収 (LRU 近似・writeback・swap) がその調停者である。性能問題の多く——起動の遅さ、書き込みの突然の停止、テールレイテンシ——はこの層の挙動で説明できる。
- overcommit と OOM killer は「約束の過剰発行」という設計判断の帰結であり、cgroup がその破綻を隔離する。コンテナ時代のメモリ運用はこの前提の上に立つ。
- 仮想記憶は分離機構であると同時に共有機構であり、共有は常に観測面（サイドチャネル）と競合面（Dirty COW）を作る。正しさ・性能・セキュリティがページという同じ単位の上で交差する、OS の中心的な層である。

### 次の接続先 (Related Topics)

- [`../../hardware/cpu/virtual-memory-mmu.md`](../../hardware/cpu/virtual-memory-mmu.md) — 本書の真下。ページテーブル・TLB・ページウォークの機構。
- [`process-thread-scheduling.md`](process-thread-scheduling.md) — もう一つの主資源 (CPU) の多重化。直接回収はスケジューリングレイテンシに化ける。
- [`../databases/storage-engines.md`](../databases/storage-engines.md) — ページキャッシュと自前バッファプールの緊張関係、fsync の意味論。
- [`../concurrency/memory-models-synchronization.md`](../concurrency/memory-models-synchronization.md) — Dirty COW 型のカーネル内競合の言語側の対応物。
- [`../../hardware/hw-sw-interface/storage-nvm-interface.md`](../../hardware/hw-sw-interface/storage-nvm-interface.md) — writeback の着地先。FLUSH/FUA と永続性。

## 参考文献 (References)

**一次資料 (Primary — 論文・公式ドキュメント)**

- Denning, P. J. (1968). The Working Set Model for Program Behavior. *Communications of the ACM*, 11(5), 323–333. DOI: 10.1145/363095.363141 — ワーキングセット・スラッシングの理論的原典。
- Linux Kernel Documentation, *Memory Management* — [Concepts overview](https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html)、[Multi-Gen LRU](https://www.kernel.org/doc/html/latest/admin-guide/mm/multigen_lru.html)、[overcommit accounting](https://www.kernel.org/doc/html/latest/mm/overcommit-accounting.html) — Accessed: 2026-07-02。
- CVE-2016-5195 (Dirty COW). — [dirtycow.ninja](https://dirtycow.ninja/) — Accessed: 2026-07-02。COW 競合による権限昇格の一次情報。

**二次資料 (Secondary — 教科書・解説)**

- Arpaci-Dusseau, R. H., & Arpaci-Dusseau, A. C. *Operating Systems: Three Easy Pieces* — 仮想記憶・スワップ・ページ置換の章。 — [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/) — Accessed: 2026-07-02。
- Gorman, M. (2004). *Understanding the Linux Virtual Memory Manager*. Prentice Hall. — [kernel.org 公開版](https://www.kernel.org/doc/gorman/) — Accessed: 2026-07-02。Linux VM のデータ構造の古典的詳説。
- Corbet, J. (2022). Merging the multi-generational LRU. *LWN.net*. — [lwn.net](https://lwn.net/Articles/894859/) — Accessed: 2026-07-02。MGLRU の設計と経緯。
