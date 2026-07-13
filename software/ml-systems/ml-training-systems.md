# ML 学習システム (ML Training Systems)

## 概要 (Overview)

`ml-inference-systems.md` が扱った推論は「1 リクエストを速く・安く」の世界だった。学習 (training) は逆で、**数千〜数万アクセラレータを 1 つの同期計算として数週間〜数ヶ月回し続ける**世界である。支配する制約も入れ替わる: 推論の主敵がメモリ帯域と動的バッチングだったのに対し、学習の主敵は**メモリ容量**（モデル状態が 1 チップに載らない）、**通信と同期**（毎ステップ全員で勾配を揃える）、そして**故障**（万台規模では数時間ごとに何かが壊れる）である。

本ドキュメントは、大規模学習システムを 4 つの問いに分解する: (1) 載らないモデルをどう**切って配る**か（データ/テンソル/パイプライン並列と ZeRO/FSDP）、(2) 通信をどう**計算の影に隠す**か、(3) 数値をどう**壊さず削る**か（混合精度）、(4) 故障とどう**共存する**か（チェックポイントと可用性）。ハードウェア側の対——チップは [`../../hardware/gpu-accelerators/`](../../hardware/gpu-accelerators/)、配管は [`interconnects-scaleout-training.md`](../../hardware/gpu-accelerators/interconnects-scaleout-training.md)——と対で読む、AI スタックのソフトウェア側最上層である。分析深度は Level 3。

## 方法論 (Methodology)

- すべての並列化戦略を**「何を分割するか（データ / 層内の行列 / 層の列）× 何を通信するか（勾配 / activations）」**の 2 軸に還元し、通信量の第一原理計算で比較する。フレームワーク名は実装例として従属的に扱う。
- メモリは**バイト数の会計**として厳密に扱う（パラメータあたり何バイトが、どの瞬間に、どこに存在するか）。「大きいから切る」ではなく「この項がこの係数で効くからこの分割が要る」を導く。
- 効率は **MFU (model FLOPs utilization)**——理論ピーク FLOPs に対するモデル演算の実効比率——を共通指標とし、公開された実測値（PaLM・Llama 3 等の一次報告）に紐づける[^palm][^llama3]。
- 故障・運用の数値は一次報告（Llama 3 の中断統計）を用い、逸話ではなく設計入力として扱う。

## 発見事項 (Findings)

1. **最初に溢れるのは計算ではなくメモリであり、犯人は optimizer である。** Adam + 混合精度の標準構成では、パラメータあたり**約 16 バイト**のモデル状態——FP16/BF16 のパラメータ (2B) と勾配 (2B)、FP32 のマスターパラメータ (4B)・モーメント (4B)・分散 (4B)——を保持する[^zero]。10B パラメータで 160 GB、これだけで単一チップの HBM を超える。さらに activations（系列長 × バッチ × 隠れ次元 × 層数に比例）が乗る。**学習システム設計はこのバイト会計から始まる。**
2. **ZeRO/FSDP の洞察は「複製をやめて分割し、必要な瞬間だけ集める」である。** 素朴なデータ並列は全チップにモデル状態の完全な複製を持つ。ZeRO は optimizer 状態 (Stage 1)・勾配 (2)・パラメータ (3) を段階的にチップ間で分割し、パラメータは各層の計算直前に all-gather で一時復元する[^zero]。メモリは台数分の 1 に近づき、通信は定数倍（Stage 3 で素朴 DP の約 1.5 倍）しか増えない——**容量を通信で買う**取引であり、`interconnects-scaleout-training.md` の帯域があって初めて成立する。
3. **3D 並列は「通信の性質をインターコネクト階層に合わせて配置する」設計である。** テンソル並列（層内の行列を分割、**毎層** activations を all-reduce——Megatron の列/行分割は 1 層あたり前後 2 回）[^megatron]は超高帯域のスケールアップ領域にしか置けない。パイプライン並列はステージ境界の activations を点対点で送るだけで安いが、**バブル**（パイプラインの充填・排出で遊ぶ時間、比率 ≈ (p−1)/(m+p−1)、p: ステージ数、m: マイクロバッチ数）を払う[^gpipe]。データ並列はステップ末の勾配 all-reduce のみで、重畳可能。この 3 つの直積が「TP はノード内・PP は近接ノード・DP は全域」という標準配置に落ちる。
4. **重畳 (overlap) がスループットの生死を分ける。** 逆伝播は出力側の層から勾配を確定させるため、確定した層からバケット単位で all-reduce を発行すれば通信は計算の影に隠れる。逆に TP の all-reduce は次の計算の入力依存なので隠せない——**「隠せる通信は無料、隠せない通信が課金される」**。MFU が 30〜50% 台に留まる主因は、この隠せない通信とパイプラインバブル、そしてカーネル間の隙間である（PaLM: 46%前後の MFU を報告[^palm]）。
5. **混合精度は数値解析の応用工学である。** BF16/FP16 で計算・保存しつつ FP32 のマスターパラメータに累積する構成は、丸め誤差の蓄積（小さい勾配 × 多数回の加算）への防御であり、FP16 では勾配のアンダーフローを防ぐ**損失スケーリング**が要る[^amp]。縮約の順序で結果が変わる浮動小数点の非結合性（[`../../foundations/mathematics/linear-algebra/matrix-factorizations-numerical.md`](../../foundations/mathematics/linear-algebra/matrix-factorizations-numerical.md)）は、並列縮約と再現性の緊張として学習システムに現れる。FP8 以降はスケーリング係数の動的管理がランタイムの仕事になり、数値形式とシステムの共進化（`../../hardware/gpu-accelerators/tpu-systolic-arrays.md`）が続く。
6. **activations は「保存する」ものから「作り直す」ものになった。** 逆伝播に必要な中間出力を全部保持すると容量が爆発するため、チェックポイント地点だけ保存して逆伝播時に区間を再計算する **activation recomputation** が標準になった[^recompute]——計算を約 1/3 増やしてメモリを桁で削る、「FLOPs は余っていてバイトが足りない」時代を象徴する取引である。
7. **万台規模の学習は可用性エンジニアリングである。** Llama 3 は 16,384 GPU・54 日で 419 回の予期しない中断（78% がハードウェア起因）を報告し、それでも 90% 超の実効稼働時間を達成した[^llama3]。鍵は高頻度・非同期のチェックポイント（GPU から先に退避しバックグラウンドで永続化）、故障の自動検知・切り分け・予備への再スケジュール、そして**goodput**（やり直しに消えなかった有効計算の比率）という指標で全体を管理することにある。

