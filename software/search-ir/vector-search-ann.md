# ベクトル検索と近似最近傍探索 (Vector Search & Approximate Nearest Neighbor)

## 概要 (Overview)

**ベクトル検索 (vector search)** は、[`inverted-index-fulltext-search.md`](inverted-index-fulltext-search.md) の語彙一致検索が原理的に届かない領域——「クエリの語が文書に 1 語も現れないが、意味は同じ」——を扱う。埋め込みモデル (embedding model) がテキスト・画像・音声を数百〜数千次元のベクトルに写像し、**意味の近さを幾何学的な近さに変換**する。すると検索は最近傍探索 (nearest neighbor search) になる：クエリベクトルに最も近い k 本のベクトルをコーパスから見つけよ。RAG (retrieval-augmented generation)、推薦、重複検出、画像検索はすべてこの一つの問題に帰着する。

問題は、高次元での厳密な最近傍探索が**事実上不可能**なことである。本ドキュメントの中心となる問いは 5 つ。(1) なぜ厳密解は諦めるしかないのか——次元の呪い (curse of dimensionality) は索引に何をするのか、(2) 近似最近傍探索 (ANN) の主要な設計族——グラフ (HNSW)、転置クラスタ (IVF)、量子化 (PQ)、ディスク常駐 (DiskANN)——は再現率・レイテンシ・メモリの三角形をどう切り分けているのか、(3) 「正しさが確率的」なシステムをどう運用するのか——recall@k という SLO、(4) ANN 索引はフィルタ・更新・複製というデータベースの日常とどう衝突するのか、(5) 語彙検索とベクトル検索はどう合成するのか（ハイブリッド検索）。通奏低音として、ANN アルゴリズムの分類とは実は**メモリ階層への配置戦略の分類である**——グラフは RAM のランダムアクセス、量子化はキャッシュへの圧縮、IVF は帯域の節約、DiskANN は SSD の活用——という視点を採る。分析深度は Level 3。

## 方法論 (Methodology)

- 各アルゴリズム（HNSW, IVF, PQ, DiskANN）は原論文に基づいて機構を再構成し、性能特性は ann-benchmarks 系の公開ベンチマーク方法論（recall-QPS 曲線）の枠組みで比較する。単発の数値ではなく**トレードオフ曲線**を比較の単位とする。
- ベクトル空間と距離の数学は [`../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md`](../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md)（内積・ノルム）と [`../../foundations/mathematics/linear-algebra/svd-low-rank.md`](../../foundations/mathematics/linear-algebra/svd-low-rank.md)（低ランク近似・次元削減）の語彙で扱う。
- ハードウェアとの結合は [`../../hardware/cpu-architecture/simd-vector.md`](../../hardware/cpu-architecture/simd-vector.md)（SIMD 距離計算）と [`../../hardware/cpu-architecture/cache-memory-hierarchy.md`](../../hardware/cpu-architecture/cache-memory-hierarchy.md)（メモリ階層）を基準系にする。
- 埋め込みの生成側（モデルの学習・推論）は [`../ml-systems/`](../ml-systems/) に譲り、本稿は「ベクトルが与えられた後」の索引・検索・運用に集中する。

## 発見事項 (Findings)

