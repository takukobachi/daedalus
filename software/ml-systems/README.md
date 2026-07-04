# ML Systems

機械学習システム (ML systems) の分析。ML システムは GPU・アクセラレータ
(`../../hardware/gpu-accelerators/`) の上に立つソフトウェア層であり、
コンパイラ・ランタイム・分散システム・OS のメモリ管理といった技法が交差する応用層。
モデルそのもの（アルゴリズム）ではなく、モデルを「実行・提供するシステム」を対象に、
スケジューリング・メモリ管理・バッチングの観点から深掘りする。

## 構成

- `ml-inference-systems.md` — ML 推論システム：CUDA 実行モデル (streams / CUDA Graphs)、
  prefill/decode の二相構造、KV cache と PagedAttention、連続バッチング (Orca)、
  FlashAttention、量子化、投機的デコーディング、推論ランタイム比較 (vLLM / TensorRT-LLM ほか)

## 今後のテーマ候補

- 学習系システム (training systems) — data / tensor / pipeline parallelism, ZeRO,
  collective communication (NCCL)。推論とは逆に「帯域と同期」が支配する世界
- ML コンパイラ (ML compilers) — XLA / TVM / Triton, グラフ最適化とカーネル自動生成。
  本ドキュメントのカーネル融合 (kernel fusion) 議論の体系化
- 量子化とスパース化の深掘り (quantization & sparsity) — PTQ/QAT, 構造化スパース性,
  ハードウェアサポート (INT4/FP8/2:4 sparsity) との共進化
