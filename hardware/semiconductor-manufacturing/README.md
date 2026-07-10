# 半導体製造 (Semiconductor Manufacturing)

チップが物理的にどう作られるかの分析。リソグラフィ (EUV)・プロセスノード・歩留まり・先端パッケージングを扱い、`../cpu-architecture/` 以降のすべての計算機システム分析の**物理的前提**を与える層である。アーキテクチャ側の大転回（マルチコア・ダークシリコン・チップレット・3D キャッシュ）を「製造物理からの強制」として読み解く。

## ドキュメント

- [`semiconductor-fabrication.md`](semiconductor-fabrication.md) — 半導体製造 (semiconductor fabrication)：スケーリングの軸乗り換え史（Dennard 終焉 → FinFET/GAA → EUV → チップレット）、リソグラフィとレイリーの式、EUV の光源・光学系・確率論的限界、プロセスノード名と実寸の乖離、歩留まりの数理 (Y = e^(−A·D₀)) とビニング、2.5D/3D パッケージング (CoWoS・ハイブリッドボンディング・UCIe)、製造の経済と地政学

## 今後のテーマ候補

- プロセス装置の各論 (process equipment) — エッチング (RIE/ALE)、成膜 (CVD/ALD)、CMP、
  イオン注入。「成膜→露光→エッチ」ループの露光以外の 2/3
- テストと DFT (test & design-for-test) — スキャンチェーン、ATE、ウェハプローブ、
  BIST。歩留まり分析の観測装置であり、`../hardware-security/` のデバッグポート攻撃面と表裏
- 計算リソグラフィ深掘り (computational lithography) — OPC/ILT の逆問題、マスク製作、
  GPU による計算リソ。設計データが物理になる最後の変換層
- メモリ製造 (memory fabrication) — DRAM のキャパシタ形成、3D NAND の高アスペクト比
  エッチ。`../cpu-architecture/dram-memory-controllers.md`・`../storage-devices/` の製造側