1. **次元の呪いの実体は距離の集中であり、空間分割型索引の死である。** 次元が上がると、ランダムな点集合では最近傍と最遠点の距離比が 1 に近づき（距離の集中）、「近い」が薄い情報になる。KD-tree のような空間分割索引は、分割面がクエリ球と交差する枝を全部訪ねる必要があり、高次元（目安として数十次元以上）では総当たりに退化する。つまり ANN は最適化ではなく**降伏と再交渉**である——厳密性を手放す代わりに、実データが持つ低次元構造（埋め込みは一様ランダムではなく多様体上に集中する）だけを頼りに実用速度を取り戻す。
2. **HNSW は skip list のグラフ版であり、貪欲探索を「粗→密」の階層で初期化する装置である。** 各点を近傍グラフで結び、クエリから貪欲に近い方へ渡り歩く——これが基本操作である。上位層ほど疎な多層グラフ（各点の所属最上層は幾何分布で決まる）が skip list と同じ役割を果たし、遠距離の移動を上位層で済ませ、最下層で精密化する。探索幅 ef が再現率とレイテンシを連続的に交換する単一ノブになる。代償はメモリである——全ベクトル＋隣接リストが RAM に要り、アクセスパターンは純粋なランダムアクセスなので、[`cache-memory-hierarchy.md`](../../hardware/cpu-architecture/cache-memory-hierarchy.md) のいうキャッシュ最悪パターンをレイテンシの低さで踏み抜く設計である。
3. **IVF は「転置インデックス」の幾何学的再発明である。** ベクトル空間を k-means で nlist 個のセルに割り、各セルに所属ベクトルのリストを持つ——語→文書リストの構造とちょうど同型で、「語」が「セル（セントロイド）」に置き換わっている。検索はクエリに近い nprobe 個のセルだけを走査する。粗い量子化で候補を絞ってから精密計算する二段構えであり、nprobe が ef と同じ再現率ノブになる。走査は順次アクセスなのでメモリ帯域に素直に律速される——グラフのランダムアクセスと対照的である。
4. **直積量子化 (PQ) は距離計算をテーブル引きに変える非可逆圧縮である。** D 次元ベクトルを m 個の部分空間に割り、各部分空間を 256 セントロイド（8 bit）で量子化すると、1 本のベクトルが m バイトに潰れる——128 次元 float32 の 512 バイトが 16 バイトになる類の圧縮率である。距離は**非対称距離計算 (ADC)** で近似する：クエリ（非量子化）と各部分セントロイドの距離を先にテーブル化し、候補ごとの距離計算を m 回のテーブル参照＋加算に置換する。これは SIMD シャッフル命令によるレジスタ内テーブル参照と噛み合い、[`simd-vector.md`](../../hardware/cpu-architecture/simd-vector.md) の世界で桁違いのスループットになる。IVF+PQ の合成が単一ノードで十億本規模を成立させた。
5. **DiskANN はグラフ探索を SSD の作法に翻訳した。** RAM に入り切らない規模では、Vamana グラフ（HNSW より平坦で次数制約付き）の隣接リストと完全精度ベクトルを SSD に置き、RAM には PQ 圧縮ベクトルだけを持つ。探索は PQ 近似距離で舵を取り、訪問ノードの完全ベクトル読みは SSD のランダムリードとして発行される——ビーム探索で読みをバッチ化し、[`../../hardware/hw-sw-interface/storage-nvm-interface.md`](../../hardware/hw-sw-interface/storage-nvm-interface.md) のいう NVMe の並列キューを飽和させる。メモリ階層のもう一段下へ索引を伸ばした設計であり、「ANN 設計＝メモリ階層への配置戦略」という本稿の視点の最も明示的な実例である。
6. **recall は保証ではなく測定値であり、ANN 運用の第一級 SLO である。** ANN の「正しさ」は recall@k（厳密 top-k との重なり率）で定義されるが、これはデータ分布・クエリ分布・パラメータ (ef/nprobe) の関数であり、**データが変わればパラメータ据え置きでも劣化する**。ベンチマークは recall-QPS のパレート曲線で比較し、運用ではサンプルクエリに対する厳密解との突き合わせを継続的に走らせて監視する——「近似の品質」を可観測性の対象に格上げすることが、ANN をプロダクションで使う条件である。
7. **フィルタ付き検索と更新が、ANN 索引を「データベース化」する圧力である。** 「近い k 本、ただし tenant=A かつ日付が今月」——述語との合成は純粋 ANN には無い。事前フィルタはグラフの連結性を壊し（許可ノードだけ辿ると経路が断絶する）、事後フィルタは選択率が低いと k 本集まらない。実システムはフィルタ選択率に応じて総当たり・フィルタ内蔵グラフ探索・事後フィルタを切り替える。更新も同様で、HNSW への削除は墓標化と定期再構築、IVF はセントロイドの陳腐化と再学習を要する——不変構造＋背景再構築という [`../databases/storage-engines.md`](../databases/storage-engines.md) と同じ結論に収束しつつある。ベクトル DB とは、ANN アルゴリズムに WAL・複製・トランザクション的更新・アクセス制御という**データベースの装備一式を着せる**プロジェクトである。

