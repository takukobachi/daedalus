# 繰り返し引用される教科書の索引 (Index of Frequently Cited Textbooks)

## 概要 (Overview)

Daedalus の分析文書と foundations 文書の参考文献のうち、3 本以上の文書から引用されている教科書をまとめた索引である。各書籍について、書誌情報、引用している文書、その文書が何のために引用しているかを示す。同じ書籍を引く文書どうしは、同じ理論的背景を共有していることになるので、この索引は文書間の関係をたどる手がかりにもなる。

「各文書での用途」は、引用している文書の参考文献に書かれた注記を要約したものである。章番号はその文書の参考文献に記載がある場合だけ示した。書籍の本文は転載せず、内容の確認は各文書の該当箇所と原典で行う。集計の基準日は 2026-10-11 で、引用の追加に応じて更新する。

## 計算機アーキテクチャ (Computer Architecture)

### Hennessy & Patterson『Computer Architecture: A Quantitative Approach』

Hennessy, J. L., & Patterson, D. A. (2019). *Computer Architecture: A Quantitative Approach* (6th ed.). Morgan Kaufmann.

`hardware/cpu/` のほぼ全文書が依拠する標準教科書である。

| 文書 | 用途 |
|---|---|
| [`cpu/isa-riscv-intro.md`](../../hardware/cpu/isa-riscv-intro.md) | RISC 設計思想と ISA/マイクロアーキテクチャの分離 (付録 A) |
| [`cpu/pipeline-superscalar-ooo.md`](../../hardware/cpu/pipeline-superscalar-ooo.md) | パイプライン (付録 C)、ILP とアウトオブオーダ実行 (Ch. 3) |
| [`cpu/branch-prediction-speculation.md`](../../hardware/cpu/branch-prediction-speculation.md) | 分岐予測・投機実行・ROB (Ch. 3)、分岐頻度とミスペナルティの代表値 |
| [`cpu/interrupts-exceptions-traps.md`](../../hardware/cpu/interrupts-exceptions-traps.md) | パイプラインにおける例外と精密性 (Ch. 3, 付録 C) |
| [`cpu/cache-memory-hierarchy.md`](../../hardware/cpu/cache-memory-hierarchy.md) | メモリ階層、キャッシュ構成、3C 分類 (Ch. 2, 付録 B) |
| [`cpu/virtual-memory-mmu.md`](../../hardware/cpu/virtual-memory-mmu.md) | 仮想記憶と TLB (Ch. 2, 付録 B) |
| [`cpu/cache-coherence.md`](../../hardware/cpu/cache-coherence.md) | スヌープ型・ディレクトリ型コヒーレンス、分散共有メモリ (Ch. 5) |
| [`cpu/simd-vector.md`](../../hardware/cpu/simd-vector.md) | ベクトル・SIMD・GPU のデータレベル並列性 (Ch. 4) |
| [`cpu/dram-memory-controllers.md`](../../hardware/cpu/dram-memory-controllers.md) | メモリ階層とメモリ壁の定量的整理 |
| [`cpu/performance-monitoring-units.md`](../../hardware/cpu/performance-monitoring-units.md) | 性能評価とパイプラインストールの定量的整理 |
| [`accelerators/gpu-simt-architecture.md`](../../hardware/accelerators/gpu-simt-architecture.md) | GPU のデータレベル並列性 (Ch. 4) |
| [`accelerators/tpu-systolic-arrays.md`](../../hardware/accelerators/tpu-systolic-arrays.md) | Dennard 則の終焉と専用アーキテクチャ (DSA) への転換 |

### Sorin, Hill & Wood『A Primer on Memory Consistency and Cache Coherence』

Sorin, D. J., Hill, M. D., & Wood, D. A. (2020). *A Primer on Memory Consistency and Cache Coherence* (2nd ed.). Morgan & Claypool. DOI: 10.2200/S00962ED2V01Y201910CAC049

コヒーレンス (同じアドレスの値の一致) と一貫性 (異なるアドレス間の順序) を区別して整理した教科書である。

