# ハードウェア調査ロードマップ (Hardware Research Roadmap)

ハードウェア領域で「次に何を調査すべきか」を体系化した計画文書。既存ドキュメントの空白を埋め、相互参照のネットワークを強化することを主眼とする。各READMEに散在していた「今後のトピック」を一元化し、優先順位と根拠 (rationale) を付与した。

本書は分析文書 (analysis document) ではなく、調査バックログ (backlog) を管理するための索引である。各トピックに着手した時点で、対応するサブディレクトリに正式な分析ドキュメントを起こし、本書から相互リンクする。

## 現状の棚卸し (Inventory)

既に分析済みの領域:

- **`cpu-architecture/`** — キャッシュ階層 (cache hierarchy)、メモリ順序とアトミック (memory ordering & atomics)、仮想記憶・MMU・TLB (virtual memory)、ISA入門 (RISC-V)
- **`hw-sw-interface/`** — MMIO・DMA・割り込み (device I/O)、ABI・呼び出し規約・システムコール (ABI & syscalls)
- **`automotive/`** — 車載ネットワークと ECU (in-vehicle networks)、機能安全 (functional safety, ISO 26262)

これらは「静的な構造」を中心に整備されており、**動的な実行機構・物理経路・横断的な安全/セキュリティ**に空白がある。以下はその空白を埋める方針で構成した。

## トピック一覧 (Topics)

### A. CPUアーキテクチャの深掘り (cpu-architecture)

| トピック | 調査すべき理由 (rationale) | 関連既存ドキュメント |
|---|---|---|
| パイプライン・スーパースカラ・アウトオブオーダ実行 (pipeline / superscalar / OoO) | 既存は静的構造が中心。動的実行機構を加えると性能モデルが完成する | `cache-memory-hierarchy.md` |
| 分岐予測と投機実行 (branch prediction / speculation) | 性能の要であると同時に、Spectre/Meltdown 等マイクロアーキテクチャ攻撃への橋渡し | `cache-memory-hierarchy.md` (side channel) |
| 割り込み・例外・トラップ (interrupts / exceptions / traps) | 特権レベルとコンテキストスイッチの基盤。hw-sw-interface と相互参照できる | `mmio-dma-interrupts.md`, `abi-syscalls.md` |
| SIMD・ベクトル演算 (SIMD / vector: AVX, RVV, SVE) | データ並列の実務的中核。並行性の議論を補完する | `memory-ordering-atomics.md` |
| マルチコア・キャッシュコヒーレンス (cache coherence: MESI 系) | メモリ順序ドキュメントの「ハードウェア側の裏付け」。最も自然な次の一手 | `memory-ordering-atomics.md` |

### B. HW/SW境界の層を拡張 (hw-sw-interface)

| トピック | 調査すべき理由 (rationale) | 関連既存ドキュメント |
|---|---|---|
| バス・相互接続 (interconnect: PCIe, AMBA/AXI, CXL) | MMIO/DMA の「その先」。CPUとデバイスを繋ぐ物理経路が未カバー | `mmio-dma-interrupts.md` |
| ブート・初期化シーケンス (boot: reset vector → firmware → OS) | 電源投入からコード実行までの空白を埋める | `abi-syscalls.md` |
| クロック・電源・リセット管理 (clock / power / reset: PLL, DVFS) | 省電力と性能の物理的制約。組込み・自動車にも波及 | — |
| ストレージ・不揮発メモリインタフェース (eMMC, NVMe, flash wear) | データ永続化のハードウェア側 | `virtual-memory-mmu.md` |

### C. 自動車の具体化 (automotive)

| トピック | 調査すべき理由 (rationale) | 関連既存ドキュメント |
|---|---|---|
| パワートレイン制御 (powertrain: エンジン/モータ, トランスミッション) | 物理システムと制御ソフトの接点。READMEに既出 | `in-vehicle-networks.md` |
| 先進運転支援 (ADAS: センサ融合 sensor fusion, SoC) | 高性能演算と安全要件が交差する領域。READMEに既出 | `functional-safety-iso26262.md` |
| 車載ソフト基盤 (AUTOSAR, SOME/IP) | ECU上のソフトウェアアーキテクチャ。READMEに既出 | `in-vehicle-networks.md` |
| 車載セキュリティ (HSM, Secure Boot, ISO/SAE 21434) | 機能安全 (26262) と対になるもう一本の柱 | `functional-safety-iso26262.md` |

### D. 新規ドメイン (new domains)

READMEトップで宣言済みの拡張方向。範囲が広く独立性が高いため、土台が固まってから着手する。

| トピック | 調査すべき理由 (rationale) |
|---|---|
| 航空 (aircraft: flight control, avionics, DO-178C/DO-254, 冗長化) | 安全クリティカル系の最高峰。冗長設計は自動車・電力にも転用できる |
| 電力・発電 (power: 電力変換 inverter/converter, 送配電, 系統制御) | 物理工学システムへの展開。パワーエレクトロニクスは車載にも接続 |

### E. 横断トピック (cross-cutting)

分野をまたいで再利用される基盤。ハブとして相互参照価値が高い。

| トピック | 調査すべき理由 (rationale) |
|---|---|
| リアルタイム性のHW支援 (timer, WCET, 割り込みレイテンシ) | 自動車・航空の安全要件と直結 |
| ハードウェアセキュリティ全般 (TEE/TrustZone/SGX, side channel, Root of Trust) | CPU・自動車・航空すべてに接続するハブ |
| ハードウェアアクセラレータ (GPU/NPU/DSP, FPGA) | ADAS・AI推論の演算基盤。アーキテクチャ比較の対象 |
| 信頼性・故障モデル (ECC, soft error/SEU, 冗長設計) | 自動車・航空の安全文書と知見を共有できる |

## 優先順位 (Prioritization)

1. **最優先 — A群・B群の補完**: 既存ドキュメント間の明らかな空白を埋め、相互参照ネットワークを強化する。とりわけ「キャッシュコヒーレンス → memory-ordering」「バス/相互接続 → MMIO/DMA」は接続効果が大きい。
2. **次点 — E群のハードウェアセキュリティ**: CPU・自動車・航空のすべてに接続するハブになる。分岐予測/投機実行 (A群) と合わせるとサイドチャネルの議論が一本につながる。
3. **その後 — D群の新ドメイン**: 範囲が広く独立性が高いので、横断トピックという共通言語が整ってから着手する。

## 運用 (Workflow)

- トピックに着手したら、対応するサブディレクトリに分析ドキュメント (Level 3 を既定) を作成し、最寄りのREADME索引と本書を更新する。
- 着手済み・完了したトピックには本書で完了リンクを張り、バックログと成果物の対応を保つ。
