# 離散数学・論理 (Discrete Mathematics & Logic)

計算機の内部を成す離散構造の数学。真/偽の代数（ブール代数）、関係の構造（グラフ）、
状態と遷移の機械（オートマトン）を積み上げ、「有限の機械に何ができて何ができないか」
（計算可能性）と、どれだけの資源で計算できるか（計算量）まで到達する4回シリーズ。論理回路・FPGA の LUT、経路制御、GC、
デッドロック検出、正規表現エンジンと ReDoS、eBPF 検証器の設計判断など、Daedalus の
分析に散在する離散的な機構に共通の語彙を与える。

## 文書一覧 (Documents)

1. [`propositional-boolean-logic.md`](propositional-boolean-logic.md) —
   命題論理の構文と意味論、ブール代数の法則と双対性、標準形 (DNF/CNF) と
   関数的完全性、Shannon 展開 (＝MUX/LUT)、簡単化 (カルノー図・Quine–McCluskey)、
   SAT と NP 完全性、CDCL ソルバと検証産業。シリーズの起点
2. [`graphs-trees-algorithms.md`](graphs-trees-algorithms.md) —
   グラフの定義と表現（隣接行列の冪＝経路数で線形代数と合流）、木と全域木
   (カット性質と STP)、BFS/DFS と到達可能性・閉路検出、DAG とトポロジカル順序・
   クリティカルパス、Dijkstra と Bellman–Ford（＝リンクステート/距離ベクトル経路制御）
3. [`automata-formal-languages.md`](automata-formal-languages.md) —
   DFA/NFA と部分集合構成、正規表現との等価性とエンジン2系譜 (ReDoS)、
   反復補題と有限メモリの限界、Chomsky 階層（メモリ構造と言語クラスの対応）、
   停止性問題と Rice の定理、「制限するか近似するか」という検証の設計論
4. [`computational-complexity.md`](computational-complexity.md) —
   多項式と指数の差、P・NP・co-NP、帰着と NP 完全性 (Cook・Karp)、
   NP 困難な問題への対処（特殊構造・近似とその限界・擬多項式時間・ヒューリスティクス）、
   PSPACE・EXPTIME と検証・型推論の困難さ、暗号が頼る平均的な困難さと Shor、
   最悪計算量と実際の性能のずれ（単体法と平滑化解析、ReDoS）。最終回

当初計画した3本に計算量理論の1本を加え、シリーズはこれで完結。以降のテーマ
（数え上げ・組合せ論、グラフの発展 (フロー・マッチング)）は必要が生じた時点で
ロードマップを再開する。
