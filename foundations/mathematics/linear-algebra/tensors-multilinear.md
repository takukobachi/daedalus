# テンソルと多線形代数 (Tensors and Multilinear Algebra)

## 概要 (Overview)

線形代数シリーズの第5回(最終回)。「テンソル」という語は文脈で意味が揺れる——物理では座標変換に対する振る舞いで定義される幾何的対象、機械学習フレームワークでは単なる**多次元配列 (multi-dimensional array)** を指す。本回はこの2つを1本の線で結ぶ: 数学的実体は**多線形写像 (multilinear map)**——複数のベクトルを引数に取り、各引数について線形な写像——であり、基底を選ぶとその成分表示として多次元配列が現れる。「行列=線形写像の表現」(第1回)の多引数版である。

この視点の実用的な収穫は**縮約 (contraction)** という統一概念にある。行列積・内積・トレース・バッチ処理・注意機構 (attention) の計算は、すべて「添字を揃えて掛けて足す」縮約の一種であり、einsum 記法で1行に書ける。GPU の Tensor Core が加速するのも、ML フレームワークが最適化するのも、結局はこの縮約の実行順序とメモリ配置である。最後に SVD(第3回)の高次元一般化であるテンソル分解に触れ、シリーズを閉じる。

## 前提知識 (Prerequisites)

- [`vectors-matrices-linear-maps.md`](vectors-matrices-linear-maps.md) — 線形写像と行列表現、行列積の計算構造
- [`svd-low-rank.md`](svd-low-rank.md) — 低ランク近似(テンソル分解で一般化する)

## 本論 (Main Content)

### 1. 多線形写像としてのテンソル (tensors as multilinear maps)

双線形形式 `f(x, y)`——2つのベクトルを取り、各引数について線形な実数値関数——を考える。基底を選ぶと `f(x, y) = Σᵢⱼ Aᵢⱼ xᵢ yⱼ` と成分で書け、係数の集まり A は2次元配列(行列)になる。引数を k 個に増やしたものが **k 階テンソル (order-k tensor)** であり、成分は k 次元配列 `T_{i₁i₂⋯i_k}` になる (Kolda & Bader 2009, §2)。

```text
0階 = スカラー、1階 = ベクトル、2階 = 行列、3階以上 = 高階テンソル
「階数 (order)」= 添字の本数 (ML では軸数・次元数と呼ばれる)
```

物理のテンソル(応力テンソル、慣性テンソル)は「基底を取り替えたとき成分が特定の規則で変換される」ことまで要求するが、その変換規則は多線形写像の定義から自動的に従う——「テンソルは配列であって配列でない」という言い回しの正体は、**配列は表現であり、実体は基底に依らない写像**という、第1回と同じ区別である。

用語の注意を1つ: テンソルの「ランク」は多義語で、(i) 添字の本数(= 階数)の意味と、(ii) ランク1テンソルの和としての最小項数(行列の階数の一般化、§4)の意味がある。本文書では (i) を階数、(ii) をランクと書き分ける。

### 2. テンソル積と成分表示 (tensor product and components)

ベクトル u, v から作る**外積 (outer product)** `(u⊗v)ᵢⱼ = uᵢvⱼ` は、SVD の回で登場した「ランク1行列」の正体である。一般に k 本のベクトルのテンソル積 `u⁽¹⁾⊗⋯⊗u⁽ᵏ⁾` が**ランク1テンソル**であり、任意のテンソルはランク1テンソルの線形結合で書ける——基底ベクトル同士のテンソル積 `eᵢ₁⊗⋯⊗eᵢ_k` が k 階テンソル全体の空間の基底になるからである。次元は各軸の次元の積 `n₁n₂⋯n_k` で、階数に対して指数的に増える。この「添字を1本増やすと容量が掛け算で効く」性質は、後述のメモリレイアウトとテンソル分解の両方の背景になる。

### 3. 縮約と einsum 記法 (contraction and einsum notation)

テンソル計算の基本演算は**縮約**——2つのテンソルの添字を1つ揃え、その添字について掛けて足す——である。アインシュタインの縮約記法(繰り返し添字は総和)を使うと、主要な演算がすべて同じ形に並ぶ。

```text
内積          c = uᵢvᵢ                    (ij,ij→)
行列×ベクトル  yᵢ = Aᵢⱼxⱼ                  (ij,j→i)
行列積        Cᵢⱼ = AᵢₖBₖⱼ                (ik,kj→ij)
トレース      t = Aᵢᵢ                     (ii→)
バッチ行列積   C_bij = A_bik B_bkj         (bik,bkj→bij)
注意スコア     S_bhij = Q_bhid K_bhjd      (bhid,bhjd→bhij)
```

NumPy / PyTorch の `einsum` はこの記法をそのまま API にしたもので、右列がその指定文字列である。見どころは2つ。第一に、**行列積は縮約の特別な場合にすぎず、バッチ化とは「縮約に関与しない添字を増やすこと」** である——ML フレームワークの計算グラフが最終的にバッチ行列積 (batched GEMM) の列に還元されるのは、この構造による。第二に、多重縮約は**実行順序で計算量が桁で変わる**(行列3つの積 `(AB)C` vs `A(BC)` の一般化)。einsum の最適化・計算グラフのオペレータ融合は、この縮約順序問題(NP困難だが実用ヒューリスティクスが効く)を解いている (Smith & Gray 2018)。