## アーキテクチャ (Architecture)

### メモリの会計 — どのバイトが、いつ、どこに

```
パラメータ数 Ψ の Adam + BF16 混合精度、素朴なデータ並列の 1 チップ分:

  モデル状態（常駐, 16Ψ バイト）
    ├ BF16 パラメータ   2Ψ ┐
    ├ BF16 勾配        2Ψ │ ZeRO Stage 3 で分割
    ├ FP32 マスター     4Ψ │ ZeRO Stage 1〜 で分割
    ├ FP32 モーメント m  4Ψ │ ZeRO Stage 1〜 で分割
    └ FP32 分散 v      4Ψ ┘
  activations（ステップ内で増減）
    ∝ バッチ × 系列長 × 隠れ次元 × 層数   → recomputation で削減
  一時バッファ（通信・ワークスペース）
```

### 3D 並列の配置とデータフロー

```mermaid
flowchart TB
    subgraph dp["データ並列グループ（スケールアウト全域）"]
        subgraph pp1["PP ステージ 1（ノード群 A）"]
            TP1[TP グループ<br/>GPU 0-7: 層 1..L/2 を列/行分割<br/>毎層 all-reduce（NVLink 内）]
        end
        subgraph pp2["PP ステージ 2（ノード群 B）"]
            TP2[TP グループ<br/>GPU 8-15: 層 L/2..L]
        end
        TP1 -->|activations 点対点| TP2
    end
    TP1 & TP2 -.->|勾配 all-reduce<br/>backward と重畳（IB/RoCE）.-> TP1
    D[データローダ / 前処理<br/>ホスト側パイプライン] --> TP1
```

| 戦略 | 分割対象 | 通信 | 頻度 | 置き場所 |
|---|---|---|---|---|
| データ並列 / ZeRO | データ（+ モデル状態の分割） | 勾配 reduce-scatter / param all-gather | ステップ末（重畳可） | 全域 |
| テンソル並列 | 層内の重み行列 | activations all-reduce | **毎層**（隠せない） | NVLink ドメイン内 |
| パイプライン並列 | 層の列 | 境界 activations 点対点 | マイクロバッチごと | 近接ノード |
| シーケンス/コンテキスト並列 | 系列次元 | attention の交換 | 毎層 | 長系列時に追加 |

## 技術詳細 (Technical Details)

### 1. パイプラインのスケジューリング (Pipeline Schedules)

