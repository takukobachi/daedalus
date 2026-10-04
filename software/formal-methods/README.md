# 形式手法 (Formal Methods)

「テストはバグの存在を示せるが、不在は示せない」(Dijkstra)。形式手法 (formal
methods) は、仕様とシステムを数学的対象として記述し、性質の成立を**全数検査
または証明**で確立する技術群である。対象は合意プロトコルの設計検証 (TLA+)、検証済み
OS カーネル (seL4)、検証済みコンパイラ (CompCert)、そして eBPF 検証器や Rust の
型システムのような「日常のツールに埋め込まれた軽量形式手法」まで及ぶ。既存の分析
文書に散在していたモデル検査・TLA+・検証器への言及を、表現力と自動化の
トレードオフという一つの地図の上で束ねる横断領域である。`security/`（攻撃の防止）・
`observability/`（挙動の観測）に続く第三の横串であり、「そもそも誤りを設計から
排除する」側からの品質保証を扱う。

## 文書一覧 (Documents)

1. [`formal-verification-landscape.md`](formal-verification-landscape.md) —
   形式手法の全体地図。時相論理と safety/liveness、TLA+/SPIN によるモデル検査と
   状態爆発、反例の価値 (AWS の実践)、Hoare 論理と演繹的検証 (seL4/CompCert の
   コストと教訓)、SMT ソルバ (Z3) という共通エンジン、抽象解釈・記号実行、
   eBPF 検証器・Rust 型システムに見る軽量形式手法、検証済み≠安全の限界論

## 今後のテーマ候補

- TLA+ 実践の各論 — 実在する分散システム（レプリケーション・トランザクション
  プロトコル）の仕様を実際に書いて検査する事例研究。`../distributed-systems/` の
  各文書の設計を仕様化して突き合わせる
- ハードウェア形式検証 (hardware formal verification) — 等価性検査、モデル検査に
  よるプロトコル検証 (キャッシュコヒーレンス)、RISC-V 形式仕様。
  `../../hardware/cpu/` との合流点
- 実行時検証と property-based testing — QuickCheck 系、ファジングとの融合
  (`../security/fuzzing-program-analysis.md`)、モニタ合成。「証明とテストの間」の
  実務地帯
