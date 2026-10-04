# グラフと木 (Graphs, Trees, and Graph Algorithms)

## 概要 (Overview)

離散数学・論理シリーズの第2回。**グラフ (graph)** は「モノの集合と、その間の関係」だけを残した最小の構造である——ルータと回線、オブジェクトと参照、トランザクションと待ち、タスクと依存。Daedalus の分析に現れる「〜のトポロジ」「依存関係」「到達可能」「デッドロック」という言葉はすべてグラフの語彙であり、本回はその正確な定義と、上に載る基本アルゴリズム（探索・トポロジカル順序・最短経路）を積み上げる。

グラフの強さは、**関係の問題を構造の問題に変換する**ことにある。「この変数はもう使われないか」は到達可能性に、「このロック待ちは永遠に解けないか」は閉路の存在に、「どの順でビルドするか」はトポロジカル順序に、「パケットをどう転送するか」は最短経路に化ける。そして BGP・ガベージコレクタ・ビルドシステムは、同じ数学の分散版・並行版・増分版として読めるようになる。行列との橋——隣接行列の冪が経路を数える——で線形代数シリーズとも合流する。

## 前提知識 (Prerequisites)

- [`propositional-boolean-logic.md`](propositional-boolean-logic.md) — 集合・論理の記法、「有限だから全数検査できる」という離散数学の作法
- [`../linear-algebra/vectors-matrices-linear-maps.md`](../linear-algebra/vectors-matrices-linear-maps.md) — 隣接行列の冪の解釈に使う行列積

## 本論 (Main Content)

### 1. 定義と表現 (definitions and representations)

グラフ G = (V, E) は頂点集合 V と辺集合 E の対である。辺に向きがあれば**有向グラフ (directed graph)**、重みが付けば**重み付きグラフ**。頂点数 n = |V|、辺数 m = |E| に対し m は最大 O(n²) だが、実システムのグラフはほぼ常に**疎 (sparse)**——ルータの隣接数もオブジェクトの参照数も n に比べ小さい——であり、この疎性が表現の選択を決める (CLRS 2022, Ch. 20)。

```text
隣接行列 A (n×n):  A[i][j] = 辺 i→j の有無/重み。辺の有無判定 O(1)、空間 O(n²)
隣接リスト:        各頂点に隣接頂点のリスト。隣接列挙 O(deg)、空間 O(n+m)
→ 疎グラフでは隣接リストが既定。行列表現の真価は代数側にある:
   (Aᵏ)[i][j] = i から j への長さ k の歩道 (walk) の本数
```

隣接行列の冪が経路を数える——グラフの問題が [`../linear-algebra/`](../linear-algebra/) の行列計算に翻訳される橋であり、到達可能性・中心性・PageRank（確率・統計シリーズのマルコフ連鎖）はすべてこの橋の上にある。

### 2. 木 — 閉路のない連結 (trees)

**木 (tree)** は「連結で閉路がない」グラフであり、以下はすべて同値な特徴づけである: (a) 連結かつ無閉路、(b) 連結かつ辺数 n−1、(c) 任意の2頂点間の路がちょうど1本。「ちょうど1本」が木の本質で——経路の一意性は**ループフリーな転送**（どこへも一通りにしか流れない）と**階層の一意な親**を同時に意味する。

連結グラフから閉路を壊して木だけ残したものが**全域木 (spanning tree)** である。重み付きなら**最小全域木 (MST)** が定義でき、「カットを横切る最小辺は必ず MST に入る」というカット性質から Kruskal・Prim の貪欲法が正当化される (CLRS 2022, Ch. 21)。L2 ネットワークのスパニングツリープロトコル (STP) は、ブロードキャストストームを避けるため物理トポロジから全域木を**分散計算で**切り出すプロトコルであり (Perlman 1985)、「冗長リンクを殺して一意性を買う」というカット性質の運用形である。

### 3. 探索 — BFS と DFS (graph traversal)

出発点から辺を辿って到達できる頂点を系統的に訪ねる操作が探索で、フロンティアの管理方法だけが違う2つの基本形がある。

```text
BFS (幅優先): フロンティアをキューで管理 — 近い順に広がる
  → 無重みグラフの最短経路、距離レベル分け。 O(n+m)
DFS (深さ優先): フロンティアをスタックで管理 — 行けるだけ深く
  → 閉路検出 (探索中の頂点への逆辺 = 閉路)、連結成分、
    トポロジカル順序 (帰りがけ順の逆)。 O(n+m)
```

**到達可能性 (reachability)** ——「根集合から辿れるか」——は探索の最も直接の応用であり、ガベージコレクションのマーク段階（ルートから参照辺を辿り、届かないオブジェクトがゴミ）、依存関係の解決、ネットワークの分断検出がこの一語に載る。**閉路検出**はもう一方の主役で、トランザクションの待ちグラフ (wait-for graph) に閉路があることがデッドロックの定義そのものである。

### 4. DAG とトポロジカル順序 (DAGs and topological ordering)

閉路のない有向グラフ **DAG (directed acyclic graph)** は「依存関係」の標準モデルである。基本定理: **有向グラフがトポロジカル順序（すべての辺が前→後を向く一列の並び）を持つ ⇔ DAG である**。順序は Kahn のアルゴリズム——入次数 0 の頂点を取り出しては辺を消す——か DFS の帰りがけ順で O(n+m) で得られる (Kahn 1962)。

