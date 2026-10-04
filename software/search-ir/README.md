# Search & Information Retrieval

検索と情報検索 (search & information retrieval) の分析。「数十億の対象から、曖昧な
述語（関連度・意味の近さ）で top-k を数十ミリ秒で返す」という、厳密一致のデータベースとは
別種の問題系を扱う。語彙一致の側は転置インデックス・BM25・動的枝刈り、意味検索の側は
埋め込みと近似最近傍探索 (ANN) が担い、両者はハイブリッド検索として合成される。
不変セグメント＋背景マージという構造は `../databases/storage-engines.md` の LSM-tree と
同型であり、分散検索の scatter-gather は `../distributed-systems/` の応用、埋め込みの
供給側は `../ml-systems/` と地続きになっている。全体に共通するのは「読まない工学」である。圧縮・
スキップ・枝刈り・近似のすべてが「関連しない対象に触れずに終える」ための仕組みである。

## 構成

- `inverted-index-fulltext-search.md` — 転置インデックスと全文検索：テキスト解析
  （形態素 vs n-gram、解析の非可逆性）、索引構造（FST 語彙表、ポスティング、doc values）、
  圧縮と交差（差分符号化、スキップ、ギャロッピング）、動的枝刈り (WAND/Block-Max WAND)、
  BM25 の確率的導出、不変セグメントと準リアルタイム検索（LSM との同型対応）、
  分散検索の scatter-gather（深いページング、統計の分断、テール増幅）
- `vector-search-ann.md` — ベクトル検索と近似最近傍探索：次元の呪い（距離の集中）、
  設計族のメモリ階層的分類（HNSW / IVF+PQ / DiskANN）、直積量子化と ADC、
  recall@k の SLO 化、フィルタ・更新・複製による「データベース化」、
  ハイブリッド検索 (RRF)、埋め込み逆転と RAG 汚染のセキュリティ

## 今後のテーマ候補

- ランキング学習 (learning to rank) — pointwise/pairwise/listwise、クリックデータの
  バイアス補正、オンライン評価 (interleaving)。`../ml-systems/` との合流点
- クロールとインデックスパイプライン (crawling & indexing pipeline) — 更新検知、
  重複排除 (SimHash/MinHash)、インデックス構築のバッチ/ストリーム設計。
  `../messaging-streaming/` の応用
- 検索品質の評価 (search quality evaluation) — オフライン評価 (nDCG, MRR)、
  判定データの作り方、RAG 時代の検索評価（回答支持率）
