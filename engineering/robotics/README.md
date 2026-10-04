# ロボティクス (Robotics)

地上ロボット（産業用マニピュレータ・脚式/ヒューマノイド・自律移動ロボット）を題材とした
工学分析。`automotive/`・`aerospace/`・`energy/` に散在していた制御工学の
各論を一般化する「制御の本丸」であり、`foundations/` の線形代数・微分方程式・確率統計の
主要な応用先でもある。物理（機構・アクチュエータ）と制御ソフトウェアの接点を、
「なぜこの設計か」で読み解く。

## スコープの線引き

- **ADAS・自動運転は扱わない** — `../automotive/` のロードマップ項目として扱う
- **ドローン・飛行力学は扱わない** — `../aerospace/` の領域。本領域は地上ロボットを軸にする
- **RTOS の一般論は扱わない** — [`../../software/embedded/`](../../software/embedded/)
  で扱い、本領域からは参照するに留める

## ドキュメント

### 基礎 (Fundamentals)

- [`robotics-fundamentals.md`](robotics-fundamentals.md) — ロボット工学基礎：ロボットの分類、
  アクチュエータ（モーター・減速機・QDD）、センサ（エンコーダ・IMU・力覚）、
  制御ループの階層と周期、協働ロボットの安全規格
- [`kinematics-dynamics.md`](kinematics-dynamics.md) — キネマティクスと動力学：回転の表現、
  順運動学・逆運動学、ヤコビアンと特異点、可操作性、ラグランジュ法とニュートン・オイラー法、
  運動方程式の構造
- [`control-systems.md`](control-systems.md) — ロボット制御：カスケード制御と PID、
  計算トルク法、状態空間制御 (LQR)、モデル予測制御 (MPC)、インピーダンス制御、
  リアルタイム制約と産業用フィールドバス (EtherCAT)

### 知覚と運動 (Perception & Motion)

- [`perception-state-estimation.md`](perception-state-estimation.md) — 知覚と状態推定：
  ベイズフィルタ、カルマンフィルタと非線形拡張 (EKF/UKF/パーティクルフィルタ)、
  センサフュージョンの定石、SLAM (因子グラフ・ループ閉じ込み)、脚式の接触支援推定
- [`locomotion-manipulation.md`](locomotion-manipulation.md) — 移動と操作：非ホロノミック拘束、
  ZMP と線形倒立振子・予見制御、捕捉点と動歩行、運動計画 (RRT/PRM・軌道最適化)、
  把持 (摩擦錐・フォースクロージャ)、ハイブリッド位置/力制御

### 事例分析とソフトウェア基盤 (Case Studies & Software Stack)

- [`humanoid-robots.md`](humanoid-robots.md) — ヒューマノイド事例分析
  (2026-10 時点)：Tesla Optimus・Figure・Unitree・Boston Dynamics の設計比較
  （アクチュエータ戦略、手と腱駆動、二系統 VLA、生産体制）と Agility・1X の動向、
  ISO 25785-1 の策定状況。目標値と実証値を峻別
- [`robot-software-stack.md`](robot-software-stack.md) — ロボットソフトウェアスタック：
  ROS 2 / DDS（QoS・ディスカバリ）、リアルタイム性の境界と ros2_control、
  Nav2 / MoveIt、シミュレーションと sim-to-real、SROS2

## 今後のテーマ候補

当初計画した7本のシリーズはこれで完結。以降のテーマ（車輪型 AMR の各論、
ソフトロボティクス、移動型ヒューマノイドの安全規格の動向、ROS 2 実時間性の実測分析
など）は必要が生じた時点でロードマップを再開する。なお `humanoid-robots.md` は
時事性が高く、製品状況が大きく動いた時点での改訂を想定する。
