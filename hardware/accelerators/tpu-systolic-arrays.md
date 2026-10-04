# TPU とシストリックアレイ (TPU & Systolic Arrays)

## 概要 (Overview)

`gpu-simt-architecture.md` は、GPU が「超多スレッドでメモリレイテンシを隠す」汎用スループットマシンであることを見た。本ドキュメントはその対極にあるドメイン特化アーキテクチャ (domain-specific architecture, DSA) を、Google **TPU (Tensor Processing Unit)** とその心臓部である**シストリックアレイ (systolic array)** を題材に分析する。DSA は、**演算の種類を行列積 (GEMM) にほぼ固定する代わりに、データ移動のエネルギーを極小化する**。

問題の根は算術にある: 現代のシリコンでは、**演算そのものより「オペランドをレジスタ・SRAM・DRAM から運ぶこと」のほうが 1〜3 桁高くつく**[^horowitz]。ゆえに性能/電力の勝負は「速く計算する」ではなく「一度読んだ値を何回使い回すか」で決まり、シストリックアレイは「データが固定された演算器の格子を脈動的に流れ、各値が通過するたびに再利用される」という構造でこの問いに答える。Dennard スケーリング終焉後の「アーキテクチャの黄金時代」[^golden-age]の旗艦例であり、GPU が Tensor Core で同じ方向に収斂したことまで含めて、`../semiconductor-manufacturing/` の物理的制約と `../../software/ai/` のワークロードに挟まれた設計空間として読む。分析深度は Level 3。

## 方法論 (Methodology)

- 出発点をエネルギーの算術（演算 vs データ移動のピコジュール比較）に置き、すべての設計判断をそこから導出する。ベンダの語る「TOPS」ではなく **TOPS/W と実効利用率**を軸にする。
- 一次資料は TPU の査読論文（v1: ISCA 2017、v4: ISCA 2023）[^tpu-v1][^tpu-v4]を基点とし、ベンダ発表の目標値と論文の実測値を区別する。
- GPU との比較は優劣ではなく **「何を固定し、何を汎用に残したか」の選択の違い**として整理し、両者が現在収斂しつつある点を明示する。
- データフローの分類（weight-stationary 等）は Eyeriss の分類法[^eyeriss]に従う。

## 発見事項 (Findings)

