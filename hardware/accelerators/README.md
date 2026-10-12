# GPU・アクセラレータ (GPU & Accelerators)

スループット指向 (throughput-oriented) プロセッサの分析。`../cpu/` が扱う「レイテンシ指向 (latency-oriented)」設計は、大容量キャッシュ・アウトオブオーダ実行・分岐予測で単一スレッドを速くする。本ディレクトリはその対照編であり、同じメモリウォールに対して「超多スレッドでレイテンシを隠す」という逆の解を採る系譜を追う。software 側の ML システム分析 (`../../software/ai/`) と対になる層であり、ここでのハードウェア理解が学習・推論システムの性能分析の基礎になる。

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
  `../networking/` と [`../../software/ai/ml-training-systems.md`](../../software/ai/ml-training-systems.md) の中間層
- [`hbm-ai-memory.md`](hbm-ai-memory.md) — HBM と AI メモリシステム (HBM & the AI memory system)：
  decode の帯域律速と KV cache の算術、TSV 積層・ベースダイ・疑似チャネル、HBM〜HBM4 の
  JEDEC 規格値とベンダ製品値、CoWoS とレティクル限界、pJ/bit の内訳、熱・歩留まり・ウェハ消費比と
  供給集中（2026-10 時点）、カスタムベースダイ、GDDR7/LPDDR5X/SRAM/CXL との階層化、
  HBM のロウハンマーと GPU コンフィデンシャルコンピューティング

## 今後のテーマ候補

- GPU 仮想化とマルチテナント (GPU virtualization) — MIG, SR-IOV, vGPU、時分割 vs 空間分割、
  クラウド GPU の隔離境界。`../../software/security/sandbox-architectures.md` の
  隔離境界の議論のアクセラレータ版
