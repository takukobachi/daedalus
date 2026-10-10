# 数学 (Mathematics)

分析を支える数学の基礎。定義・定理を積み上げる本文に加え、各文書は
「Daedalus内の応用」セクションで既存の分析ドキュメントへリンクする
（`foundations/README.md` の規約）。

## 構成 (Structure)

- `linear-algebra/` — 線形代数(ベクトル空間、線形写像、行列積の計算構造、
  数値線形代数 …)。
  SIMD (`../../hardware/cpu/simd-vector.md`)、
  GPU・Tensor Core (`../../hardware/accelerators/`)、
  機械学習システム (`../../software/ai/`) の計算の実体
- `probability-statistics/` — 確率・統計(確率空間、確率変数、主要分布、
  極限定理 …)。
  分岐予測 (`../../hardware/cpu/branch-prediction-speculation.md`)、
  製造歩留まり (`../../hardware/semiconductor-manufacturing/`)、
  待ち行列・パフォーマンス分析 (`../../software/observability/`) の言語
- `calculus-analysis/` — 微分積分・解析(極限、微分、積分、テイラー展開、
  微分方程式 …)。物理学シリーズ全体と制御理論の前提
- `discrete-logic/` — 離散数学・論理(命題論理・ブール代数、グラフと木、
  オートマトン・形式言語、計算可能性、計算量理論 …)。
  論理回路・FPGA (`../physics/quantum-solid-state/mosfet-cmos.md`,
  `../../hardware/fpga/`)、経路制御 (`../../software/networking/`)、
  正規表現・パーサ・検証の限界 (`../../software/security/`,
  `../../software/operating-systems/`) の基礎
- `number-theory-algebra/` — 数論・代数(合同算術と群、有限体、公開鍵暗号の数学 …)。
  TLS・セキュアブート (`../../software/networking/tls-internals.md`,
  `../../hardware/security/`)、CRC・スクランブラ
  (`../../hardware/signal-integrity/`)、イレイジャーコーディング
  (`../../software/object-storage/`)・ECC (`../../hardware/storage/`) の代数
- `fourier-signal-processing/` — フーリエ解析・信号処理(フーリエ級数/変換、
  畳み込み、サンプリング定理、FFT …)。無線
  (`../../hardware/peripherals/wifi-architecture.md`)、オーディオ
  (`../../hardware/peripherals/audio-subsystem.md`)、
  画像・ディスプレイ (`../../hardware/peripherals/display-pipeline.md`)、
  高速シリアルリンク (`../../hardware/signal-integrity/`) の基礎
- `optimization/` — 最適化(凸性と LP/QP/SOCP、勾配法、KKT 条件、双対性と影の価格 …)。
  着陸誘導 (`../../engineering/aerospace/avionics-gnc.md`)、電力市場
  (`../../engineering/energy/electricity-markets-dispatch.md`)、
  ML 訓練 (`../../software/ai/`)、MPC (`../../engineering/robotics/control-systems.md`)
  の共通言語
- `numerical-analysis/` — 数値解析(浮動小数点と誤差解析、条件数と安定性、
  求根・反復線形ソルバ、ODE の数値積分と硬さ、シンプレクティック積分 …)。
  SIMD・GPU の数値形式 (`../../hardware/cpu/simd-vector.md`,
  `../../hardware/accelerators/`)、混合精度学習 (`../../software/ai/`)、
  軌道伝播・ロボットのシミュレーション (`../../engineering/aerospace/`,
  `../../engineering/robotics/`) の計算側の基礎。線形代数(数値線形代数)と
  微分積分(常微分方程式)の続き

## 今後のテーマ候補

当初ロードマップに挙げた候補(数値解析を含む)はすべて着手済み。以降のテーマは、
各サブディレクトリの README の「今後のテーマ候補」で管理する。