1. **支配的なコストは演算ではなくデータ移動である。** 45nm 世代の代表値で、32 bit 浮動小数点乗算が数 pJ に対し、SRAM アクセスは数十 pJ、DRAM アクセスは数百 pJ（**演算の 100 倍超**）である[^horowitz]。行列積が「アクセラレータに最適な題材」なのは重要だからだけではなく、`O(n³)` の演算に対しデータが `O(n²)` しかない、つまり**構造的に再利用可能**だからである。この再利用可能性は [`../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md`](../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md) が計算構造の観点から述べた演算強度 (arithmetic intensity) の議論そのものである。
2. **シストリックアレイは「制御とメモリアクセスを演算器の格子で償却する」構造である。** 2 次元格子の各セルは乗加算器 (MAC) とレジスタのみを持ち、データは隣接セルへ 1 サイクルずつ流れる。汎用プロセッサが 1 演算ごとに払う命令フェッチ・デコード・レジスタファイルアクセスが存在せず、SRAM から読んだ値は格子を貫通する間に数百回使われる。Kung が 1982 年に「メモリ帯域がボトルネックになる計算のための構造」として定式化した設計[^kung]が、35 年後に深層学習という理想の顧客を得た。
3. **TPUv1 (2015 稼働) は「決定性とレイテンシ」が最初の勝因だった。** 256×256 の MXU（65,536 MAC、INT8）で 92 TOPS、同世代 CPU/GPU 比で 15〜30 倍の性能/電力を実測した[^tpu-v1]。見落とされがちな点として、論文はデータセンターの推論が**テールレイテンシ SLA に縛られる**ことを強調する。キャッシュ・投機・動的スケジューリングを持たない TPU の実行時間は決定的であり、SLA 下で高い利用率を保てた。`../../software/embedded/realtime-fundamentals.md` の「平均を売って最悪を買う」取引が、データセンターの推論でも勝つことを示した例である。
4. **データフローの選択（何を動かさず何を流すか）がアクセラレータの分類軸である。** 重み定常 (weight-stationary、TPU: 重みを格子に焼き付け活性値を流す)、出力定常 (output-stationary、部分和を動かさない)、行定常 (row-stationary、Eyeriss) は、どれも「最も高頻度に再利用されるものを最も近いメモリに固定する」の変種である[^eyeriss]。畳み込み・Transformer の attention など演算の形が変わると最適なデータフローも変わり、単一の正解はない。
5. **GPU と TPU は互いに向かって収斂した。** GPU は Volta 以降 **Tensor Core**（小型の行列積エンジン）を SIMT コアに埋め込み、FP16/BF16/FP8 と世代ごとに低精度化を進めた（「汎用機に行列エンジンを内蔵」）。TPU は世代を重ねて HBM・ベクタユニット・柔軟なインターコネクトを獲得した（「行列エンジンに汎用性を追加」）。2026 年時点の両者は「**行列エンジン + ベクタユニット + HBM + 専用インターコネクト**」という同じ構成の別実装であり、差は主にプログラミングモデル（CUDA の動的性 vs XLA の静的コンパイル）とエコシステムに残る。
6. **チップ単体ではなく「ポッド」が設計単位になった。** TPUv4 は 4,096 チップを専用インターコネクト (ICI) の 3D トーラスで結び、**光サーキットスイッチ (OCS)** でトポロジ自体を再構成可能にした[^tpu-v4]。故障ノードの迂回とジョブ形状への適合を、パケットスイッチではなく光の物理回線切替で行う。学習ワークロードの通信が予測可能（`interconnects-scaleout-training.md`）だからこそ成立する共設計であり、「アクセラレータの分析はクラスタの分析」へと単位が繰り上がったことを示す。
7. **数値形式そのものが設計変数である。** BF16（FP32 と同じ 8 bit 指数 + 7 bit 仮数）は「学習は指数レンジが命、仮数精度は削れる」という観測をハードウェア仕様にしたものであり、以後 FP8、さらに狭い形式へと「学習・推論が壊れない最小ビット数」の探索が続く。丸め・スケーリングの数値解析は [`../../foundations/mathematics/linear-algebra/matrix-factorizations-numerical.md`](../../foundations/mathematics/linear-algebra/matrix-factorizations-numerical.md) の浮動小数点の議論の直接の応用で、ソフトウェア側の対策（損失スケーリング等）は `../../software/ai/ml-training-systems.md` で扱う。

## アーキテクチャ (Architecture)

### シストリックアレイの動作（重み定常・3×3 の例）

```
重み W をあらかじめ格子に焼き付け（weight-stationary）、
活性値 A を左から時差で流し込み、部分和が下へ伝播する:

        A の行が 1 サイクルずつ遅れて左端から入る
        a13 a12 a11 →│ w11 │→│ w12 │→│ w13 │
        a23 a22 a21 →│ w21 │→│ w22 │→│ w23 │   各セル:
        a33 a32 a31 →│ w31 │→│ w32 │→│ w33 │   in→ MAC →右へ
                        ↓部分和↓        ↓        部分和は下へ
                      y_1     y_2     y_3
```

- SRAM アクセスは格子の縁でのみ発生し、内部はレジスタ間の隣接転送だけである。そのため**配線が最短で、かつ全セルが毎サイクル働く**。パイプラインが満ちるまでのウォームアップ（行列が小さいと利用率が落ちる）が弱点で、「大きい行列を切れ目なく供給する」責務がコンパイラ (XLA) とメモリ系に移る。

### TPU と GPU の設計比較

| 観点 | GPU (`gpu-simt-architecture.md`) | TPU |
|---|---|---|
| 基本戦略 | 超多スレッドでレイテンシ隠蔽 | 決定的データフローでレイテンシ自体を排除 |
| 演算の主体 | SIMT コア + Tensor Core | MXU（シストリックアレイ）+ ベクタユニット |
| 制御 | 動的スケジューラ・分岐対応 | ほぼ静的（コンパイラが全て決める） |
| メモリ | キャッシュ階層 + HBM | 大容量スクラッチパッド + HBM（キャッシュなし） |
| プログラミング | CUDA（動的形状・任意カーネル） | XLA（静的形状・グラフコンパイル） |
| 得意領域 | 多様なカーネル・研究の速度 | 大規模で形の揃った学習・推論 |
| スケール | NVLink/NVSwitch + InfiniBand | ICI 3D トーラス + OCS |

