# AI / LLM

大規模言語モデル (LLM) と機械学習 (ML) のシステムの分析。モデルそのもの（アルゴリズム）ではなく、
モデルを中心に置いたシステムを、モデルの下と上の 2 つの層に分けて扱う。

- **モデルの下の層（ML システム）** — モデルを「実行・提供するシステム」: 推論ランタイム、学習基盤。
  GPU・アクセラレータ (`../../hardware/accelerators/`) の上に立つソフトウェア層であり、
  コンパイラ・ランタイム・分散システム・OS のメモリ管理といった技法が交差する。
  スケジューリング・メモリ管理・バッチングの観点から「1 回の forward pass をどう速く安く実行するか」を分析する
- **モデルの上の層（エージェント）** — モデルを部品として組み込んだシステム: エージェント (agents)、
  エージェントループの工学 (agentic loop engineering)、ツール使用 (tool use)、コンテキスト管理。
  LLM を「呼び出す側」のアーキテクチャとして「多数のモデル呼び出しをどう編成して仕事を成すか」を分析する

隣接する領域との境界は次のとおりである。

- **`../search/`** — ベクトル検索そのものの機構（HNSW/IVF-PQ）。RAG の検索側部品
- **`../../foundations/`** — 数理的基礎。とくに `machine-learning/`（Transformer の
  数理、スケーリング則）が本ディレクトリの理論側の対

## 構成

### モデルの下の層（ML システム）

- `ml-inference-systems.md` — ML 推論システム：CUDA 実行モデル (streams / CUDA Graphs)、
  prefill/decode の二相構造、KV cache と PagedAttention、連続バッチング (Orca)、
  FlashAttention、量子化、投機的デコーディング、推論ランタイム比較 (vLLM / TensorRT-LLM ほか)
- `ml-training-systems.md` — ML 学習システム：モデル状態 16 バイト/パラメータの会計、
  ZeRO/FSDP（複製をやめ分割する）、3D 並列とインターコネクト階層への配置、
  パイプラインスケジュール (GPipe/1F1B) とバブル、混合精度と損失スケーリング、
  activation recomputation、チェックポイントと故障運用 (goodput)。
  `../../hardware/accelerators/interconnects-scaleout-training.md` と対で読む

### モデルの上の層（エージェント）

- `agent-architectures.md` — LLM エージェントのアーキテクチャ:
  ワークフローとエージェントの区別、ReAct 系譜（推論と行動の交互実行）、
  計画・自己修正 (Reflexion)、メモリ設計、マルチエージェント構成、
  MCP (Model Context Protocol) によるツール統合の標準化、評価 (SWE-bench, τ-bench)、
  プロンプトインジェクションと権限設計
- `llm-internals-prompting.md` — LLM の内部構造とプロンプティング:
  トークン化、固定計算と思考の外部化 (chain-of-thought)、注意＝文脈への検索、
  学習パイプライン（事前学習 → SFT → RLHF）と追従性 (sycophancy)、
  文脈内学習と例示、サンプリングと幻覚。「構造 → 挙動 → 使い方」の因果で
  プロンプティング原則を導出する、エージェント 2 本の 1 層下の解剖学
- `agent-loop-engineering.md` — エージェントループの工学:
  ループの解剖（モデル呼び出し → stop_reason 分岐 → ツール実行 → 履歴追加）、
  ハーネスの責務、ツールサーフェス設計（bash 対 専用ツール）、
  コンテキストエンジニアリング（compaction・context editing・プレフィックス安定性と
  プロンプトキャッシュ）、エラー処理と耐久性、human-in-the-loop ゲート、観測性。
  `agent-architectures.md` が「何を作るか」なら、こちらは「どう安定して動かすか」

## 今後のテーマ候補

- ML コンパイラ (ML compilers) — XLA / TVM / Triton, グラフ最適化とカーネル自動生成。
  推論ドキュメントのカーネル融合 (kernel fusion) 議論の体系化
- 量子化とスパース化の深掘り (quantization & sparsity) — PTQ/QAT, 構造化スパース性,
  ハードウェアサポート (INT4/FP8/2:4 sparsity) との共進化
- RAG システムアーキテクチャ (retrieval-augmented generation) —
  チャンキング、ハイブリッド検索、リランキング、評価。
  `../search/vector-search-ann.md`（検索部品）との接続
- LLM 評価 (evals) の方法論 — ベンチマーク設計、LLM-as-judge のバイアス、
  能力評価と安全性評価の区別
- LLM セキュリティの深掘り — プロンプトインジェクションの分類と防御設計、
  jailbreak、エージェントの権限モデル。`../security/` との横断
- 学習パイプライン (training pipeline) — 事前学習 → SFT → RLHF/DPO の全体像。
  実行基盤側は `ml-training-systems.md` が担当。
  理論側の入口は `../../foundations/machine-learning/` に整備済み
