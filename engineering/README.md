# 工学システム (Engineering)

計算機の外に広がる物理的な工学システムの分析。航空宇宙・自動車・ロボット・電力・データセンター設備・モータ・鉄道・医療機器・通信インフラを題材に、物理プラントと制御ソフトウェアの接点を軸に読む。計算機そのもののハードウェアは `../hardware/` で扱う。

## 構成

- `automotive/` — 自動車（車載ネットワーク, ECU, E/E アーキテクチャ, 機能安全, ADAS・自動運転, EV …）
- `energy/` — 電力・エネルギー（発電方式, 電力系統, 送配電網, 電力市場, HVDC, 系統用蓄電池, 系統制御 …）
- `datacenters/` — データセンター（給電チェーン, UPS, 冷却・液冷,
  Clos トポロジ, PUE/WUE …）。`energy/` と計算機システムの合流点で、
  AI クラスタ (`../hardware/accelerators/`) の電力・冷却制約の物理的実体
- `aerospace/` — 航空・宇宙（航空機の基礎工学, ロケット, SpaceX, 衛星, 小型無人航空機 …）
- `robotics/` — ロボティクス（アクチュエータ, キネマティクス・動力学, 制御, 状態推定・SLAM,
  移動と操作 …）。
  `automotive/`・`aerospace/` に散在する制御工学の一般化であり、`../foundations/` の
  線形代数・微分方程式の主要な応用先
- `motors/` — 電動機・モータドライブ（機種比較, FOC, 弱め界磁, インバータと SiC, 効率規格,
  STO …）。`automotive/ev/`・`robotics/`・`energy/` に分散するモータと駆動の共通基盤
- `railway/` — 鉄道（閉塞・連動・ATS/ATC・ETCS/CBTC、フェールセーフ、EN 5012x と SIL …）。
  `automotive/` の機能安全と対比する、「止まれば安全」が成り立つ輸送ドメイン
- `medical/` — 医療機器（ISO 14971・IEC 62304 などの規格群、薬機法・FDA・EU MDR、
  Therac-25・輸液ポンプ・ペースメーカの事例、サイバーセキュリティと SBOM、SaMD …）。
  `automotive/`・`railway/` に続く安全規格ドメイン
- `telecom/` — 通信インフラ（海底ケーブル、中継器と給電、修理船、陸揚げの規制 …）。
  `../hardware/networking/` の光部品と `../software/networking/` の経路制御が前提にする、
  建屋と国境の外の伝送路