## アーキテクチャ (Architecture)

### 検索スタックの中のベクトル検索 — RAG を例に

```
 文書群 ──▶ チャンク分割 ──▶ 埋め込みモデル ──▶ ベクトル + メタデータ
                              (ml-systems/)         │ upsert
                                                    ▼
                                        ┌──────────────────────┐
 クエリ ──▶ 埋め込みモデル ──▶ q ──────▶│ ベクトル索引 (ANN)      │
   │                                    │  HNSW / IVF-PQ / Disk │
   │ (同一モデル・同一前処理!)            │  + メタデータフィルタ   │
   │                                    └──────────┬───────────┘
   └──▶ BM25 (inverted-index) ──┐                  │ top-k'
                                ▼                  ▼
                         ┌─────────────────────────────┐
                         │ 融合 (RRF) → リランカ → top-k │──▶ LLM 文脈へ
                         └─────────────────────────────┘
```

埋め込みモデルは索引時とクエリ時で**同一**でなければならない——語彙検索における解析器の一致（[`inverted-index-fulltext-search.md`](inverted-index-fulltext-search.md) 発見事項 2）と同じ契約であり、**モデルの更新は全ベクトルの再埋め込み＝索引の全再構築**を意味する。この「モデルバージョンが索引のスキーマである」という性質が、ベクトル検索特有の運用重量の源泉である。

### 設計族の比較 — メモリ階層への配置戦略として

| 軸 | HNSW（グラフ） | IVF(+PQ)（クラスタ+量子化） | DiskANN（SSD グラフ） |
|---|---|---|---|
| 主資源 | RAM ランダムアクセス | メモリ帯域（順次走査） | SSD ランダムリード |
| メモリ消費 | 全ベクトル+隣接（最大） | PQ で 1/16〜1/64 に圧縮可 | RAM は PQ のみ、本体は SSD |
| 再現率ノブ | ef（探索幅） | nprobe（走査セル数） | ビーム幅 |
| 更新 | 逐次挿入は得意・削除が苦手 | 追記容易・セントロイド陳腐化 | 静的構築＋差分階層 |
| 得意域 | 高 QPS・低レイテンシ・RAM に収まる規模 | 大規模・メモリ制約・GPU 化 | RAM を超える十億本規模を単一ノードで |

三者は優劣ではなく、「どの記憶階層を主戦場にするか」の選択である。規模がスケールアウトを要求する場合はこの上にシャーディング（各シャードでローカル ANN → top-k 統合）が載り、[`inverted-index-fulltext-search.md`](inverted-index-fulltext-search.md) の scatter-gather の幾何学がそのまま再現される。

## 技術詳細 (Technical Details)

### 1. 埋め込み空間と距離 — 検索の意味論は索引の外で決まる

- 距離はコサイン類似度・内積・L2 の三択が基本で、ベクトルを単位ノルムに正規化すればコサインは内積に、L2 の順位は内積の順位に一致する——実装が「正規化＋内積」に収斂するのはこのためである（[`vectors-matrices-linear-maps.md`](../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md) の内積の幾何）。学習時の目的関数（対照学習が典型）が想定する距離と索引の距離設定がずれると、索引が正しくても検索は壊れる。
- 検索品質の上限は埋め込みモデルが決め、ANN はそこから recall 分だけ割り引く。「関連しているのに出ない」の原因切り分け——モデルの写像が悪いのか、ANN の近似が漏らしたのか、チャンク分割が悪いのか——が運用の日常であり、recall 監視（発見事項 6）はこの切り分けの前提である。
- 次元数は数百〜数千。Matryoshka 型（先頭次元ほど情報を密に持つよう学習し、切り詰めても機能する）埋め込みは、次元数を精度-コストのノブに変える——[`svd-low-rank.md`](../../foundations/mathematics/linear-algebra/svd-low-rank.md) の低ランク近似の思想を学習で作り込んだものと読める。

### 2. HNSW — 構築と探索の詳細