- **GPipe 型**: バッチを m 個のマイクロバッチに割り、全 forward → 全 backward。バブル比率 ≈ `(p−1)/(m+p−1)` で、m を増やすほど薄まるが activations の同時保持が増える[^gpipe]。
- **1F1B**: 定常状態で forward と backward を交互に流し、保持 activations をステージ深さ分に抑える。interleaved 1F1B（1 GPU に複数の不連続ステージを割り当てる）はバブルをさらに削る代わりに通信回数を増やす[^megatron-sc]——バブル・メモリ・通信の三すくみの中の点の選択である。
- パイプラインの本質的な弱点は**負荷の均一性への依存**で、層ごとの計算量が不揃いだと最遅ステージが全体を律速する（`interconnects-scaleout-training.md` のストラグラーと同型の構造が、今度は設計内部に現れる）。

### 2. ZeRO / FSDP の通信パターン (Sharded Data Parallelism)

- Stage 3 / FSDP の 1 ステップ: 各層の forward 直前に**その層のパラメータだけ** all-gather → 計算 → 即解放、backward で再度 all-gather + 勾配は reduce-scatter で分割先へ直接縮約。ピークメモリは「最大の 1 層 + 分割されたモデル状態」に落ちる[^zero]。
- 通信量は素朴 DP の約 1.5 倍（all-gather ×2 + reduce-scatter ×1 ≈ 3Ψ vs all-reduce の 2Ψ）で、prefetch（次層の all-gather を先行発行）により大部分を重畳できる。**「メモリを大量に食う複製」を「帯域を少し多く食う分割」に交換する**——この取引の成立条件がインターコネクト帯域であることは、hardware 側 (`interconnects-scaleout-training.md`) の存在理由そのものである。
- CPU/NVMe への offload 系（optimizer 状態の退避）は、さらに容量を階層下位で買う変種であり、`../../hardware/hw-sw-interface/storage-nvm-interface.md` の帯域が成立条件になる。

### 3. 混合精度と数値の運用 (Mixed Precision in Practice)

- **FP16 + 損失スケーリング**: 勾配の分布が FP16 の正規化数の下限を割るため、損失を定数倍して勾配を表現可能域に持ち上げ、更新前に戻す。オーバーフロー検出時はそのステップを捨ててスケールを下げる動的制御が標準[^amp]。
- **BF16**: 指数部が FP32 と同じため損失スケーリングが不要になり、大規模学習の既定になった。それでも FP32 マスターと FP32 での縮約（少なくとも部分和）は保持される——「計算は低精度・**累積は高精度**」が壊れない構成の不変条件である。
- **再現性との緊張**: 並列縮約の順序非決定性（NCCL のアルゴリズム選択・SHARP のスイッチ内加算）はビット再現性を壊す。デバッグ・規制対応でビット一致が要る場合は決定的な縮約順序を強制し、性能を払う——`../../software/security/fuzzing-program-analysis.md` の「オラクル」の議論と同じく、**何を「同じ」とみなすかが先に決まらないと検証は定義できない**。

### 4. チェックポイントと故障運用 (Checkpointing & Failure Operations)

- チェックポイントの内容はモデル状態一式（≈16Ψ バイト + データローダ位置・RNG 状態）で、分割並列に合わせて**シャーディングされたまま**書く。最適化の定石は二段階化——GPU → ホストメモリへ同期的に退避（数秒）し、永続ストレージへは非同期に書く——で、学習停止時間を退避時間だけに縮める。
- 復旧時間の会計: 損失 = (チェックポイント間隔の期待損失 ≈ 間隔/2) + 検知 + 再スケジュール + 復元 + ウォームアップ。間隔を詰めるほど前者が減り書き込み負荷が増える——`realtime` の世界と同じく、**期待値ではなく分布**（故障間隔 ≈ 3 時間という Llama 3 の実測[^llama3]）から逆算する。
- 「遅くなる故障」（ECC 再試行・温度スロットリング・リンク劣化）は停止しないぶん検出が難しく、全ランクのステップ時間・通信時間の分布監視（`../observability/observability-performance-engineering.md`）が実務の主力になる。
- データパイプライン（トークン化・シャッフル・シャーディング）は見落とされがちな律速点で、GPU を遊ばせないための先読み・キャッシュと、中断時に**正確に同じ位置から**再開できる決定的なイテレータ状態の両立が要る。

## セキュリティ考察 (Security Considerations)

