# 分散合意 (Consensus: Paxos, Raft & Replicated State Machines)

## 概要 (Overview)

[`consistency-models.md`](consistency-models.md) は「複製されたデータが**何を保証するか**」を扱った。本ドキュメントはその実現手段の核心——**合意 (consensus)**、すなわち故障するノード群が**単一の値（実際には単一のログ）について不可逆に一致する**問題を扱う。

合意が解ければ、**複製状態機械 (RSM: Replicated State Machine)** が作れる: 全ノードが同じ操作ログに合意し、決定的な状態機械へ同じ順で適用すれば、全ノードは同じ状態を辿る。線形化可能なデータベース、分散ロック、設定ストア (etcd, ZooKeeper) は、すべてこの一つの還元の上に立つ。

本ドキュメントは、(1) なぜ合意が難しいか（FLP 不可能性）、(2) Paxos と Raft を「クォーラム交差＋単調な番号」という**同じ骨格の異なる提示**として読み解き、(3) ログ複製・リーダ交代・再構成という実務の本体、(4) 実システム (etcd / ZooKeeper / Spanner) での姿を整理する。分析深度は Level 3。

## 方法論 (Methodology)

- **safety（悪いことが起きない: 二つの値が決まらない）と liveness（良いことがいつか起きる: いつかは決まる）を厳密に分離**する。実用プロトコルの設計は一貫して「safety は無条件・liveness は条件付き」である。
- FLP 不可能性を出発点に「何を諦めれば解けるか」を特定し、各プロトコルをその選択として位置づける。
- Paxos (single-decree → Multi-Paxos) と Raft を、同じ不変条件（クォーラム交差、番号の単調性）の実装違いとして対応づける。
- 一次資料（Lamport、Ongaro & Ousterhout、Liskov らの原論文）に基づき、実装は etcd/ZooKeeper のドキュメント・検証事例で裏づける。

## 発見事項 (Findings)

1. **非同期システムでは合意は決定的には解けない (FLP)。** メッセージ遅延に上限がなく1台でも停止しうるなら、必ず終わる決定的な合意アルゴリズムは存在しない。実用系はこれを、**部分同期の仮定（タイムアウト）・故障検出器・ランダム化**のいずれかで回避する——そして遅延が乱れたときに失うのは liveness だけで、**safety は決して失わない**ように作る。
2. **核はクォーラム交差 (quorum intersection)。** 過半数集合はどの二つも必ず交わる。「値を決めるにも、新しいリーダが過去を学ぶにも過半数が要る」なら、**決まった値は必ず次代に引き継がれる**。Paxos の prepare も Raft の投票制限も、この一行の具体化である。
3. **Paxos と Raft は同じ骨格の別の提示である。** どちらも「単調増加する番号（ballot / term）＋2フェーズ（過去を学ぶ→値を固定する）＋過半数」。Raft の新規性は理論ではなく**分解の仕方**——強いリーダ・連続したログ・明確な状態遷移で、理解と実装の状態空間を刈り込んだ点にある（論文の設計目標自体が understandability）。
4. **合意の単位は「値」ではなく「ログ」である。** 実システムが欲しいのは1回の合意ではなく操作列。single-decree Paxos をスロットごとに走らせる Multi-Paxos は、安定リーダの下でフェーズ1を償却し、定常状態を「リーダがログを追記し過半数が確認する」だけにする——これは Raft の定常状態と同じ形。
5. **リーダは性能の道具であり、safety の前提ではない。** プロトコルの本体はむしろ**リーダ交代 (view change / election)** にある: 新リーダが「決まったかもしれない値」を確実に引き継ぐ部分が正しさの核心で、コード量も難しさもここに集中する。
6. **再構成 (membership change) が最難関。** ノード集合そのものを変えると「過半数」の定義が変わり、新旧の過半数が交わらない瞬間ができうる。joint consensus（新旧両方の過半数を要求）や1台ずつの変更で交差を保つ。
7. **クラッシュ故障と Byzantine 故障は別世界。** 本書のプロトコルは「止まるが嘘はつかない」故障モデル (crash fault) 上に立つ。嘘をつくノードを許すなら 3f+1 と署名付きプロトコル (PBFT 系) が要る——航空宇宙の冗長計算機（[`../../engineering/aerospace/avionics-gnc.md`](../../engineering/aerospace/avionics-gnc.md)）が実装してきたのと同じ理論の系譜である。

