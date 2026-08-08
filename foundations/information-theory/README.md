# 情報理論 (Information Theory)

「情報」を確率で定量化する理論。圧縮の下限（これ以上縮まない）と通信の上限
（これ以上速く送れない）という2つの限界を、同じ1つの量——エントロピー——から
導く2回シリーズ。通奏低音は、**限界が先に証明され、実装が後から追いつく**という
この分野特有の歴史である。Shannon が 1948 年に示した通信路容量に、実用符号
(LDPC/Turbo/Polar) が到達するまで約半世紀かかった。限界を知っていることは、
「いまの実装があとどれだけ改善しうるか」の相場観そのものであり、圧縮率・無線リンク・
ストレージ ECC・機械学習の損失関数を貫く共通の物差しを与える。

数学 (`../mathematics/`)・物理 (`../physics/`) と並ぶ foundations 直下の第三の系列
（`../README.md` の「数学・物理以外の基礎理論」の最初の1本）。

## 文書一覧 (Documents)

1. [`entropy-source-coding.md`](entropy-source-coding.md) —
   自己情報量と対数の必然性、エントロピー、結合・条件付きエントロピーと
   相互情報量、KL ダイバージェンス、Kraft 不等式と情報源符号化定理
   （可逆圧縮の下限）、クロスエントロピー損失とパープレキシティ
   （「圧縮＝予測」の等価性）。シリーズの起点
2. [`channel-capacity-coding.md`](channel-capacity-coding.md) —
   通信路モデル (BSC/BEC)、通信路符号化定理と容量 C = max I(X;Y)、
   AWGN と Shannon–Hartley 公式、Eb/N0 の究極限界 (−1.59 dB)、
   実用符号の系譜 (Hamming → 畳み込み → Turbo/LDPC/Polar) と
   「シャノン限界まであと何 dB」の読み方、rate-distortion と分離定理。最終回

## 今後のテーマ候補

- レート歪み理論の各論 (rate-distortion theory) — 歪み尺度と知覚品質の乖離、
  非可逆圧縮 ([`../../software/compression/`](../../software/compression/)) の
  理論的下限の深掘り
- 情報理論とセキュリティ (information-theoretic security) — 完全秘匿 (one-time pad)、
  エントロピー源の健全性評価 (min-entropy, NIST SP 800-90B)、サイドチャネルの
  情報量的定式化。`../../hardware/hardware-security/` との合流点
- Kolmogorov 複雑性と計算理論 — 「個々の系列の圧縮不可能性」という確率によらない
  情報量。将来の計算理論シリーズ（`../README.md` のロードマップ）との接続点
