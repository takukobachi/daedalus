# トランザクションと分離レベル (Transactions & Isolation Levels)

## 概要 (Overview)

[`storage-engines.md`](storage-engines.md) は単一操作の永続性 (WAL・ARIES) までを扱い、「並行制御への入口」で止めた。本ドキュメントはその続き——**複数の読み書きを一つの単位として、並行実行しても正しく見せる**仕組み、すなわちトランザクションの **ACID** のうち **I (Isolation, 分離)** を主題とする。

分離の理想は**直列化可能性 (serializability)**——並行実行の結果が「何らかの逐次実行」と等価であること——だが、厳密に守ると性能を失う。そこで実務のデータベースは**分離レベル (isolation level)** というダイヤルを提供し、「性能と引き換えに、どの異常 (anomaly) を許すか」を選ばせる。問題は、このダイヤルの目盛りが**標準 (ANSI SQL) の定義からして曖昧**で、同じ名前のレベルが DB ごとに別物だという点にある。

本ドキュメントは、(1) 異常のカタログとレベル定義（ANSI とその批判）、(2) 実装機構——2PL・MVCC・スナップショット分離 (SI)・SSI、(3) 主要 DB の実際の挙動、(4) 分離の失敗がそのままセキュリティバグになる構造を整理する。分析深度は Level 3。

## 方法論 (Methodology)

- 分離レベルを「防ぐ**異常**の集合」として定義する。Berenson らの古典的批判 (1995) に従い、ANSI の曖昧さと、ANSI に存在しないレベル（スナップショット分離）を明示する。
- 理論（直列化可能性・先行グラフ）→ 機構（2PL / MVCC / SI / SSI）→ 実装（PostgreSQL / MySQL InnoDB）の3層で対応づける。
- 各異常は「口座残高」など具体的な失敗シナリオで示す。実 DB の挙動は Hermitage（分離レベル検証スイート）で再現・検証可能なものに基づく。
- 単一ノードに閉じ、分散トランザクションは接続先として示すに留める。

## 発見事項 (Findings)

1. **分離レベルは「正しさの度合い」ではなく「許す異常の集合」である。** serializable 未満のレベルはすべて、特定の並行パターンで**アプリケーションから見て不正な結果**を許す。どの異常が自分のワークロードで実害になるかを知らずにレベルを選ぶことはできない。
2. **ANSI SQL の定義は壊れている (Berenson et al. 1995)。** 現象の英語定義が多義的で、素直に読むと dirty write を禁止し損ね、スナップショット分離——ANSI の表のどこにも収まらないのに repeatable read の異常を防ぐ——を分類できない。実務の語彙は ANSI ではなく Berenson らの拡張（P0〜、write skew 等）に基づく。
3. **現代の主流は MVCC——「読者は書き手をブロックせず、書き手は読者をブロックしない」。** 読み取りは過去のスナップショットを見るため読みロックが不要になる。PostgreSQL・MySQL InnoDB・Oracle はいずれも MVCC 基盤で、純粋な 2PL の DB は少数派になった。
4. **スナップショット分離 (SI) は serializable ではない。** 各トランザクションが一貫したスナップショットを読み、write-write 衝突だけを検出する SI は、ほとんどの異常を防ぐが **write skew**——互いの読んだものを書き換える2トランザクションが両方成功する——を許す。Oracle が SERIALIZABLE と呼ぶものは SI であり、PostgreSQL の REPEATABLE READ も SI である。
5. **SSI が SI と serializable の溝を埋めた。** rw 反依存 (rw-antidependency) の「危険な構造」を実行時に検出して片方を中断する Serializable Snapshot Isolation (Cahill 2008) が、PostgreSQL 9.1 の SERIALIZABLE として実装された——ロックで防ぐ (2PL) のでなく、**楽観的に走らせて危険なパターンだけ落とす**という第三の道。
6. **同じレベル名でも DB ごとに意味が違う。** PostgreSQL の REPEATABLE READ は SI（ファントムも防ぐが write skew を許す）、MySQL InnoDB の REPEATABLE READ はスナップショット読み＋ギャップロックの混成で、**同じ SQL が DB を替えると別の結果を返す**。移植やマルチ DB 構成での落とし穴。
7. **分離の異常はそのままセキュリティバグになる。** lost update は残高の二重引き出しに、write skew は「検査してから使う」型の制約破り（TOCTOU の DB 版）になる。分離レベルの選択はセキュリティ設計の一部である。

