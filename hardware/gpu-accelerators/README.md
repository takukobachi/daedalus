# GPU・アクセラレータ (GPU & Accelerators)

スループット指向 (throughput-oriented) プロセッサの分析。`../cpu-architecture/` が扱う「レイテンシ指向 (latency-oriented)」設計は、大容量キャッシュ・アウトオブオーダ実行・分岐予測で単一スレッドを速くする。本ディレクトリはその対照編であり、同じメモリウォールに対して「超多スレッドでレイテンシを隠す」という逆の解を採る系譜を追う。software 側の ML システム分析 (`../../software/ml-systems/`) と対になる層であり、ここでのハードウェア理解が学習・推論システムの性能分析の基礎になる。

## ドキュメント

- [`gpu-simt-architecture.md`](gpu-simt-architecture.md) — GPU アーキテクチャと SIMT 実行モデル (GPU architecture & SIMT execution)
- [`tpu-systolic-arrays.md`](tpu-systolic-arrays.md) — TPU とシストリックアレイ (TPU & systolic arrays)：
  演算 vs データ移動のエネルギー算術、重み定常データフロー、TPUv1 の決定性、
  データフロー分類 (Eyeriss)、GPU との収斂（Tensor Core ⇔ 汎用化する TPU）、
  BF16/FP8 という数値形式の共設計
- [`interconnects-scaleout-training.md`](interconnects-scaleout-training.md) — インターコネクトと
  スケールアウト学習 (interconnects & scale-out training)：スケールアップ (NVLink/NVSwitch) と
  スケールアウト (InfiniBand/RoCEv2) の2階層、リング/木 all-reduce の算術、GPUDirect RDMA、
  SHARP (スイッチ内リダクション)、レール最適化、万台規模の故障と可用性。
  `../network-hardware/` と `../../software/ml-systems/ml-training-systems.md` の中間層

## 今後のテーマ候補

- GPU 仮想化とマルチテナント (GPU virtualization) — MIG, SR-IOV, vGPU、時分割 vs 空間分割、
  クラウド GPU の隔離境界。`../../software/security/sandbox-architectures.md` の
  隔離境界の議論のアクセラレータ版
