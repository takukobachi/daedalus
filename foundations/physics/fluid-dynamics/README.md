# 流体力学 (Fluid Dynamics)

流れの物理。圧縮性流れ（音速・衝撃波・ノズル）から始め、`hardware/aerospace/` の
ロケット・航空機分析が依拠する理論を基礎から積み上げる。各文書は
「Daedalus内の応用」セクションで既存の分析ドキュメントへリンクする
（`foundations/README.md` の規約）。

## 文書一覧 (Documents)

1. [`compressible-flow-nozzles.md`](compressible-flow-nozzles.md) —
   圧縮性流れとノズル理論：マッハ数、面積–速度関係（デ・ラバルノズルの理由）、
   チョーク流量、推力式と最適膨張、排気速度と Isp（v_e ∝ √(T0/M̄)）、
   海面ノズル vs 真空ノズル。`hardware/aerospace/spacex-rocket-engines.md` の理論的基礎
2. [`shock-waves-hypersonics.md`](shock-waves-hypersonics.md) —
   衝撃波と極超音速流：波の突っ立ち、ランキン–ユゴニオ関係、斜め衝撃波と膨張扇、
   造波抗力、極超音速の新物理（薄い衝撃層、マッハ数独立、実在気体効果、輻射加熱）、
   鈍頭理論の再訪。再突入 (`../../../hardware/aerospace/reentry-thermal-protection.md`) の理論的基礎

## 今後のロードマップ (Roadmap)

3. **非圧縮性流れと境界層** — ベルヌーイ、粘性、境界層と剥離、揚力と抗力の由来。
   `../../../hardware/aerospace/aircraft-engineering-fundamentals.md` の空力節の理論化
