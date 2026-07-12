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
5. [`circuit-theory.md`](circuit-theory.md) —
   集中定数近似とその適用限界、キルヒホッフの法則の由来と破れ、テブナン等価、
   フェーザとインピーダンス、共振と Q 値・PDN の反共振。
   デカップリング・電源分配網 (PDN) の解析手段

6. [`transmission-lines-signal-integrity.md`](transmission-lines-signal-integrity.md) —
   電信方程式と分布定数、特性インピーダンス、反射と終端、表皮効果・
   誘電損失と符号間干渉 (ISI)、差動伝送とイコライゼーション (SerDes)。
   高速シリアルリンク (PCIe/USB/Ethernet) の物理層の共通土台。シリーズ最終回

当初計画した6本のシリーズはこれで完結。以降のテーマ(アンテナ工学の深掘り、
導波路・光学など)は必要が生じた時点でロードマップを再開する。