## アーキテクチャ (Architecture)

### 複製状態機械 (Replicated State Machine)

```
   クライアント ──▶ リーダ
                    │ ①操作をログに追記し、フォロワへ複製
                    ▼
     [ログ]  op1 op2 op3 op4 …   ←── 全ノードでこの「順序」に合意する
                    │ ②過半数が書けたらコミット
                    ▼
   [状態機械] 決定的に順次適用 → 全ノードが同じ状態を辿る
```

- 合意プロトコルが保証するのは**ログの一致**だけ。状態の一致は「決定的な状態機械に同順で適用する」ことから従う。
- クライアントへの応答は**コミット後**（過半数に複製された後）に返す。ここが [`consistency-models.md`](consistency-models.md) の線形化可能性の実装点になる。

### Raft の状態遷移 — 3つの役割

```
                    タイムアウト:            過半数の票:
                    リーダ不在を疑う          リーダに就任
   ┌──────────┐ ──────────────▶ ┌───────────┐ ─────────────▶ ┌────────┐
   │ Follower │                 │ Candidate │                │ Leader │
   └──────────┘ ◀────────────── └───────────┘                └────────┘
        ▲        より大きい term を観測（自分は古い）                │
        └───────────────────────────────────────────────────────────┘
```

- **term（任期）**: 単調増加する論理時刻。全メッセージに乗り、「より大きい term を見たら自分は古い」という一方向の規則が、脳の分裂（同一 term に2リーダ）を防ぐ。1つの term で投票は1回だけ——**過半数の票は交わるので、同一 term のリーダは高々1人**。

### プロトコルの2フェーズ構造（Paxos と Raft の対応）

| 骨格 | Paxos (single-decree) | Raft |
|---|---|---|
| 単調な番号 | ballot number | term |
| フェーズ1: 過去を学ぶ | prepare/promise — 過半数から「既に accept した値」を収集 | 選挙 — ただし**投票制限**（自分よりログが新しい候補にしか投票しない）で「学ぶ」を投票に埋め込む |
| フェーズ2: 値を固定する | accept/accepted — 学んだ最大の値を過半数に書く | AppendEntries — リーダのログを過半数に複製 |
| 決定の条件 | 過半数が同一 ballot で accepted | 過半数に複製（＋現 term の縛り、後述） |

## 技術詳細 (Technical Details)

### 1. FLP 不可能性 — 何を諦めるか

Fischer–Lynch–Paterson (1985) の結果: **完全に非同期なシステムでは、1プロセスの停止故障を許すだけで、常に停止する決定的な合意アルゴリズムは存在しない**。直観は「遅いだけのノードと死んだノードを、有限時間では区別できない」——待てば safety が守れるが決まらず、待たなければ二重決定の危険が生じる。

実用系の回避は3系統ある。

- **部分同期 (partial synchrony)**: 「いつかはタイムアウトが正しく働く」と仮定する。Paxos/Raft はこれ——ネットワークが荒れている間はリーダ選挙が空転する（liveness 喪失）が、**その間も二つの値が決まることはない**（safety 維持）。
- **故障検出器**: タイムアウトを抽象化した理論的枠組み。
- **ランダム化**: 期待有限時間で停止する確率的アルゴリズム。

「safety は仮定なしで成立し、liveness だけがタイミング仮定に依存する」——この非対称性が、合意プロトコルの設計を貫く第一原理である。

