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
- `ml-compilers.md` — ML コンパイラ：グラフ捕捉 (TorchDynamo のガードとグラフブレーク、
  JAX のトレーシング)、グラフ最適化（融合・レイアウト割当・CSE）、カーネル生成
  (Triton のブロック単位モデル、Halide のアルゴリズム/スケジュール分離、TVM/Ansor の探索)、
  MLIR の方言スタックと StableHLO、コンパイル時間と実行時性能、動的形状。
  推論ドキュメントのカーネル融合議論の体系化

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
- `rag-system-architecture.md` — RAG システムアーキテクチャ：チャンキング戦略、
  埋め込みモデル、ハイブリッド検索 (BM25 + 密検索、RRF)、再順位付け
  (交差エンコーダ、ColBERT)、クエリ書き換え (HyDE)、長文脈との使い分け、GraphRAG、
  2 層評価 (recall/nDCG と忠実性)、失敗モード、間接プロンプトインジェクションと検索時の認可。
  `../search/vector-search-ann.md`（検索部品）の上に組み立てる側
- `llm-evaluation-methodology.md` — LLM 評価の方法論：測定システムとしての評価
  (妥当性・系統誤差・偶然誤差・過小評価)、ベンチマークの飽和と汚染検出、
  LLM-as-a-judge のバイアス、Chatbot Arena と Bradley–Terry、エージェント評価の信頼性、
  統計的厳密さ（標準誤差・対応のある比較・検出力）、能力評価と危険能力評価・
  フロンティア安全フレームワーク

## 今後のテーマ候補

LLM セキュリティの深掘り（プロンプトインジェクションの分類と防御設計、jailbreak、
エージェントの権限モデル）は [`../security/llm-application-security.md`](../security/llm-application-security.md)
で扱い済み。残る候補は以下。

- 量子化とスパース化の深掘り (quantization & sparsity) — PTQ/QAT, 構造化スパース性,
  ハードウェアサポート (INT4/FP8/2:4 sparsity) との共進化
- 学習パイプライン (training pipeline) — 事前学習 → SFT → RLHF/DPO の全体像。
  実行基盤側は `ml-training-systems.md` が担当。
  理論側の入口は `../../foundations/machine-learning/` に整備済み
