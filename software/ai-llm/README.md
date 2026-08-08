# AI / LLM

大規模言語モデル (LLM) の「モデルとその応用」レイヤの分析。隣の `../ml-systems/` が
「モデルを実行・提供するシステム」（GPU 実行、KV cache、バッチング、サービング）を
担当するのに対し、本ディレクトリはその**上に立つ層**——モデルを部品として組み込んだ
システムの設計——を対象とする。エージェント (agents)、エージェントループの工学
(agentic loop engineering)、ツール使用 (tool use)、コンテキスト管理といった、
LLM を「呼び出す側」のアーキテクチャを深掘りする。

境界の定義:

- **`../ml-systems/`** — モデルの下のシステム: 推論ランタイム、学習基盤。
  「1 回の forward pass をどう速く安く実行するか」
- **`ai-llm/`（本ディレクトリ）** — モデルの上のシステム: エージェント、ハーネス、
  ツール統合、コンテキストエンジニアリング。「多数のモデル呼び出しをどう編成して
  仕事を成すか」
- **`../search-ir/`** — ベクトル検索そのものの機構（HNSW/IVF-PQ）。RAG の検索側部品
- **`../../foundations/`** — 数理的基礎（線形代数、確率統計、最適化、情報理論）

## 構成

- `agent-architectures.md` — LLM エージェントのアーキテクチャ:
  ワークフローとエージェントの区別、ReAct 系譜（推論と行動の交互実行）、
  計画・自己修正 (Reflexion)、メモリ設計、マルチエージェント構成、
  MCP (Model Context Protocol) によるツール統合の標準化、評価 (SWE-bench, τ-bench)、
  プロンプトインジェクションと権限設計
- `agent-loop-engineering.md` — エージェントループの工学:
  ループの解剖（モデル呼び出し → stop_reason 分岐 → ツール実行 → 履歴追加）、
  ハーネスの責務、ツールサーフェス設計（bash 対 専用ツール）、
  コンテキストエンジニアリング（compaction・context editing・プレフィックス安定性と
  プロンプトキャッシュ）、エラー処理と耐久性、human-in-the-loop ゲート、観測性。
  `agent-architectures.md` が「何を作るか」なら、こちらは「どう安定して動かすか」

## 今後のテーマ候補

- RAG システムアーキテクチャ (retrieval-augmented generation) —
  チャンキング、ハイブリッド検索、リランキング、評価。
  `../search-ir/vector-search-ann.md`（検索部品）との接続
- LLM 評価 (evals) の方法論 — ベンチマーク設計、LLM-as-judge のバイアス、
  能力評価と安全性評価の区別
- LLM セキュリティの深掘り — プロンプトインジェクションの分類と防御設計、
  jailbreak、エージェントの権限モデル。`../security/` との横断
- 学習パイプライン (training pipeline) — 事前学習 → SFT → RLHF/DPO の全体像。
  実行基盤側は `../ml-systems/ml-training-systems.md` が担当
- Transformer の数理 — アテンションの線形代数的構造、スケーリング則。
  置き場所は `../../foundations/machine-learning/`（新設候補）が適切
