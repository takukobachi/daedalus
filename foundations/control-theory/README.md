# 制御理論 (Control Theory)

測った結果を入力へ戻し、系の振る舞いを望む形に保つための理論。ロボットの関節サーボ、
航空機の姿勢制御、電力系統の周波数制御、PLL・CDR、オートスケーラなど、Daedalus の
分析文書にはフィードバックループが繰り返し現れる。このシリーズは、それらに共通する
言語（伝達関数、感度関数、安定余裕、状態空間、推定）を基礎から積み上げる。

微分方程式 ([`../mathematics/calculus-analysis/`](../mathematics/calculus-analysis/))、
フーリエ解析 ([`../mathematics/fourier-signal-processing/`](../mathematics/fourier-signal-processing/))、
線形代数 ([`../mathematics/linear-algebra/`](../mathematics/linear-algebra/)) の合流点に立つ
foundations 直下の系列で、情報理論・機械学習に続く第三の「数学・物理以外の基礎理論」にあたる。

## 文書一覧 (Documents)

1. [`feedback-transfer-functions.md`](feedback-transfer-functions.md) —
   負帰還の原理 (Black の増幅器)、ラプラス変換、伝達関数と極・零点、
   非最小位相系とむだ時間、感度関数 S と相補感度関数 T、過渡応答仕様と
   定常偏差、内部モデル原理、PID の数理と実装（微分フィルタ・アンチワインドアップ・
   離散化）、ラウス–フルヴィッツによる安定限界。シリーズの起点
2. [`frequency-response-stability-margins.md`](frequency-response-stability-margins.md) —
   周波数応答とボード線図、最小位相系と利得–位相関係、ナイキスト安定判別、
   ゲイン余裕・位相余裕・最大感度 M_s、むだ時間と遅れ余裕、
   ボードの感度積分（ウォーターベッド効果と不安定プラントの限界）、
   進み・遅れ補償とノッチによるループ整形

## ロードマップ (Roadmap)

3. **状態空間と推定** — 状態方程式、可制御性・可観測性、極配置、LQR、
   オブザーバとカルマンフィルタ、LQG と分離定理。
   [`../../engineering/aerospace/avionics-gnc.md`](../../engineering/aerospace/avionics-gnc.md)・
   [`../../engineering/robotics/perception-state-estimation.md`](../../engineering/robotics/perception-state-estimation.md)
   のセンサ融合の理論的基礎

以降のテーマ（非線形制御とリアプノフ安定性、ロバスト制御、MPC の理論）は
必要が生じた時点でロードマップを再開する。