- スクラッチパッド（ソフトウェア管理 SRAM）とキャッシュ（ハードウェア管理）の対比は `../cpu/cache-memory-hierarchy.md` の議論の延長にある: アクセスパターンが完全に既知なら、キャッシュのタグ・一致検査のエネルギーすら無駄になる。

## 技術詳細 (Technical Details)

### 1. エネルギーの算術 (The Energy Arithmetic)

- 代表的な桁感（45nm、値はプロセス・設計で変動するが**比率**は頑健）[^horowitz]: 8 bit 整数加算 ~0.03 pJ、32 bit 浮動小数点乗算 ~4 pJ、32 bit SRAM 読み出し ~5 pJ（小容量）、32 bit DRAM 読み出し **~640 pJ**。
- 帰結: (1) 低精度化は演算エネルギーを超線形に下げる（乗算器面積は仮数幅の 2 乗に比例）、(2) DRAM にアクセスするたびに演算 100 回分を失うため、**オンチップ再利用がすべて**、(3) よって TOPS の比較はほぼ無意味で、「その TOPS に給餌できるメモリ階層か」が本体である。物理側の対は `../cpu/dram-memory-controllers.md` と `../semiconductor-manufacturing/semiconductor-fabrication.md`（HBM の 2.5D 実装）である。
- ルーフライン (roofline) モデルで言えば、DSA の設計とは「ワークロードの演算強度に合わせて屋根の折れ点を配置する」ことに等しい。Transformer の GEMM は高強度側（演算律速）、推論の decode は低強度側（帯域律速、`../../software/ai/ml-inference-systems.md`）に落ちるため、同じチップでも局面で律速が入れ替わる。

### 2. TPU 世代の進化 — 何が汎用化されたか (Generational Evolution)

- **v1**（推論専用・INT8・DDR3）: PCIe 接続のコプロセッサ。ホスト経由の給餌が律速で、MXU 利用率は平均 20% 前後とワークロード依存が大きかった[^tpu-v1]。「演算器は安く、給餌は高い」ことの初期実証である。
- **v2/v3**（学習対応）: BF16 の導入、HBM 化、チップ間 ICI でポッド化。学習は活性値の保存・勾配の通信を伴うため、メモリ容量と相互接続が一気に主役になった。
- **v4 以降**: 4,096 チップ 3D トーラス + OCS[^tpu-v4]。SparseCore（埋め込み表の疎アクセス専用ユニット）の搭載は、「GEMM 以外の 2 割」である疎な gather/scatter が全体を律速し始めたことへの応答であり、Amdahl の法則が DSA にも容赦なく働くことを示す。
- 一貫した教訓: **専用化の対象は「演算」から「メモリアクセスパターン」そして「クラスタトポロジ」へと外側に移動**してきた。

### 3. コンパイラという前提 (The Compiler Contract)

- シストリックアレイは自ら何も判断しない。タイル分割・データ搬入のスケジュール・二重バッファリングはすべて **XLA** が静的に決め、静的形状 (static shapes) を要求する。この「ハードウェアの単純化はコンパイラへの請求書になる」という構図は `../../software/compilers-runtimes/` の主題であり、ML コンパイラの各論（XLA/Triton）は `../../software/ai/` のロードマップに置く。
- GPU 側も Tensor Core の性能はコンパイラ・ライブラリ（cuBLAS/CUTLASS、カーネル融合）に依存しており、「DSA の性能はソフトウェアスタックの成熟で決まる」点で両者は同じ制約下にある。

## セキュリティ考察 (Security Considerations)

