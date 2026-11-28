# 流体力学 (Fluid Dynamics)

流れの物理。圧縮性流れ（音速・衝撃波・ノズル）から始め、`engineering/aerospace/` の
ロケット・航空機分析が依拠する理論を基礎から積み上げる。各文書は
「Daedalus内の応用」セクションで既存の分析ドキュメントへリンクする
（`foundations/README.md` の規約）。

## 文書一覧 (Documents)

1. [`compressible-flow-nozzles.md`](compressible-flow-nozzles.md) —
   圧縮性流れとノズル理論：マッハ数、面積–速度関係（デ・ラバルノズルの理由）、
   チョーク流量、推力式と最適膨張、排気速度と Isp（v_e ∝ √(T0/M̄)）、
   海面ノズル vs 真空ノズル。`engineering/aerospace/spacex-rocket-engines.md` の理論的基礎
2. [`shock-waves-hypersonics.md`](shock-waves-hypersonics.md) —
   衝撃波と極超音速流：波の突っ立ち、ランキン–ユゴニオ関係、斜め衝撃波と膨張扇、
   造波抗力、極超音速の新物理（薄い衝撃層、マッハ数独立、実在気体効果、輻射加熱）、
   鈍頭理論の再訪。再突入 (`../../../engineering/aerospace/reentry-thermal-protection.md`) の理論的基礎
3. [`incompressible-flow-boundary-layers.md`](incompressible-flow-boundary-layers.md) —
   非圧縮性流れとベルヌーイの定理、粘性とナビエ=ストークス方程式・レイノルズ数と相似則、
   ダランベールのパラドックスと Prandtl の境界層 (Blasius 解)、遷移と剥離、
   循環・クッタ条件による揚力と摩擦・圧力・誘導抗力の由来。
   `../../../engineering/aerospace/aircraft-engineering-fundamentals.md` の空力節の理論化