## アーキテクチャ (Architecture)

### 異常のカタログ (Anomaly Catalog)

| 異常 | 略号 | 内容（失敗シナリオ） |
|---|---|---|
| **dirty write** | P0 | 未コミットの書き込みを上書きする。ロールバックの意味が壊れるため全レベルで禁止 |
| **dirty read** | P1 | 未コミットの値を読む。後で消える残高を見て判断してしまう |
| **non-repeatable read** | P2 | 同じ行を2回読むと値が変わっている（間に他がコミット） |
| **phantom** | P3 | 同じ**条件**で2回検索すると行集合が変わっている（挿入・削除） |
| **lost update** | P4 | 読んだ値に基づく更新が、並行更新を黙って上書きする（`残高−100` が並行すると片方が消える） |
| **read skew** | A5A | 関連する複数行を別々の時点で読み、全体として存在しなかった状態を観測する |
| **write skew** | A5B | 2つのトランザクションが**互いの読んだ行を**書き換え、両方成功する（後述） |

### レベル × 異常 × 実装機構の対応

| レベル | P1 dirty read | P2 non-repeat | P3 phantom | P4 lost update | A5B write skew | 代表実装機構 |
|---|---|---|---|---|---|---|
| READ UNCOMMITTED | 許す | 許す | 許す | 許す | 許す | （ほぼ使われない） |
| READ COMMITTED | 防ぐ | 許す | 許す | 許す | 許す | 文単位スナップショット (MVCC) |
| REPEATABLE READ (ANSI) | 防ぐ | 防ぐ | 許す | 防ぐ | 許す | 行ロック保持 (2PL) |
| **SNAPSHOT (SI)** | 防ぐ | 防ぐ | 防ぐ | 防ぐ | **許す** | Tx 単位スナップショット＋WW 検出 |
| SERIALIZABLE | 防ぐ | 防ぐ | 防ぐ | 防ぐ | 防ぐ | S2PL＋predicate lock ／ **SSI** |

要点: SI は ANSI の表の「REPEATABLE READ と SERIALIZABLE の間」に**斜めに**入る——phantom を防ぐのに write skew を許すため、ANSI の一次元の物差しでは表せない。これが「ANSI 定義は壊れている」の実質である。

### 実装機構の3系譜

```
  悲観 (pessimistic)                楽観 (optimistic)
  ┌─────────────────┐   ┌──────────────────────────────────┐
  │ 2PL: 触る前にロック │   │ MVCC/SI: まず走らせ、コミット時に検証 │
  │ 衝突は「待ち」になる │   │ 衝突は「中断・リトライ」になる         │
  └─────────────────┘   └──────────────────────────────────┘
        │                     │                    │
     S2PL+predicate lock    SI (WW 検出のみ)    SSI (WW + 危険な rw 構造)
     = serializable         = write skew 残る   = serializable
```

## 技術詳細 (Technical Details)

### 1. 直列化可能性の理論 — 先行グラフ

履歴（操作の交錯列）が **conflict serializable** であるとは、衝突する操作対（同一データに対し少なくとも一方が書き）の順序を保ったまま逐次履歴に並べ替えられること。判定は**先行グラフ (precedence graph)**——Tᵢ の操作が Tⱼ の衝突操作に先行するとき辺 Tᵢ→Tⱼ を張る——が**非巡回**であることと同値 (Papadimitriou 1979)。

