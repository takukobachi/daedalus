# 電磁気学 (Electromagnetism)

電荷と電磁場の理論。デジタル回路の「ビットの物理的実体」から、バス・無線・
電力系統まで、Daedalus のハードウェア分析のほぼ全域が依拠する物理を、
静電気学からマクスウェル方程式・伝送線路まで積み上げて文書化する。

## 文書一覧 (Documents)

1. [`charge-field-potential.md`](charge-field-potential.md) —
   電荷とクーロンの法則、電場とガウスの法則、電位、導体と静電遮蔽、
   キャパシタンス。DRAM・NAND・CMOS 動的電力の物理的土台。シリーズの起点
2. [`current-magnetostatics.md`](current-magnetostatics.md) —
   電流と電荷保存、オームの法則の微視的描像(ドルーデモデル)、ジュール熱、
   ローレンツ力、アンペールの法則。配線抵抗・IR ドロップ・
   エレクトロマイグレーション・モーターの物理
3. [`induction-inductance.md`](induction-inductance.md) —
   ファラデーの法則とレンツの法則、発電機と逆起電力・回生ブレーキ、
   自己/相互インダクタンスと変圧器、渦電流。寄生インダクタンス・
   di/dt ノイズ・電力変換の物理
4. [`maxwell-equations-em-waves.md`](maxwell-equations-em-waves.md) —
   変位電流、マクスウェル方程式への統合、電磁波の導出 (c = 1/√(μ₀ε₀))、
   偏波・スペクトル、ポインティングベクトルとアンテナ。無線通信と光の言語

## ロードマップ (Roadmap)

番号は推奨する読み書きの順序(依存順)。

5. 回路理論 (circuit theory) — 集中定数近似、RLC、インピーダンス、過渡応答。
   デカップリング・電源分配網 (PDN) の解析手段
6. 伝送線路とシグナルインテグリティ (transmission lines & signal integrity) —
   分布定数、特性インピーダンス、反射・終端。高速シリアルリンク
   (PCIe/USB/Ethernet) の物理層の共通土台