- **構築**: 点を逐次挿入する。各点の最上層 l を確率 ~exp(−l/mL) の幾何分布で引き（mL = 1/ln M が標準）、各層で「近傍 ef_construction 本を探索 → 多様性ヒューリスティック（近すぎる隣接同士を間引き、方向の異なる辺を残す）で M 本に刈る」。この間引きが、クラスタ内に閉じない**航行可能な (navigable)** 長距離辺を保存し、貪欲探索が局所最適に嵌まるのを防ぐ。
- **探索**: 最上層の入口点から貪欲降下し、最下層で幅 ef の best-first 探索（訪問済み集合＋候補ヒープ）を行い上位 k を返す。計算量は経験的に O(log n) 級のホップ数×次数 M の距離計算で、1 クエリあたり数千回の距離計算が典型——この内側ループが SIMD 化の主対象である。
- **パラメータの意味**: M（次数）はグラフの密度＝メモリと頑健性、ef_construction は構築品質（低いと悪い辺が固定化する）、ef は実行時の再現率-レイテンシ交換。ef だけが実行時に動かせる点が運用上重要で、recall SLO の制御入力になる。

### 3. IVF と直積量子化 — 粗量子化 × 細量子化

- **IVF**: 学習フェーズで k-means により nlist 個（√n オーダが経験則）のセントロイドを得る。挿入は最近傍セルのリストへ追記、検索は nprobe セルの走査。セル境界近くの点が隣接セルに「隠れる」のが recall 損失の主因で、nprobe を増やして償う。
- **PQ の数理**: D 次元を m 個の D/m 次元部分空間に分解し、各部分空間で独立に 256 セントロイドを学習する。組み合わせ空間は 256^m 個の再構成点を張るが、記憶するのは m×256 個のセントロイドだけ——**直積 (product) 構造が指数的な符号帳を線形のメモリで表現する**。量子化誤差は部分空間間の相関を無視した分だけ増え、事前の回転（OPQ）で相関を減らせる。
- **ADC と SIMD**: クエリ 1 本につき m×256 の部分距離テーブルを作り、候補の距離を m 回の 8bit 添字テーブル参照の和で近似する。テーブルを 4bit 量子化してレジスタに載せ、シャッフル命令で 16/32 並列に引く高速化 (fast scan) が現代実装の標準で、メモリ帯域と演算のバランスが [`simd-vector.md`](../../hardware/cpu-architecture/simd-vector.md) の教科書的題材になっている。
- **再ランク**: PQ 近似距離で候補を広めに取り、完全精度ベクトル（RAM か SSD）で上位だけ再計算する二段構えが標準。近似は「順位の粗選別」にだけ使い、最終順位は正確に——語彙検索の二段階検索と同じ役割分担である。
- **量子化の現在**: 8bit/4bit スカラ量子化や 1bit（二値）量子化＋再ランクのような単純な方式が、埋め込み側の頑健化と再ランクの併用で復権している。圧縮方式の選択は「再ランク前提でどこまで粗くできるか」の交渉になった。

### 4. スケールと更新 — 索引はサービスになる