### 2. Single-decree Paxos — クォーラム交差の証明スケッチ

一つの値を決める最小形。各提案者は一意で単調増加する ballot 番号 n を使う。

```
フェーズ1: prepare(n) を過半数へ
  受信側: n が過去に応じたどの番号より大きければ「n 未満はもう accept しない」
          と約束し、既に accept 済みの (ballot, value) があれば返す
フェーズ2: 返答中に accept 済みの値があれば「最大 ballot の値」を、
          なければ自分の値を accept(n, v) として過半数へ
決定:     過半数が accept(n, v) を受理したら v は決定 (chosen)
```

**safety の核心**: 値 v が ballot n で決まった（過半数 Q₁ が accept した）とする。それより大きい任意の ballot n′ の提案者は、フェーズ1で過半数 Q₂ から応答を得るが、**Q₁ ∩ Q₂ ≠ ∅** なので少なくとも1台は「v を accept 済み」と報告する。規則により提案者は v を引き継ぐしかない。よって**一度決まった値は、以後のどの ballot でも変わらない**。帰納法でこれが全ての ballot に伝播する。

- フェーズ1は「値を決める」のではなく「**過去に決まったかもしれない値を学び、古い提案を無効化する**」ためにある。
- 2提案者が ballot を競り上げ続けると永遠に決まらない（liveness 喪失 = FLP の顕在化）。だから実用では**1人に提案を集中させる**——これがリーダであり、Multi-Paxos への動機である。

### 3. Multi-Paxos と Raft の定常状態

ログの各スロットで Paxos を走らせるのが Multi-Paxos。安定リーダがいれば、フェーズ1を「全スロット一括」で済ませ（リーダ在任中は省略）、定常状態は次のようになる。

```
リーダ: クライアント要求をログ末尾に追記 → 全フォロワへ複製 → 過半数の ACK でコミット → 適用・応答
```

これは Raft の AppendEntries と同一の形である。Raft がこの定常状態に加えて規定するのが、ログの健全性を保つ2つの性質である。

- **Log Matching**: 「同じ index に同じ term のエントリを持つなら、そこまでのログ全体が一致する」。AppendEntries が直前エントリの (index, term) を運び、不一致なら拒否 → リーダが遡って上書きする、という機構で帰納的に維持される。
- **選挙制限 (election restriction)**: 候補は「自分のログが投票者と同等以上に新しい」場合しか票を得られない（最後のエントリの term、次に長さで比較）。**コミット済みエントリは過半数に存在する**ので、過半数の票を得たリーダは必ずそれを持っている——Paxos のフェーズ1（過去を学ぶ）が、投票の条件に畳み込まれている。

**コミットの微妙な点**: リーダは**現在の term のエントリ**の複製が過半数に達したときだけ「直接」コミットできる。過去 term のエントリは、現 term のエントリのコミットに**巻き込まれて**間接的に確定する。過去 term のエントリを複製数だけで確定させると、リーダ交代の連鎖で上書きされうる反例が存在する（Raft 論文 §5.4.2 の Figure 8）——合意プロトコルの正しさが「もっともらしい単純化」で壊れる代表例。

### 4. 系譜 — VSR と ZAB

- **Viewstamped Replication (VSR, 1988)**: Paxos とほぼ同時期に独立に発明されたリーダベースのプロトコル。view（≒term）と view change（≒選挙）という語彙はここから来ており、Raft は思想的に VSR/ZAB 系の「リーダ中心」提示に近い。
- **ZAB (ZooKeeper Atomic Broadcast)**: ZooKeeper の複製プロトコル。プライマリの提案を FIFO 順で原子的にブロードキャストする。「合意」より「原子ブロードキャスト」として提示されるが、両問題は相互に還元可能である。

### 5. 再構成 (Membership Change)

ノード集合 C_old → C_new の変更は「過半数」の定義を変える。素朴に切り替えると、移行の瞬間に **C_old の過半数と C_new の過半数が交わらず**、同時に2つの決定が成立しうる。