| 文書 | 用途 |
|---|---|
| [`cpu/cache-coherence.md`](../../hardware/cpu/cache-coherence.md) | コヒーレンスと一貫性の区別、スヌープ型・ディレクトリ型、MESI 系プロトコル |
| [`cpu/memory-ordering-atomics.md`](../../hardware/cpu/memory-ordering-atomics.md) | TSO・弱順序とコヒーレンスの関係 |
| [`cpu/cache-memory-hierarchy.md`](../../hardware/cpu/cache-memory-hierarchy.md) | MESI 系プロトコル |
| [`cpu/numa-multi-socket.md`](../../hardware/cpu/numa-multi-socket.md) | ディレクトリ型コヒーレンスとディレクトリキャッシュの回収 |

### Bryant & O'Hallaron『Computer Systems: A Programmer's Perspective』

Bryant, R. E., & O'Hallaron, D. R. (2016). *Computer Systems: A Programmer's Perspective* (3rd ed.). Pearson.

ハードウェアの機構をプログラマの視点から説明する教科書である。

| 文書 | 用途 |
|---|---|
| [`cpu/cache-memory-hierarchy.md`](../../hardware/cpu/cache-memory-hierarchy.md) | 局所性、行優先の走査、メモリマウンテン (Ch. 6) |
| [`cpu/interrupts-exceptions-traps.md`](../../hardware/cpu/interrupts-exceptions-traps.md) | 割り込み・トラップ・フォールト・アボートの分類 (Ch. 8) |
| [`hw-sw-interface/abi-syscalls.md`](../../hardware/hw-sw-interface/abi-syscalls.md) | 手続き呼び出しとスタックフレーム (Ch. 3)、システムコール (Ch. 8) |

## ソフトウェアとデータシステム (Software & Data Systems)

### Kleppmann『Designing Data-Intensive Applications』

Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. ISBN 978-1-449-37332-0

複製、分割、一貫性、ストリーム処理を横断して整理した解説書である。Daedalus では二次資料として語彙の整理に使い、個別の主張は論文や一次資料で確認している。

| 文書 | 用途 |
|---|---|
| [`distributed-systems/replication-partitioning.md`](../../software/distributed-systems/replication-partitioning.md) | 複製 (Ch. 5) と分割 (Ch. 6) の体系的整理 |
| [`messaging/distributed-logs-delivery-semantics.md`](../../software/messaging/distributed-logs-delivery-semantics.md) | ログ、複製、exactly-once の横断的整理 |
| [`cloud/event-driven-architecture.md`](../../software/cloud/event-driven-architecture.md) | ストリーム処理と導出データ (Ch. 11–12)、outbox・CDC・イベントソーシング |
| [`object-storage/object-storage-architecture.md`](../../software/object-storage/object-storage-architecture.md) | 複製・分割・一貫性の語彙の整理 |
| [`networking/rpc-serialization.md`](../../software/networking/rpc-serialization.md) | 形式の比較と、後方互換・前方互換の枠組み (Ch. 4) |

## 制御 (Control)

### Åström & Murray『Feedback Systems』

Åström, K. J., & Murray, R. M. (2021). *Feedback Systems: An Introduction for Scientists and Engineers* (2nd ed.). Princeton University Press. ISBN 978-0-691-19398-4

制御理論シリーズの主参照であり、ロボットの制御とソフトウェアのフィードバックループの分析からも引用されている。

| 文書 | 用途 |
|---|---|
| [`control-theory/feedback-transfer-functions.md`](../../foundations/control-theory/feedback-transfer-functions.md) | フィードバックの原理、感度関数、PID と実装上の工夫 |
| [`control-theory/frequency-response-stability-margins.md`](../../foundations/control-theory/frequency-response-stability-margins.md) | ナイキスト判別、安定余裕と最大感度、ループ整形 |
| [`control-theory/state-space-estimation.md`](../../foundations/control-theory/state-space-estimation.md) | 状態空間、可制御性・可観測性、状態フィードバックとオブザーバ |
| [`robotics/control-systems.md`](../../engineering/robotics/control-systems.md) | PID、状態フィードバック、LQR、安定余裕 |
| [`cloud/autoscaling-capacity-control.md`](../../software/cloud/autoscaling-capacity-control.md) | フィードバック、フィードフォワード、積分ワインドアップ、カスケード制御 |