### 4. 多次元配列の実装 — レイアウトとビュー (memory layout and views)

フレームワークの「テンソル」の実体は、**連続メモリ + 形状 (shape) + ストライド (strides)** の3つ組である。ストライドは「添字 i を1進めるとメモリを何要素飛ぶか」を軸ごとに持つ。

```text
形状 (2,3,4) の行優先 (row-major) 配列 → ストライド (12,4,1)
transpose / 一部の reshape / スライス = ストライドの書き換えのみ (ビュー、コピーなし)
非連続ビューへの演算 / 連続化 (contiguous) = 実データのコピー
```

「reshape はタダ、transpose もタダ、ただし直後の GEMM が非連続データを嫌ってコピーが走る」という ML 実務の性能あるあるは、この3つ組を知っていれば予測できる。畳み込みの NCHW vs NHWC 選択(どの軸をメモリ連続にするか)も同型の判断であり、**同じ数学的テンソルに対する物理レイアウトの選択**が、キャッシュ・SIMD・Tensor Core の効率を決める——第1回 §4「行列積の計算構造」の高階版である。

### 5. テンソル分解 — SVD の高次元化 (tensor decompositions)

行列の SVD に相当する「高階テンソルの低ランク表現」には標準形が2つある (Kolda & Bader 2009)。

```text
CP 分解:     T ≈ Σᵣ λᵣ · aᵣ⊗bᵣ⊗cᵣ        (ランク1テンソル R 個の和)
Tucker 分解: T ≈ G ×₁ A ×₂ B ×₃ C        (小さなコアテンソル G を各軸で変換)
```

保存量は元の `n³` に対し CP で `3nR`、Tucker で `r³ + 3nr`——特異値の減衰(第3回)と同じく、データに低ランク構造があれば劇的に縮む。ただし行列と違い、高階では最良ランク R 近似の存在すら保証されず、計算も NP 困難——Eckart–Young のような綺麗な定理は失われる、という限界も含めて第3回との対比で押さえておく。応用としてはニューラルネット層の圧縮(畳み込み核の CP 分解)、多次元データの解析のほか、「軸を増やして畳んで縮約する」テンソルネットワークが量子多体系・量子回路シミュレーションの主力表現になっている。

## 工学への接続 (Engineering Connections)

- **Tensor Core の演算単位**: Tensor Core が実行するのは小さな行列積タイル (例: 4×4×4 の積和) であり、フレームワークは高階テンソルの縮約をバッチ GEMM に還元 (§3)、レイアウト変換 (§4) でタイルに整列させてから投入する。「テンソル」計算の加速器が実際には行列積エンジンである理由は、縮約の還元構造そのものである。
- **オペレータ融合と縮約順序**: 計算グラフ最適化 (XLA, TorchInductor) の主要な仕事は、縮約列の実行順序と中間テンソルの実体化判断 (§3, §4)——数学的に等価な式から計算量・メモリの異なる実装を選ぶ、線形代数第4回 §5 と同じ構図の高階版。
- **量子回路シミュレーション**: n 量子ビットの状態は 2ⁿ 成分の n 階テンソルであり、ゲート適用は縮約。テンソルネットワークの縮約順序最適化が古典シミュレーション可能性の境界を決める——「指数的な対象を低ランク構造で押し込む」§5 の最前線。
- **記法としての einsum**: カスタムカーネルの仕様、論文の式、フレームワーク実装の三者を1つの文字列で突き合わせられる——添字記法は人間とコンパイラの共通言語として機能する。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/gpu-accelerators/gpu-simt-architecture.md`](../../../hardware/gpu-accelerators/gpu-simt-architecture.md) | Tensor Core のタイル行列積への縮約の還元 (§3)、レイアウト整列 (§4) |
| [`hardware/cpu-architecture/simd-vector.md`](../../../hardware/cpu-architecture/simd-vector.md) | ストライドとメモリ連続性 (§4) が SIMD ロードの効率を決める——ギャザー/スキャッタとユニットストライドの差 |
| [`software/ml-systems/ml-inference-systems.md`](../../../software/ml-systems/ml-inference-systems.md) | 推論グラフの実体はバッチ GEMM 列 (§3)。オペレータ融合・レイアウト選択・注意機構の縮約構造 |
| [`hardware/cpu-architecture/cache-memory-hierarchy.md`](../../../hardware/cpu-architecture/cache-memory-hierarchy.md) | 高階テンソルの巡回順序=多重ループのブロック化。キャッシュ効率の議論の一般形 |

## 参考文献 (References)

- Kolda, T. G., & Bader, B. W. (2009). Tensor decompositions and applications. *SIAM Review*, 51(3), 455–500. DOI: 10.1137/07070111X — テンソルの定義・CP/Tucker 分解の主参照
- Golub, G. H., & Van Loan, C. F. (2013). *Matrix Computations* (4th ed.). Johns Hopkins University Press. — 行列側の対応物(SVD・ブロック化)の参照
- Smith, D. G. A., & Gray, J. (2018). opt_einsum — A Python package for optimizing contraction order for einsum-like expressions. *Journal of Open Source Software*, 3(26), 753. DOI: 10.21105/joss.00753 — 縮約順序最適化の実装側参照
- Hackbusch, W. (2019). *Tensor Spaces and Numerical Tensor Calculus* (2nd ed.). Springer. DOI: 10.1007/978-3-030-35554-8 — 多線形代数の数学的基礎の主参照