依存グラフが DAG である限り「正しい実行順序」が存在し、閉路はその不可能性（循環依存）を意味する——ビルドシステム・パッケージマネージャ・タスクスケジューラの土台である。さらに DAG 上では**最長経路（クリティカルパス）** が線形時間で計算でき、「並列度を無限にしても縮まない所要時間の下限」を与える——プロジェクト計画から命令スケジューリング、データフローパイプラインまで同じ式が支配する。

### 5. 最短経路 — Dijkstra と Bellman–Ford (shortest paths)

重み付きグラフの単一始点最短経路には、前提の異なる2つの古典がある (CLRS 2022, Ch. 22)。

```text
Dijkstra (1959):  非負重み前提。確定済み集合から最小距離の頂点を貪欲に確定。
                  優先度付きキューで O((n+m) log n)。全体情報が必要 (集中型)
Bellman–Ford:     負重みも可。全辺の緩和 (relax) を n−1 回繰り返す。O(nm)。
                  「隣の推定値だけ」で動く — 分散化できる
```

この対比はそのまま経路制御プロトコルの系譜である。**リンクステート型** (OSPF/IS-IS) は全ルータがトポロジ全体を複製し、各自が Dijkstra を回す。**距離ベクトル型** (RIP) は Bellman–Ford の分散版で、各ルータが隣接へ距離推定を広告し合う——収束が遅く、リンク断で推定値が徐々に膨らむ**無限カウント (count-to-infinity)** という病理を持つ。BGP は距離でなく**経路そのもの（AS パス）をベクトルとして運ぶ**パスベクトル型で、経路にループ検出（自 AS が既に載っていれば拒否）とポリシー（最短でなく契約上の優先）を載せる——「最短経路問題」から「ポリシー付き経路選択」への一般化である。

## 工学への接続 (Engineering Connections)

- **経路制御はグラフアルゴリズムの分散実行である** (§5): リンクステート＝複製グラフ上の Dijkstra、距離ベクトル＝分散 Bellman–Ford、BGP＝ポリシー付きパスベクトル。プロトコルの病理（収束・ループ）はアルゴリズムの前提が分散環境で崩れる箇所に対応する。
- **ガベージコレクションは到達可能性計算である** (§3): ルート集合からの探索でライブ集合を定め、補集合を回収する。並行 GC の難しさは「探索中にグラフが書き換わる」こと——スナップショットとバリアはグラフ探索を動くグラフへ拡張する装置である。
- **デッドロック検出は閉路検出である** (§3): ロック待ちを辺とする wait-for graph の閉路がデッドロック。検出して壊すか（DB のビクティム選択）、順序付けで閉路を作らせないか（ロック順序規約＝辺の向きの制約による DAG 化）。
- **ビルドと依存解決はトポロジカル順序である** (§4): 依存 DAG の順序が実行計画、クリティカルパスが並列ビルドの下限、循環依存の検出が健全性検査になる。
- **近傍グラフ上の検索** (§3): HNSW のようなベクトル索引は「貪欲探索が局所最適に嵌まらないよう辺を設計した」グラフであり、探索の性質（直径・次数・航行可能性）が検索性能を決める。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`software/networking/ip-routing-bgp.md`](../../../software/networking/ip-routing-bgp.md) | リンクステート＝Dijkstra、距離ベクトル＝分散 Bellman–Ford、BGP＝パスベクトルとループ検出 (§5) |
| [`software/compilers-runtimes/managed-runtimes-gc-jit.md`](../../../software/compilers-runtimes/managed-runtimes-gc-jit.md) | GC のマーク＝到達可能性探索、並行 GC＝動くグラフ上の探索 (§3) |
| [`software/databases/transactions-isolation.md`](../../../software/databases/transactions-isolation.md) | wait-for graph の閉路＝デッドロック、直列化可能性グラフの無閉路性 (§3–4) |
| [`software/search-ir/vector-search-ann.md`](../../../software/search-ir/vector-search-ann.md) | HNSW＝航行可能性を設計した近傍グラフ、貪欲探索と長距離辺 (§3) |
| [`hardware/network-hardware/nic-switch-architecture.md`](../../../hardware/network-hardware/nic-switch-architecture.md) | L2 の STP＝分散全域木、冗長リンクの遮断とループフリー転送 (§2) |
| [`software/messaging-streaming/distributed-logs-delivery-semantics.md`](../../../software/messaging-streaming/distributed-logs-delivery-semantics.md) | ストリーム処理のデータフロー DAG、クリティカルパスとパイプライン律速 (§4) |

## 参考文献 (References)

- Cormen, T. H., Leiserson, C. E., Rivest, R. L., & Stein, C. (2022). *Introduction to Algorithms* (4th ed.). MIT Press. — 探索・MST・最短経路・DAG の主参照
- Diestel, R. (2017). *Graph Theory* (5th ed.). Springer. DOI: 10.1007/978-3-662-53622-3 — グラフ理論本体（木の特徴づけ・連結性）の主参照
- Dijkstra, E. W. (1959). A Note on Two Problems in Connexion with Graphs. *Numerische Mathematik*, 1, 269–271. DOI: 10.1007/BF01386390 — 最短経路貪欲法の原典
- Kahn, A. B. (1962). Topological Sorting of Large Networks. *Communications of the ACM*, 5(11), 558–562. DOI: 10.1145/368996.369025 — トポロジカル順序の原典
- Perlman, R. (1985). An Algorithm for Distributed Computation of a Spanning Tree in an Extended LAN. *SIGCOMM '85*, 44–53. — 分散全域木 (STP) の原典