### Franklin, Powell & Emami-Naeini『Feedback Control of Dynamic Systems』

Franklin, G. F., Powell, J. D., & Emami-Naeini, A. (2019). *Feedback Control of Dynamic Systems* (8th ed.). Pearson.

| 文書 | 用途 |
|---|---|
| [`control-theory/feedback-transfer-functions.md`](../../foundations/control-theory/feedback-transfer-functions.md) | 極・零点と応答の対応、非最小位相系の事例 |
| [`control-theory/frequency-response-stability-margins.md`](../../foundations/control-theory/frequency-response-stability-margins.md) | 位相余裕と減衰比の近似、進み・遅れ補償 |
| [`control-theory/state-space-estimation.md`](../../foundations/control-theory/state-space-estimation.md) | 極配置とオブザーバの極の選び方 |

## 数学 (Mathematics)

### Golub & Van Loan『Matrix Computations』

Golub, G. H., & Van Loan, C. F. (2013). *Matrix Computations* (4th ed.). Johns Hopkins University Press.

数値線形代数の標準的な参考書であり、線形代数シリーズの数値計算側の主参照である。

| 文書 | 用途 |
|---|---|
| [`linear-algebra/vectors-matrices-linear-maps.md`](../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md) | 行列計算のブロック化 |
| [`linear-algebra/eigenvalues-diagonalization.md`](../../foundations/mathematics/linear-algebra/eigenvalues-diagonalization.md) | 固有値の数値計算 |
| [`linear-algebra/matrix-factorizations-numerical.md`](../../foundations/mathematics/linear-algebra/matrix-factorizations-numerical.md) | LU・Cholesky・QR 分解と最小二乗法 |
| [`linear-algebra/svd-low-rank.md`](../../foundations/mathematics/linear-algebra/svd-low-rank.md) | SVD の存在・構成・数値計算 |
| [`linear-algebra/tensors-multilinear.md`](../../foundations/mathematics/linear-algebra/tensors-multilinear.md) | 行列側の対応物 (SVD・ブロック化) |

### Strang『Introduction to Linear Algebra』

Strang, G. (2016). *Introduction to Linear Algebra* (5th ed.). Wellesley-Cambridge Press.

線形代数シリーズの副読本で、幾何的な直観と応用の視点を補う。

| 文書 | 用途 |
|---|---|
| [`linear-algebra/vectors-matrices-linear-maps.md`](../../foundations/mathematics/linear-algebra/vectors-matrices-linear-maps.md) | 幾何的直観と応用の視点 |
| [`linear-algebra/eigenvalues-diagonalization.md`](../../foundations/mathematics/linear-algebra/eigenvalues-diagonalization.md) | 対角化の応用 (差分方程式、マルコフ行列) |
| [`linear-algebra/svd-low-rank.md`](../../foundations/mathematics/linear-algebra/svd-low-rank.md) | SVD とその応用 |

## 物理とデバイス (Physics & Devices)

### Sze & Ng『Physics of Semiconductor Devices』

Sze, S. M., & Ng, K. K. (2007). *Physics of Semiconductor Devices* (3rd ed.). Wiley. DOI: 10.1002/0470068329

量子力学・固体物理シリーズのデバイス側の主参照である。