以降の機構はすべて「このグラフにサイクルを作らせない（または検出して壊す）方法」として読める: 2PL はロックで辺の向きを一方向に強制し、SSI はサイクルの必要条件（連続する rw 辺）を実行時に見張る。

### 2. 2PL — ロックで防ぐ

- **二相ロック (2PL)**: 各トランザクションはロックを「取る一方の相」と「放す一方の相」に分かれる（取得後に解放したら、以後新規取得しない）。これだけで conflict serializable が保証される。実務は**厳格 2PL (S2PL)**——書きロックをコミットまで保持——で、カスケード中断を防ぐ。
- **デッドロック**: 2PL の構造的な代償。待機グラフのサイクル検出（InnoDB）か、タイムアウト＋犠牲者選択で解消する。アプリ側は「リトライ可能に書く」ことが前提になる。
- **ファントムと述語ロック**: 行ロックは「まだ存在しない行」を守れない。条件を守るには**述語ロック (predicate lock)**、実装的には**インデックス範囲のギャップロック**（InnoDB の gap / next-key lock）で「この範囲への挿入」を塞ぐ。
- ロックの実体はストレージエンジンのラッチとは別物である（[`storage-engines.md`](storage-engines.md) の「ラッチ・ロック」の区別: ラッチは物理構造をマイクロ秒守る、ロックは論理データをトランザクション期間守る）。

### 3. MVCC — バージョンで読み書きを分離する

**多版並行制御 (MVCC: Multi-Version Concurrency Control)** は、更新を上書きでなく**新バージョンの追加**として行い、読者には「開始時点で見えるべきバージョン」を選んで見せる。

```
 行 X のバージョンチェーン:
   [X v1: 作成Tx=100, 削除Tx=150] → [X v2: 作成Tx=150, 削除Tx=∞]

 スナップショット = 「自分の開始時にコミット済みだった Tx の集合」
 可視性判定: 作成Txが自分から見えてコミット済み、かつ削除Txが見えない → そのバージョンを読む
```

- PostgreSQL は各行に xmin/xmax（作成/削除トランザクション ID）を持ち、古いバージョンは VACUUM が回収する。InnoDB は undo ログからバージョンを再構成する。**「読者は書き手をブロックしない」**が最大の効能で、長い分析クエリと更新が共存できる。
- 代償は**バージョンの蓄積**: 長寿命トランザクションが1本あるだけで、その開始以降の全旧バージョンが回収できず、テーブル膨張 (bloat) と回収コストが積み上がる。ストレージ（[`storage-engines.md`](storage-engines.md) の space amplification）と運用に直結する。

### 4. スナップショット分離と write skew

**SI**: 各トランザクションは開始時点のスナップショットを読み、コミット時に「自分の書いた行を、並行トランザクションも書いていないか」だけ検証する (first-committer-wins)。write-write 衝突は防がれるが、**read-write の絡み**は素通りする。

古典例——当直医師制約「常に1人以上が当直」:

```
初期状態: Alice, Bob とも on-call
Tx1 (Alice): 当直者数を読む → 2人 → 自分を off に更新
Tx2 (Bob):   当直者数を読む → 2人 → 自分を off に更新
両者は別の行を書くので WW 衝突なし → 両方コミット成功 → 当直者 0 人
```

どちらの判断も自分のスナップショット内では正しい。**互いの読んだ前提を、互いの書き込みが壊した**——これが write skew (A5B) であり、「別々の行を書くが読みが交差する」あらゆる不変条件（残高合計、一意性の手動チェック、在庫引当）で再現する。

### 5. SSI — 危険な構造の実行時検出

理論的基礎: SI 下で非直列化可能な履歴には、先行グラフ上で**連続する2本の rw 反依存辺**（Tᵢ が読んだものを Tⱼ が書き、Tⱼ が読んだものを Tₖ が書く）を持つ「危険な構造」が必ず現れる (Fekete et al.)。**SSI (Serializable Snapshot Isolation)** はこの必要条件を実行時に監視する:

- 各トランザクションの読みを SIREAD ロック（ブロックしない目印）として記録し、rw 反依存の入り辺・出り辺を追跡。
- 両方の辺が揃った瞬間、関与するどれかを**中断**してサイクルの可能性を断つ（偽陽性はある——必要条件なので、実際には直列化可能な履歴も落としうる）。
- PostgreSQL の SERIALIZABLE (9.1〜) がこの実装 (Ports & Grittner 2012)。読みが多いと追跡メモリと偽陽性中断が増えるが、2PL のように読者が書き手を待たせることはない。アプリ側の契約は「**serialization failure をリトライする**」こと。

### 6. 実 DB の対応表 — 同名別物

| DB | READ COMMITTED | REPEATABLE READ | SERIALIZABLE |
|---|---|---|---|
| **PostgreSQL** | 文ごとの新スナップショット | **SI**（phantom も防ぐ / write skew 許す） | **SSI** |
| **MySQL InnoDB** | 文ごとスナップショット | スナップショット読み＋**ギャップロック**（ロック読みのみ）。通常読みは write skew 許す | S2PL 化（全読みが共有ロック） |
| **Oracle** | 文ごとスナップショット | （提供なし） | **SI**（真の serializable ではない） |
| **SQL Server** | ロック or RCSI（設定依存） | 2PL 系 | S2PL ／ SNAPSHOT レベルは SI |

- 既定値も揃っていない: PostgreSQL/Oracle/SQL Server は READ COMMITTED、MySQL は REPEATABLE READ。**「既定のまま」は DB ごとに違う異常を許している**。
- この表の挙動は Hermitage（Kleppmann の検証スイート）で最小 SQL により再現できる。移植・比較の一次検証手段として優れている。

### 7. アプリケーションからの実務指針

- **まず不変条件を言語化する**（「合計が負にならない」「重複しない」）。その不変条件が「読んだ前提を他人が壊す」型なら SI では守れない——SERIALIZABLE にするか、`SELECT ... FOR UPDATE` で読みを衝突に格上げするか、制約 (UNIQUE, CHECK) を DB に押し込む。
- **リトライを設計に組み込む**。SSI・SI・デッドロック解消はいずれも「中断してやり直す」ことを正常系に含む。冪等性・リトライ上限・バックオフはアプリの責務。
- **トランザクションは短く**。長寿命 Tx は MVCC の bloat・ロック保持・SSI の追跡コストのすべてを悪化させる、系全体への外部性。

## セキュリティ考察 (Security Considerations)

- **分離異常は攻撃者が誘発できる**: lost update / write skew は偶然を待たなくても、**リクエストを並行に投げるだけ**で意図的に起こせる。残高チェック→引き出しの並行実行によるダブルスペンド、クーポンの多重適用、在庫の過剰引当は、いずれも「弱い分離レベル＋検査してから使う」パターンの帰結で、**TOCTOU（[`../concurrency/memory-models-and-synchronization.md`](../concurrency/memory-models-and-synchronization.md)）のデータベース版**である。対策も同型——検査と使用を同一の原子単位（SERIALIZABLE、FOR UPDATE、一意制約）に入れる。
- **決済・資金移動は serializable 側に倒す**: [`../specifications/stripe-connect-specification.md`](../specifications/stripe-connect-specification.md) が扱う資金整合は、この異常クラスを冪等キーと台帳突合で多層防御する実務例。
- **ロックは DoS の道具になる**: 長い明示ロックや巨大トランザクションで他者を待たせる・デッドロックを量産する攻撃面がある。タイムアウト (`lock_timeout` 等) と権限分離で bound する。
- **バージョンは情報を残す**: MVCC の旧バージョンは回収まで物理的に残る（[`storage-engines.md`](storage-engines.md) の data remanence と同根）。削除の即時性を仮定しない。

