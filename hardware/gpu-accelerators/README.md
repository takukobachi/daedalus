# GPU・アクセラレータ (GPU & Accelerators)

スループット指向 (throughput-oriented) プロセッサの分析。`../cpu-architecture/` が扱う「レイテンシ指向 (latency-oriented)」設計——大容量キャッシュ・アウトオブオーダ実行・分岐予測で単一スレッドを速くする——への対照編であり、同じメモリウォールに対して「超多スレッドでレイテンシを隠す」という逆の解を採る系譜を追う。software 側の ML システム分析 (`../../software/ml-systems/`) と対になる層であり、ここでのハードウェア理解が学習・推論システムの性能分析の土台になる。

## ドキュメント

- [`gpu-simt-architecture.md`](gpu-simt-architecture.md) — GPU アーキテクチャと SIMT 実行モデル (GPU architecture & SIMT execution)

## 今後のテーマ候補

- TPU とシストリックアレイ (TPU & systolic arrays) — 重み定常 (weight-stationary) データフロー、
  GEMM 専用化の極点、GPU との設計比較の深掘り
- GPU 仮想化とマルチテナント (GPU virtualization) — MIG, SR-IOV, vGPU、時分割 vs 空間分割、
  クラウド GPU の隔離境界
- インターコネクトとスケールアウト学習 (interconnect & scale-out training) — NVLink/NVSwitch、
  InfiniBand/RoCE、集合通信 (all-reduce) のハードウェア支援。`../hw-sw-interface/bus-interconnect.md` と
  `../../software/distributed-systems/` の交点