| 文書 | 用途 |
|---|---|
| [`quantum-solid-state/band-theory-semiconductors.md`](../../foundations/physics/quantum-solid-state/band-theory-semiconductors.md) | ドーピングとキャリア統計 |
| [`quantum-solid-state/pn-junction-diode.md`](../../foundations/physics/quantum-solid-state/pn-junction-diode.md) | 接合の形成、ダイオード方程式、降伏 |
| [`quantum-solid-state/mosfet-cmos.md`](../../foundations/physics/quantum-solid-state/mosfet-cmos.md) | MOS 構造と MOSFET の特性 |
| [`quantum-solid-state/tunneling-nonvolatile-memory.md`](../../foundations/physics/quantum-solid-state/tunneling-nonvolatile-memory.md) | トンネリングと不揮発メモリのデバイス |
| [`quantum-solid-state/optoelectronics.md`](../../foundations/physics/quantum-solid-state/optoelectronics.md) | フォトダイオード、LED、レーザー |
| [`electromagnetism/charge-field-potential.md`](../../foundations/physics/electromagnetism/charge-field-potential.md) | 誘電率と絶縁膜の物性値 |

### Purcell & Morin『Electricity and Magnetism』

Purcell, E. M., & Morin, D. J. (2013). *Electricity and Magnetism* (3rd ed.). Cambridge University Press.

電磁気学シリーズで、物理的な描像を補う参考書として使われている。

| 文書 | 用途 |
|---|---|
| [`electromagnetism/charge-field-potential.md`](../../foundations/physics/electromagnetism/charge-field-potential.md) | 導体と遮蔽の物理的直観 |
| [`electromagnetism/current-magnetostatics.md`](../../foundations/physics/electromagnetism/current-magnetostatics.md) | 磁場を特殊相対論から導く視点 |
| [`electromagnetism/induction-inductance.md`](../../foundations/physics/electromagnetism/induction-inductance.md) | 誘導と交流回路 |
| [`electromagnetism/maxwell-equations-em-waves.md`](../../foundations/physics/electromagnetism/maxwell-equations-em-waves.md) | 電磁波の導出と物理的描像 |

### Bogatin『Signal and Power Integrity — Simplified』

Bogatin, E. (2018). *Signal and Power Integrity — Simplified* (3rd ed.). Prentice Hall.

シグナルインテグリティの理論 (foundations) と工学 (hardware) の両方から主参照として引用されている。

| 文書 | 用途 |
|---|---|
| [`electromagnetism/transmission-lines-signal-integrity.md`](../../foundations/physics/electromagnetism/transmission-lines-signal-integrity.md) | 反射、終端、実装の設計則 |
| [`electromagnetism/circuit-theory.md`](../../foundations/physics/electromagnetism/circuit-theory.md) | 電源分配網 (PDN)、反共振、実装の寄生成分 |
| [`signal-integrity/signal-integrity-engineering.md`](../../hardware/signal-integrity/signal-integrity-engineering.md) | 反射、クロストーク、ビアスタブ、PDN、実装の設計則 |
| [`signal-integrity/serdes-link-architecture.md`](../../hardware/signal-integrity/serdes-link-architecture.md) | チャネル側との接続 |

## 航空宇宙 (Aerospace)

### Sutton & Biblarz『Rocket Propulsion Elements』

Sutton, G. P., & Biblarz, O. (2016). *Rocket Propulsion Elements* (9th ed.). Wiley.

ロケット推進の標準的な教科書である。SpaceX の機体固有の事実は、各文書で公式資料によって別に確認している。

| 文書 | 用途 |
|---|---|
| [`fluid-dynamics/compressible-flow-nozzles.md`](../../foundations/physics/fluid-dynamics/compressible-flow-nozzles.md) | 推力の式、理想排気速度、膨張比の設計、剥離の限界 |
| [`aerospace/rocket-structures-tanks.md`](../../engineering/aerospace/rocket-structures-tanks.md) | 推進剤の質量比、タンク構造、加圧系 |
| [`aerospace/propellant-fluid-systems.md`](../../engineering/aerospace/propellant-fluid-systems.md) | 加圧方式 (不活性ガス加圧と自生加圧) |
| [`aerospace/combustion-instability.md`](../../engineering/aerospace/combustion-instability.md) | 噴射器の設計と安定性、ガス-ガス噴射の応答特性 |