- **学習データは最大の攻撃面である。** ウェブ規模のコーパスは誰でも「サプライヤ」になれる供給網であり、データポイズニング（特定トリガーへの挙動の埋め込み）は学習時にしか仕込めず推論時の検査では見つけにくい。データソースの来歴・フィルタリングパイプラインの完全性は、`../../software/security/software-supply-chain-security.md` の「出所の検証」をデータへ拡張したものである。
- **チェックポイントは実行可能物として扱う。** 復元はデシリアライズであり（pickle 系の任意コード実行の系譜）、加えて重み自体への改竄（バックドア注入）は学習途中のチェックポイント経由が最も検出困難である。保存形式の安全化（テンソル専用形式）、チェックポイントの署名・完全性検証、アクセス監査が対策の柱になる。
- **クラスタは高価値・長時間・単一目的の標的である。** 数万 GPU 時間の焼却（可用性攻撃）、重みの持ち出し、ジョブへの介入のいずれも被害が巨額になる。`interconnects-scaleout-training.md` で述べたとおりファブリック内の信頼モデルは弱く、境界防御・テナント分離・ストレージ暗号化が前提となる。
- **再現性は監査可能性でもある。** 「このモデルはこのデータとこのコードから、この乱数で作られた」を主張できる決定的学習は、規制・インシデント調査への備えであり、モデル来歴 (provenance) の技術的土台になる——性能との交換（発見事項 5・技術詳細 3）を、セキュリティ要件として意識的に選ぶ場面が増えている。

## 結論 (Conclusions)

学習システムは「バイトの会計」から始まり「可用性の工学」で終わる。モデル状態 16Ψ バイトという算術が分割（ZeRO・3D 並列）を強制し、分割は通信を生み、通信は隠せるか否かで無料と課金に分かれ、隠せない部分とバブルが MFU の上限を画す。数値形式は削れる限界まで削られ、累積だけが高精度に残る。そして万台規模では故障が常態となり、チェックポイントと goodput が性能指標に昇格する。推論 (`ml-inference-systems.md`) が「メモリ帯域の経済」だったのに対し、学習は**「容量・帯域・可用性の三元経済」**である——そしてその三つとも、hardware 側の章（チップ・HBM・インターコネクト・光）が供給する物理量の上に立っている。AI スタックの分析が hardware/software の相互参照なしに成立しない理由が、ここに集約されている。

---

## 参考文献 (References)

[^zero]: Rajbhandari, S., Rasley, J., Ruwase, O. & He, Y. (2020). ZeRO: Memory Optimizations Toward Training Trillion Parameter Models. *Proc. SC20*. DOI: 10.1109/SC41405.2020.00024 — 16 バイト/パラメータの会計と 3 段階分割の原典。

[^megatron]: Shoeybi, M. et al. (2019). Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism. arXiv:1909.08053 — 列/行分割テンソル並列の原典。

[^megatron-sc]: Narayanan, D. et al. (2021). Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM. *Proc. SC21*. DOI: 10.1145/3458817.3476209 — 3D 並列の合成と interleaved 1F1B、MFU 分析。

[^gpipe]: Huang, Y. et al. (2019). GPipe: Efficient Training of Giant Neural Networks using Pipeline Parallelism. *Proc. NeurIPS 2019*. — マイクロバッチとパイプラインバブルの原典。

[^amp]: Micikevicius, P. et al. (2018). Mixed Precision Training. *Proc. ICLR 2018*. arXiv:1710.03740 — FP32 マスター・損失スケーリングの原典。

[^recompute]: Chen, T., Xu, B., Zhang, C. & Guestrin, C. (2016). Training Deep Nets with Sublinear Memory Cost. arXiv:1604.06174 — activation recomputation（gradient checkpointing）の原典。

[^palm]: Chowdhery, A. et al. (2022). PaLM: Scaling Language Modeling with Pathways. arXiv:2204.02311 — MFU の定義と大規模実測（46% 前後）の一次報告。

[^llama3]: Grattafiori, A. et al. (2024). The Llama 3 Herd of Models. arXiv:2407.21783 — 16,384 GPU・54 日・419 中断・78% ハードウェア起因、90% 超の実効稼働時間の一次報告。

その他の標準的典拠:

- NVIDIA. *NCCL Documentation*. https://docs.nvidia.com/deeplearning/nccl/ — Accessed: 2026-07-13. — 集合通信の実装・重畳の一次資料。
- Jiang, Z. et al. (2024). MegaScale: Scaling Large Language Model Training to More Than 10,000 GPUs. *Proc. NSDI 2024*. — 万台規模運用（診断・ストラグラー対策）の一次報告。
