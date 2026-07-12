# ロボティクス (Robotics)

地上ロボット（産業用マニピュレータ・脚式/ヒューマノイド・自律移動ロボット）を題材とした
ハードウェア分析。`automotive/`・`aerospace/`・`power-generation/` に散在していた制御工学の
各論を一般化する「制御の本丸」であり、`foundations/` の線形代数・微分方程式・確率統計の
主要な応用先でもある。物理（機構・アクチュエータ）と制御ソフトウェアの接点を、
「なぜこの設計か」で読み解く。

## スコープの線引き

- **ADAS・自動運転は扱わない** — `../automotive/` のロードマップ項目として扱う
- **ドローン・飛行力学は扱わない** — `../aerospace/` の領域。本領域は地上ロボットを軸にする
- **RTOS の一般論は扱わない** — `software/` 側の「組み込み・リアルタイム OS」候補として
  別途扱い、本領域からは参照するに留める

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

## 今後のテーマ候補

優先度は hardware/software 両スタックとの相互参照の張りやすさを基準に付けている。

### 優先度 高

- 知覚と状態推定 (perception & state estimation) — センサフュージョン、カルマンフィルタ
  (KF/EKF/UKF)、SLAM。`../../foundations/mathematics/probability-statistics/` の推定理論の
  直接の応用で、`../peripherals-connectivity/camera-image-sensors.md` の上位層
- 移動と操作 (locomotion & manipulation) — 脚式移動 (ZMP・動歩行・全身制御)、
  軌道計画 (RRT・最適化ベース)、把持。`kinematics-dynamics.md`・`control-systems.md` の応用編

### 優先度 中

- ヒューマノイド事例分析 (humanoid robots) — Tesla Optimus・Figure・Unitree の
  アーキテクチャ比較（アクチュエータ構成、計算機構成、学習ベース制御・VLA）。
  `../automotive/ev/` の Tesla 分析と同じ一次資料からの事例再構成パターンで、
  `../../software/ml-systems/` と接続する
- ロボットソフトウェアスタック (robot software stack) — ROS 2 / DDS、リアルタイム
  ミドルウェア、シミュレーション (Isaac, MuJoCo)。software 側の RTOS 文書の開設後に
  着手するのが書きやすい