- **マルチテナントの隔離が新しい攻撃面である。** クラウドのアクセラレータは複数顧客で共有され（時分割・MIG 型の空間分割）、GPU のコンテキスト切替時のメモリ消去漏れ・共有キャッシュ経由のサイドチャネルが研究で示されてきた。TEE のアクセラレータ拡張（GPU の confidential computing モード等）は `../security/side-channels-tee-secure-boot.md` の信頼実行環境をデバイス側へ広げる動きであり、DMA 経路の保護は `../hw-sw-interface/mmio-dma-interrupts.md` の IOMMU が基礎になる。
- **決定的実行はサイドチャネルを狭める副作用を持つ。** TPU 型の静的スケジューリングは、実行時間が入力データにほぼ依存しないため、タイミングチャネルの分散が小さい。リアルタイム性（発見事項 3）と同様、決定性がセキュリティにも配当を払う例である。
- **モデル重みは最高価値の資産である。** アクセラレータのメモリ・チェックポイント経路・テレメトリは「数億ドルの学習成果物」への到達経路であり、抽出・改竄への防御（暗号化・完全性検証・アクセス監査）は `../../software/security/software-supply-chain-security.md` のモデル来歴の議論と接続する。
- **カスタムシリコンのサプライチェーン**: 自社設計チップは `../semiconductor-manufacturing/` の製造委託を経由し、ハードウェアトロイ・改竄検出は `../security/` のロードマップ項目（サプライチェーン）の実例になる。

## 結論 (Conclusions)

シストリックアレイと TPU の物語は、「エネルギーの算術（データ移動 ≫ 演算）を認めれば、アーキテクチャは再利用の幾何学になる」ことを示す。TPU は演算の種類を固定して制御とメモリアクセスを償却し、GPU は汎用性を保ちながら行列エンジンを内蔵し、両者は同じ構成に収斂した。そして専用化の前線は演算器からメモリ、さらに**クラスタのトポロジと光**へ移った。チップ単体の分析はもはや半分でしかない。

### 続編への橋渡し

残り半分、つまり数千チップを結ぶインターコネクトと集合通信は [`interconnects-scaleout-training.md`](interconnects-scaleout-training.md) で扱う。TPU の 3D トーラス + OCS と、GPU 陣営の NVLink + InfiniBand/RoCE は、同じ学習トラフィックへの対照的な解である。

---

## 参考文献 (References)

[^horowitz]: Horowitz, M. (2014). Computing's Energy Problem (and what we can do about it). *Proc. IEEE ISSCC 2014*, 10–14. DOI: 10.1109/ISSCC.2014.6757323 — 演算・SRAM・DRAM のエネルギー比較の標準的典拠。

[^kung]: Kung, H. T. (1982). Why Systolic Architectures? *IEEE Computer*, 15(1), 37–46. DOI: 10.1109/MC.1982.1653825 — シストリックアレイの動機（メモリ帯域ボトルネック）の原典。

[^tpu-v1]: Jouppi, N. P. et al. (2017). In-Datacenter Performance Analysis of a Tensor Processing Unit. *Proc. ACM/IEEE ISCA 2017*, 1–12. DOI: 10.1145/3079856.3080246 — TPUv1 の実測性能・利用率・設計判断の一次資料。

[^tpu-v4]: Jouppi, N. P. et al. (2023). TPU v4: An Optically Reconfigurable Supercomputer for Machine Learning with Hardware Support for Embeddings. *Proc. ACM/IEEE ISCA 2023*. DOI: 10.1145/3579371.3589350 — OCS・3D トーラス・SparseCore の一次資料。

[^eyeriss]: Chen, Y.-H., Emer, J. & Sze, V. (2016). Eyeriss: A Spatial Architecture for Energy-Efficient Dataflow for Convolutional Neural Networks. *Proc. ACM/IEEE ISCA 2016*, 367–379. DOI: 10.1109/ISCA.2016.40 — データフロー分類（weight/output/row-stationary）の典拠。

[^golden-age]: Hennessy, J. L. & Patterson, D. A. (2019). A New Golden Age for Computer Architecture. *Communications of the ACM*, 62(2), 48–60. DOI: 10.1145/3282307 — Dennard 終焉と DSA への転換の総説（Turing 賞講演）。

その他の標準的典拠を以下に挙げる。

- Sze, V., Chen, Y.-H., Yang, T.-J. & Emer, J. (2020). *Efficient Processing of Deep Neural Networks*. Morgan & Claypool. — DNN アクセラレータ設計の体系的教科書。
- Williams, S., Waterman, A. & Patterson, D. (2009). Roofline: An Insightful Visual Performance Model for Multicore Architectures. *Communications of the ACM*, 52(4), 65–76. DOI: 10.1145/1498765.1498785 — ルーフラインモデルの原典。