- **joint consensus (Raft 論文の原案)**: 中間状態 C_old,new を挟み、その間は**両方の過半数**を要求する。交差が常に保証される。
- **1台ずつの変更 (single-server change)**: 1台の追加/削除なら新旧過半数は必ず交わる、という性質を使い、変更を直列化する。実装が単純で etcd 等はこちら。ただし正しさの証明は見かけより微妙で、原論文発表後にバグ（liveness ではなく safety の）が発見・修正された経緯があり、再構成が最難関という主張の傍証になっている。

### 6. 読み取りの線形化 — 合意を「読むたび」に払わない

読み取りもログに通せば線形化可能だが遅い。実用の最適化を以下に示す。

- **ReadIndex**: リーダが「自分はまだリーダか」を過半数へのハートビートで確認してから、確認時点のコミット index まで適用を待って読む。ログ追記なしで線形化可能。
- **リースリード (lease read)**: 選挙タイムアウトの間は新リーダが生まれない、というクロック仮定を使いハートビートすら省く。**クロックのずれが仮定を破ると stale read になる**——性能と引き換えに同期仮定を追加する典型例。
- フォロワ読み（stale 許容）を選ぶなら、それは [`consistency-models.md`](consistency-models.md) の一貫性ダイヤルを弱い側へ回す行為そのものである。

### 7. 実システムでの姿

| システム | プロトコル | 位置づけ |
|---|---|---|
| **etcd** | Raft | Kubernetes の制御平面ストア。単一 Raft グループ |
| **ZooKeeper** | ZAB | 分散ロック・設定。セッションと watch を持つ |
| **Spanner** | Paxos（シャードごと）+ TrueTime | 多数の Paxos グループ＋大域タイムスタンプで分散トランザクションへ拡張 |
| **CockroachDB / TiDB** | Raft（Range/Region ごと） | 数万の Raft グループを束ねる。合意は「シャード内複製」の部品になる |

単一グループの合意はスケールしない（全操作が1リーダを通る）ため、大規模系は**データを分割して多数の合意グループを並走させ**、グループ間は分散トランザクション（[`../databases/transactions-isolation.md`](../databases/transactions-isolation.md) の先の話題）で繋ぐ、という階層になる。

## セキュリティ考察 (Security Considerations)

- **故障モデルの破れが safety を壊す**: Paxos/Raft の正しさは「ノードは止まるが嘘はつかない・**ディスクに書いたものは消えない**」に依存する。fsync が嘘をつく、ディスクが巻き戻る（[`../databases/storage-engines.md`](../databases/storage-engines.md) の永続性の議論）、あるいはノードが侵害されて偽メッセージを送ると、クラッシュ耐性プロトコルの保証は静かに消える。**侵害ノード1台で任意の合意を壊せる**——Byzantine 耐性が必要なら PBFT 系 (3f+1) へ移る必要があり、コストは桁で違う。
- **選挙妨害による DoS**: 偽のタイムアウト誘発（遅延注入）や高い term の乱発でリーダ交代を空転させれば、safety を破らずに可用性を奪える。ノード間通信の認証 (mTLS) とネットワーク分離が前提条件になる。
- **検証の文化**: 合意実装のバグは「もっともらしい最適化」に宿る（§3 のコミット規則、§5 の再構成）。Jepsen による故障注入テスト、TLA+ による仕様検証（Raft/VSR は形式仕様が公開されている）が実務の標準になりつつある。「テストで動いた」が保証にならない点は、並行処理（[`../concurrency/memory-models-synchronization.md`](../concurrency/memory-models-synchronization.md)）と同じ構造である。

## 結論 (Conclusions)