## 結論 (Conclusions)

- 分離レベルは「許す異常の集合」であり、ANSI の名前ではなく**異常カタログ**（dirty read / lost update / write skew / phantom）で考えるのが唯一実用的な語彙である。
- 実装は悲観 (2PL) から楽観 (MVCC/SI/SSI) へ重心を移した。MVCC は「読者と書き手の非干渉」で並行性を稼ぎ、SI の残した write skew を SSI が実行時検出で塞いだ。
- 同じレベル名でも DB ごとに実体が違う。守りたい不変条件から逆算してレベルと明示ロックを選び、**中断・リトライを正常系として設計する**ことが、この層のエンジニアリングの本体である。
- 分離の失敗は正しさのバグであると同時にセキュリティの穴である。並行リクエストは攻撃者の自由変数であることを忘れてはならない。

### 次の接続先 (Related Topics)

- [`storage-engines.md`](storage-engines.md) — 本書の下の層。MVCC のバージョンが載るページ・WAL・VACUUM の物理。
- [`../concurrency/memory-models-and-synchronization.md`](../concurrency/memory-models-and-synchronization.md) — 同じ「並行実行の正しさ」をメモリ語で扱う層。データ競合/TOCTOU ↔ 分離異常の対応。
- [`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) ／ [`../distributed-systems/consensus.md`](../distributed-systems/consensus.md) — 分離 (isolation) と一貫性 (consistency) の交点。分散トランザクション（2PC + Paxos/Raft グループ）は両者の合流点で、今後のテーマ。

## 参考文献 (References)

**一次資料 (Primary — 論文・公式ドキュメント)**

- Berenson, H., Bernstein, P., Gray, J., Melton, J., O'Neil, E., & O'Neil, P. (1995). A Critique of ANSI SQL Isolation Levels. *SIGMOD '95*, 1–10. DOI: 10.1145/223784.223785 — 異常カタログ（P0〜、A5A/A5B）とスナップショット分離の定式化。本書の語彙の出典。
- Papadimitriou, C. H. (1979). The Serializability of Concurrent Database Updates. *Journal of the ACM*, 26(4), 631–653. DOI: 10.1145/322154.322158 — 先行グラフによる直列化可能性理論。
- Fekete, A., Liarokapis, D., O'Neil, E., O'Neil, P., & Shasha, D. (2005). Making Snapshot Isolation Serializable. *ACM TODS*, 30(2), 492–528. DOI: 10.1145/1071610.1071615 — 危険な構造（連続 rw 反依存）の理論。
- Cahill, M. J., Röhm, U., & Fekete, A. D. (2008). Serializable Isolation for Snapshot Databases. *SIGMOD '08*. DOI: 10.1145/1376616.1376690 — SSI の原典。
- Ports, D. R. K., & Grittner, K. (2012). Serializable Snapshot Isolation in PostgreSQL. *PVLDB*, 5(12), 1850–1861. DOI: 10.14778/2367502.2367523 — PostgreSQL 実装の一次解説。
- Adya, A. (1999). *Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions* (PhD thesis, MIT). — [PDF](https://pmg.csail.mit.edu/papers/adya-phd.pdf) — Accessed: 2026-07-02。実装非依存の分離レベル形式化 (G0/G1/G2)。
- PostgreSQL Documentation, *Transaction Isolation*. — [postgresql.org](https://www.postgresql.org/docs/current/transaction-iso.html) — Accessed: 2026-07-02。

**二次資料 (Secondary — 検証・教科書)**

- Kleppmann, M. *Hermitage: Testing transaction isolation levels*. — [github.com/ept/hermitage](https://github.com/ept/hermitage) — Accessed: 2026-07-02。§6 の DB 別挙動の再現スイート。
- Gray, J., & Reuter, A. (1993). *Transaction Processing: Concepts and Techniques*. Morgan Kaufmann. — 2PL・ロック・回復の標準教科書。
