# 熱力学・統計力学 (Thermodynamics & Statistical Mechanics)

エネルギーの量と質、熱の流れを支配する物理。「チップに入った電力はなぜ全部
熱になるのか」「冷却はなぜ電力を食うのか」「発電所の効率はなぜ頭打ちなのか」に、
熱力学の法則からエントロピーの統計的意味・熱移動の工学まで積み上げて
答えることがこのシリーズの目標である。データセンター冷却
(`../../../hardware/datacenter-infrastructure/`)、発電
(`../../../hardware/power-generation/`)、クロック・電力管理
(`../../../hardware/hw-sw-interface/clock-power-reset.md`) の支配法則を与える。

## 文書一覧 (Documents)

1. [`laws-entropy-heat-engines.md`](laws-entropy-heat-engines.md) —
   温度と熱平衡(第0法則)、第一法則(消費電力 = 発熱量の由来)、
   第二法則とエントロピー (S = k ln W)、熱機関とカルノー限界・冷凍機の COP、
   熱伝導・対流・放射と熱抵抗回路。シリーズの起点

## 今後のロードマップ (Roadmap)

2. **統計力学とボルツマン分布** — 分配関数、ボルツマン分布、熱雑音
   (ジョンソン=ナイキスト)、ランダウアー限界(1 ビット消去の熱力学的コスト)。
   半導体のキャリア統計 (`../quantum-solid-state/`) と計算の物理的下限
3. **伝熱工学の深掘り** — 熱抵抗網の過渡応答(熱容量・熱時定数)、
   フィン・ヒートシンク設計、相変化伝熱(沸騰・ヒートパイプ)。
   液冷・液浸 (`../../../hardware/datacenter-infrastructure/datacenter-cooling.md`)
   の定量的な設計言語
