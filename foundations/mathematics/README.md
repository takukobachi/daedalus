# 数学 (Mathematics)

分析を支える数学の基礎。定義・定理を積み上げる本文に加え、各文書は
「Daedalus内の応用」セクションで既存の分析ドキュメントへリンクする
（`foundations/README.md` の規約）。

## 構成 (Structure)

- `linear-algebra/` — 線形代数(ベクトル空間、線形写像、行列積の計算構造 …)。
  SIMD (`../../hardware/cpu-architecture/simd-vector.md`)、
  GPU・Tensor Core (`../../hardware/gpu-accelerators/`)、
  機械学習システム (`../../software/ml-systems/`) の計算の実体

## 今後のテーマ候補

優先度は「既存の分析ドキュメントとの相互参照が何本張れるか」を基準に付けている。

### 優先度 高 (既存分析に直結)

- 確率・統計 (probability & statistics) — 確率分布、期待値、推定、マルコフ連鎖。
  分岐予測 (`../../hardware/cpu-architecture/branch-prediction-speculation.md`)、
  製造歩留まり (`../../hardware/semiconductor-manufacturing/`)、
  待ち行列・パフォーマンス分析 (`../../software/observability/`) の言語

### 優先度 中

- フーリエ解析・信号処理 (Fourier analysis & signal processing) — フーリエ級数/変換、
  サンプリング定理、FFT。無線
  (`../../hardware/peripherals-connectivity/wifi-architecture.md`)、オーディオ
  (`../../hardware/peripherals-connectivity/audio-subsystem.md`)、
  画像・ディスプレイ (`../../hardware/peripherals-connectivity/display-pipeline.md`) の土台
- 微分積分・解析 (calculus & analysis) — 極限、微分方程式、多変数。物理学全般と
  制御の言語。それ自体が応用に直結するというより、他分野の前提として必要
- 離散数学・論理 (discrete mathematics & logic) — ブール代数、グラフ、オートマトン。
  論理回路・ISA (`../../hardware/cpu-architecture/`)、プロトコルの状態機械
  (`../../software/networking/`)、データベース (`../../software/databases/`) の骨格

### 優先度 低 (将来の拡張)

- 数論・代数 (number theory & algebra) — 合同算術、群・環・体。暗号
  (`../../software/security/`) と誤り訂正符号の前提
- 数値解析 (numerical analysis) — 浮動小数点、誤差解析、反復法。
  科学技術計算と機械学習の数値的基盤
