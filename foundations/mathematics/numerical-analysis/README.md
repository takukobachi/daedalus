# 数値解析 (Numerical Analysis)

数学を計算機の上で実行するときに何が起きるかを扱う分野。計算機は実数を扱えず、
有限個の浮動小数点数と有限回の演算しか持たない。そのため、どの計算結果にも
丸め誤差と打ち切り誤差が含まれる。このシリーズの主題は2つある。
1つは**誤差を「問題の敏感さ」(条件数)と「算法の品質」(安定性)に分けて
定量化すること**で、これにより「計算できた」と「その答を何桁信用できるか」を
区別できる。もう1つは**収束の速さ(精度・次数)と破綻しないこと(安定性)が
別の概念であること**で、ODE の硬さや長時間積分のエネルギー誤差は後者の問題である。
SIMD・GPU の数値形式、機械学習の混合精度、軌道伝播やロボットのシミュレーション、
最適化ソルバの内側の計算は、すべてこの2つの言葉で読める。

## 文書一覧 (Documents)

1. [`floating-point-error-analysis.md`](floating-point-error-analysis.md) —
   IEEE 754-2019 の形式と丸めモード、機械イプシロンと単位丸め、非正規化数・NaN・Inf、
   bfloat16 と FP8 (E4M3/E5M2, OCP OFP8)、桁落ちと条件数、前進/後退誤差、
   総和の誤差解析(再帰・ペアワイズ・Kahan 補償総和)、FMA、非結合性と
   並列縮約の再現性、混合精度と損失スケーリング
2. [`equations-ode-numerical-methods.md`](equations-ode-numerical-methods.md) —
   求根法と収束次数(二分法・ニュートン法・割線法)、反復線形ソルバ
   (Jacobi・Gauss–Seidel・共役勾配法・Krylov 部分空間法・前処理)、ODE 積分法の
   次数と安定領域、硬い方程式と A/L 安定性、埋め込み型 RK による刻み幅制御、
   シンプレクティック積分と修正ハミルトニアン

## 推奨読書順 (Recommended Reading Order)

1 → 2 の順に読む。第1回の単位丸め・条件数・後退誤差の語彙を、第2回で反復法と
時間積分に適用する構成である。前提として
[`../linear-algebra/matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md)
(直接法と後退誤差解析の初出)と
[`../calculus-analysis/ordinary-differential-equations.md`](../calculus-analysis/ordinary-differential-equations.md)
(初期値問題と安定性)を先に読んでおくとよい。

## 今後のテーマ候補 (Roadmap)

- 偏微分方程式の数値解法 (numerical PDEs) — 有限差分・有限要素・有限体積法、
  CFL 条件と Lax の同値定理。流体 (`../../physics/fluid-dynamics/`)・
  熱・電磁界解析の計算側
- 数値最適化の実装各論 (numerical optimization) — 直線探索と信頼領域、
  準ニュートン法 (L-BFGS)、内点法の線形代数。`../optimization/` の今後のテーマ候補と合流する
- 固有値計算と大規模疎行列 (eigenvalue algorithms) — QR 法、Lanczos/Arnoldi 法、
  PageRank やモード解析への応用
- 区間演算と精度保証付き数値計算 (verified computing) — 丸め方向の制御による
  誤差の厳密な包含
