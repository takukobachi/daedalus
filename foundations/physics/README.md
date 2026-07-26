# 物理学 (Physics)

分析を支える物理学の基礎。法則・モデルを積み上げる本文に加え、各文書は
「Daedalus内の応用」セクションで既存の分析ドキュメントへリンクする
（`foundations/README.md` の規約）。

## 構成 (Structure)

- `classical-mechanics/` — 古典力学(運動方程式、運動量とエネルギー、剛体と回転、
  振動・共振、安定性 …)。ロボティクス (`../../hardware/robotics/`)、
  航空宇宙 (`../../hardware/aerospace/`)、自動車 (`../../hardware/automotive/`) の
  制御対象 (プラント) のモデルを与える
- `electromagnetism/` — 電磁気学(電荷・電場・電位、静磁気、電磁誘導、
  マクスウェル方程式、回路理論、伝送線路 …)。
  バス・インターコネクト (`../../hardware/hw-sw-interface/bus-interconnect.md`)、
  無線 (`../../hardware/peripherals-connectivity/`)、
  電力系統 (`../../hardware/power-generation/`) の土台。hardware README が
  優先度高に挙げる SerDes・シグナルインテグリティの前提でもある
- `quantum-solid-state/` — 量子力学・固体物理(バンド理論、半導体、
  ドーピング、pn 接合、MOSFET …)。半導体製造
  (`../../hardware/semiconductor-manufacturing/`)、NAND フラッシュ
  (`../../hardware/storage-devices/`)、原子時計
  (`../../hardware/timekeeping/`) の物理的実体
- `thermodynamics-statistical/` — 熱力学・統計力学(熱力学法則、エントロピー、
  熱機関とカルノー効率、熱伝導 …)。クロック・電力管理
  (`../../hardware/hw-sw-interface/clock-power-reset.md`)、発電
  (`../../hardware/power-generation/`)、データセンター冷却
  (`../../hardware/datacenter-infrastructure/`) の支配法則
- `fluid-dynamics/` — 流体力学(圧縮性流れ、マッハ数、デ・ラバルノズル、
  膨張比と Isp …)。ロケットエンジン
  (`../../hardware/aerospace/spacex-rocket-engines.md`) と航空機空力
  (`../../hardware/aerospace/aircraft-engineering-fundamentals.md`) の理論的土台

## 今後のテーマ候補

優先度は「既存の分析ドキュメントとの相互参照が何本張れるか」を基準に付けている。

### 優先度 低 (将来の拡張)

- 光学 (optics) — 回折、レーザー、フォトニクス。EUV リソグラフィ
  (`../../hardware/semiconductor-manufacturing/`) と光通信の前提
- 相対論 (relativity) — 特殊・一般相対論の基礎。GNSS 時刻補正
  (`../../hardware/timekeeping/`) で実務に現れる