- **シャーディング**: ベクトル ID または埋め込み空間のクラスタでシャードに分け、全シャード検索→top-k 統合。深いページング・テール増幅の幾何学は語彙検索と共通だが、大域統計問題（IDF 相当）が無い分だけ統合は単純である。
- **鮮度**: 静的構築が得意な索引（IVF, DiskANN）は「不変のベース索引＋新着を受ける小さな追記層（総当たりか小 HNSW）＋背景での再構築・統合」という構成を取る——LSM 的解法への収束（発見事項 7）。削除は墓標フィルタで隠し、再構築で物理削除する。
- **複製と一貫性**: upsert の複製は通常のレプリケーションだが、「書いた直後に検索で見える」保証は追記層の可視性設計に依存し、語彙検索の refresh と同じく**準リアルタイム**が既定の意味論になる。強い読み取り一貫性を要求するなら追記層を同期複製に載せる必要があり、コストは [`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) の議論の通りである。
- **GPU**: 距離計算の稠密さは GPU と好相性で、構築（k-means, グラフ構築）と大バッチ検索は GPU 実装（Faiss-GPU, CAGRA 系グラフ）が桁の高速化を出す。低レイテンシ単発クエリは CPU、高スループットバッチと構築は GPU という分担が現在の均衡である（[`../ml-systems/ml-inference-systems.md`](../ml-systems/ml-inference-systems.md) のバッチング議論と同型）。

### 5. ハイブリッド検索 — 語彙とベクトルの合成

- 語彙検索（BM25）は固有名詞・型番・厳密句に強く、ベクトル検索は言い換え・多言語・曖昧クエリに強い——誤りモードが直交に近いことが合成の根拠である。
- **RRF (reciprocal rank fusion)**: 各検索系の順位 r から Σ 1/(κ + r)（κ ≈ 60）を合算する。スコアの尺度が異なる系（BM25 の点数と内積）を**順位だけで**融合するため校正が不要で、頑健な既定解になっている。スコアの線形結合は尺度合わせ（正規化・重み学習）を要するが、上限性能は高い。
- 融合後に交差エンコーダ等のリランカを載せる三段構成（BM25/ANN → RRF → rerank)が RAG の標準形であり、各段の役割は「再現率の確保 → 粗い統合 → 精密な順序付け」と純化される。

## セキュリティ考察 (Security Considerations)

- **埋め込みはハッシュではなく、データそのものである**: 埋め込みからの原文復元（embedding inversion）は現実的な攻撃で、短いテキストならかなりの割合のトークンが復元できることが示されている。「ベクトルだけなら預けても安全」という直観は誤りであり、ベクトルストアは原文と同じ機密度で保護する——暗号化・アクセス制御・保持期限のすべてが原文と同等に要る。
- **RAG は間接プロンプトインジェクションの配管である**: 検索結果は LLM の文脈に入る。攻撃者が索引に文書を投入できるなら（UGC、クロール、共有ドライブ）、「この指示に従え」を埋めた文書を検索で釣り上げさせることで、モデルへの指示注入が成立する。埋め込み空間で「よく検索される」ように最適化した毒文書は語彙的なスパムより検出しにくい。防御は投入経路の認可、検索結果の出所表示と信頼度階層、LLM 側の指示/データ分離であり、索引汚染（[`inverted-index-fulltext-search.md`](inverted-index-fulltext-search.md) セキュリティ考察）の LLM 時代版として一段深刻である。
- **アクセス制御は検索時に効かせる**: 認可はベクトル索引のフィルタとして検索時に適用しなければならない——生成後の後段フィルタでは、スコア・件数・LLM の応答に「見えないはずの文書」が漏れ込む。フィルタ付き ANN の機構（発見事項 7)がそのままセキュリティ機構になる構図で、テナント分離は「共有索引＋フィルタ」か「テナント毎索引」かの選択になり、フィルタの実装バグが即座に越境閲覧になる前者はより高い検証水準を要する。
- **メンバーシップ推論と統計的漏えい**: あるデータが索引に入っているかを、類似検索の結果分布から推定する攻撃面がある。公開 API に距離スコアを生で返すことは、空間内の点配置という内部状態の開示であり、必要がなければ順位のみ返す。
- **可用性**: 高次元総当たりへのフォールバック（低選択率フィルタ時）や巨大 k の要求は CPU・帯域を占有できる。クエリ毎の計算量上限とフォールバック経路の予算化は、語彙検索のサーキットブレーカと同じ発想の防御である。

## 結論 (Conclusions)

- 高次元では厳密最近傍が距離の集中により実質不可能になり、ANN は厳密性を recall という測定可能な品質に置き換えて速度を取り戻す。近似は欠陥ではなく、この問題設定で実用系を成立させる唯一の通貨である。
- 主要な設計族はメモリ階層への配置戦略として整理できる——HNSW は RAM のランダムアクセス、IVF は帯域、PQ はキャッシュとレジスタ、DiskANN は SSD。再現率ノブ (ef/nprobe/ビーム幅) がどれも「どこまで読むか」の制御である点で、ANN もまた「読まない工学」である。
- recall@k は保証されない。データ・クエリ分布の変化で静かに劣化するため、厳密解との突き合わせによる継続監視を SLO として運用に組み込むことが、ANN をプロダクションに置く条件である。
- フィルタ・更新・複製・認可という要求が ANN 索引をデータベース化し、不変ベース＋追記層＋背景再構築という LSM 型の構成に収束させている。ベクトル検索は「アルゴリズム」から「ストレージエンジンの一種目」へ移行した。
- 語彙検索とベクトル検索は誤りモードが直交するため合成が正当化され、RRF＋リランカの三段構成が標準形になった。そして埋め込みはデータそのものであり、ベクトルストアの機密度・認可・投入経路の検証は原文と同じ厳しさで扱う必要がある。

### 次の接続先 (Related Topics)

- [`inverted-index-fulltext-search.md`](inverted-index-fulltext-search.md) — 合成相手である語彙検索の機構と、scatter-gather・準リアルタイムという共通の幾何学。
- [`../ml-systems/ml-inference-systems.md`](../ml-systems/ml-inference-systems.md) — 埋め込み・リランカを供給する推論基盤とバッチングの経済。
- [`../databases/storage-engines.md`](../databases/storage-engines.md) — 追記層＋背景再構築が収束していく先の LSM/WAL 機構。
- [`../../hardware/cpu-architecture/simd-vector.md`](../../hardware/cpu-architecture/simd-vector.md) — 距離計算・PQ fast scan を駆動する SIMD。
- [`../../hardware/cpu-architecture/cache-memory-hierarchy.md`](../../hardware/cpu-architecture/cache-memory-hierarchy.md) / [`../../hardware/hw-sw-interface/storage-nvm-interface.md`](../../hardware/hw-sw-interface/storage-nvm-interface.md) — 設計族が配置戦略を立てる対象の記憶階層。
- [`../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md`](../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md) / [`svd-low-rank.md`](../../foundations/mathematics/linear-algebra/svd-low-rank.md) — 内積・ノルム・低ランク近似というベクトル検索の数学的土台。

## 参考文献 (References)

**一次資料 (Primary — 原論文)**

- Indyk, P., & Motwani, R. (1998). Approximate Nearest Neighbors: Towards Removing the Curse of Dimensionality. *STOC '98*. DOI: 10.1145/276698.276876 — ANN 問題の定式化と LSH、「厳密を諦める」路線の原典。
- Beyer, K., Goldstein, J., Ramakrishnan, R., & Shaft, U. (1999). When Is "Nearest Neighbor" Meaningful? *ICDT '99*. DOI: 10.1007/3-540-49257-7_15 — 距離の集中（次元の呪いの実体）の原典。
- Jégou, H., Douze, M., & Schmid, C. (2011). Product Quantization for Nearest Neighbor Search. *IEEE TPAMI*, 33(1), 117–128. DOI: 10.1109/TPAMI.2010.57 — PQ と非対称距離計算 (ADC) の原典。
- Malkov, Y. A., & Yashunin, D. A. (2020). Efficient and Robust Approximate Nearest Neighbor Search Using Hierarchical Navigable Small World Graphs. *IEEE TPAMI*, 42(4), 824–836. DOI: 10.1109/TPAMI.2018.2889473 — HNSW の原典。
- Subramanya, S. J., Devvrit, F., Simhadri, H. V., Krishnaswamy, R., & Kadekodi, R. (2019). DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node. *NeurIPS '19*. — SSD 常駐グラフ索引 (Vamana) の原典。
- Johnson, J., Douze, M., & Jégou, H. (2019). Billion-scale Similarity Search with GPUs. *IEEE Transactions on Big Data*, 7(3), 535–547. DOI: 10.1109/TBDATA.2019.2921572 — Faiss と GPU 化の一次報告。
- Aumüller, M., Bernhardsson, E., & Faithfull, A. (2020). ANN-Benchmarks: A Benchmarking Tool for Approximate Nearest Neighbor Algorithms. *Information Systems*, 87. DOI: 10.1016/j.is.2019.02.006 — recall-QPS 曲線による比較方法論の標準。
- Cormack, G. V., Clarke, C. L. A., & Buettcher, S. (2009). Reciprocal Rank Fusion Outperforms Condorcet and Individual Rank Learning Methods. *SIGIR '09*. DOI: 10.1145/1571941.1572114 — RRF の原典。
- Morris, J. X., Kuleshov, V., Shmatikov, V., & Rush, A. M. (2023). Text Embeddings Reveal (Almost) As Much As Text. *EMNLP '23*. DOI: 10.18653/v1/2023.emnlp-main.765 — 埋め込み逆転（原文復元）攻撃の実証。