- 合意は「故障があっても単一のログに不可逆に一致する」問題であり、複製状態機械への還元によって分散システムの基盤部品になる。
- FLP により非同期では決定的に解けない。実用プロトコルは liveness だけをタイミング仮定に依存させ、**safety はクォーラム交差と番号の単調性だけで無条件に守る**。
- Paxos と Raft は同じ骨格の異なる提示であり、難所は定常状態ではなく**リーダ交代と再構成**——「決まったかもしれない値を確実に引き継ぐ」部分に集中する。
- 読み取り最適化・マルチグループ化など実務の拡張は、一貫性・同期仮定・スケールのトレードオフを一つずつ明示的に選び直す行為である。
- 保証は故障モデルの内側でしか成立しない。ディスクの嘘・ノード侵害はモデル外であり、運用（認証・分離・検証）でモデルを現実に繋ぎ止める必要がある。

### 次の接続先 (Related Topics)

- [`consistency-models.md`](consistency-models.md) — 合意が実現する保証の側（線形化可能性、クォーラム）。本書はその実現機構。
- [`../databases/storage-engines.md`](../databases/storage-engines.md) — ログの永続化 (WAL/fsync) は合意の safety の物理的前提。
- [`../databases/transactions-isolation.md`](../databases/transactions-isolation.md) — 合意グループの上に載る並行制御。Spanner 型システムの残り半分。
- [`../../engineering/aerospace/avionics-gnc.md`](../../engineering/aerospace/avionics-gnc.md) — Byzantine 障害 (3f+1) を実装してきた冗長飛行計算機。故障モデルの強い側の世界。
- [`../../hardware/cpu/cache-coherence.md`](../../hardware/cpu/cache-coherence.md) — 単一値の合意機構としてのコヒーレンスプロトコルとの対比。

## 参考文献 (References)

**一次資料 (Primary — 原論文・仕様)**

- Fischer, M. J., Lynch, N. A., & Paterson, M. S. (1985). Impossibility of Distributed Consensus with One Faulty Process. *Journal of the ACM*, 32(2), 374–382. DOI: 10.1145/3149.214121 — FLP 不可能性の原典。
- Lamport, L. (1998). The Part-Time Parliament. *ACM TOCS*, 16(2), 133–169. DOI: 10.1145/279227.279229 — Paxos の原典。
- Lamport, L. (2001). Paxos Made Simple. *ACM SIGACT News*, 32(4), 51–58. — [PDF (著者サイト)](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) — Accessed: 2026-07-02。single-decree の平易な提示。
- Ongaro, D., & Ousterhout, J. (2014). In Search of an Understandable Consensus Algorithm. *USENIX ATC '14*. — [raft.github.io](https://raft.github.io/raft.pdf) — Accessed: 2026-07-02。Raft の原典（Figure 8 のコミット規則、joint consensus を含む）。
- Oki, B. M., & Liskov, B. (1988). Viewstamped Replication. *PODC '88*. DOI: 10.1145/62546.62549 ／ Liskov, B., & Cowling, J. (2012). *Viewstamped Replication Revisited* (MIT-CSAIL-TR-2012-021). — [PDF](https://pmg.csail.mit.edu/papers/vr-revisited.pdf) — Accessed: 2026-07-02。
- Junqueira, F. P., Reed, B. C., & Serafini, M. (2011). Zab: High-performance broadcast for primary-backup systems. *IEEE/IFIP DSN '11*. DOI: 10.1109/DSN.2011.5958223
- Castro, M., & Liskov, B. (1999). Practical Byzantine Fault Tolerance. *OSDI '99*. — [PDF](https://pmg.csail.mit.edu/papers/osdi99.pdf) — Accessed: 2026-07-02。Byzantine 側の代表。

**二次資料 (Secondary — 検証・解説)**

- Jepsen — 分散システムの故障注入検証レポート群。 — [jepsen.io/analyses](https://jepsen.io/analyses) — Accessed: 2026-07-02。
- Ongaro, D. (2014). *Consensus: Bridging Theory and Practice* (PhD thesis, Stanford). — single-server 再構成の詳細と修正の経緯を含む。
