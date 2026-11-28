# 古典力学 (Classical Mechanics)

質量をもつ物体の運動を支配する物理。ロボット・ロケット・自動車の制御ループが
作用する「プラント」の正体はニュートン力学の運動方程式である。このシリーズは
ロボティクス (`../../../engineering/robotics/`)、航空宇宙
(`../../../engineering/aerospace/`)、車載 (`../../../engineering/automotive/`) の
分析群が暗黙に依拠する力学を、三法則から振動・安定性・多体系・軌道まで
積み上げて文書化する。

## 文書一覧 (Documents)

1. [`newton-laws-oscillations.md`](newton-laws-oscillations.md) —
   ニュートンの三法則と運動方程式(慣性系・自由体図)、運動量とエネルギーの
   保存則・ツィオルコフスキーのロケット方程式、剛体回転(トルク・慣性
   モーメント・ジャイロ効果とリアクションホイール)、単振動・減衰・共振
   (RLC の機械的双子)、振り子の線形化と倒立振子。シリーズの起点
2. [`lagrangian-multibody-dynamics.md`](lagrangian-multibody-dynamics.md) —
   一般化座標とホロノミック拘束、最小作用の原理とオイラー=ラグランジュ方程式、
   2リンクアームの例、マニピュレータ方程式 M(q)q̈ + C(q,q̇)q̇ + g(q) = τ の導出と
   構造(正定値性・Ṁ − 2C の歪対称性・パラメータ線形性)、ネーターの定理、
   非ホロノミック拘束、O(n) の再帰アルゴリズム
3. [`central-force-kepler-orbits.md`](central-force-kepler-orbits.md) —
   二体問題の換算、角運動量保存と面積速度、有効ポテンシャルと脱出速度、
   軌道方程式と円錐曲線・ヴィス・ヴィヴァ式・第3法則の導出、軌道要素と
   ケプラー方程式、J2 摂動と太陽同期軌道
