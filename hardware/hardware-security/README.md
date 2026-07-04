# ハードウェアセキュリティ (Hardware Security)

ハードウェアセキュリティは、`cpu-architecture/` の投機実行・キャッシュ機構と、software 側のセキュリティ分析（iOS/Android のプラットフォーム防御）とを橋渡しする層である。ここで一貫して問うのは「性能最適化がそのまま情報チャネルになる」という構造、すなわち命令セット (ISA) が保証しないマイクロアーキテクチャ状態 (microarchitectural state) の副作用が、いかにして信頼境界 (trust boundary) を貫通するかである。同時に、その漏洩を封じ込めるための隔離機構（TEE, Secure Boot, 信頼の起点）を、攻撃と防御の対で整理する。

## 構成

- [`side-channels-tee-secure-boot.md`](side-channels-tee-secure-boot.md) — サイドチャネル・TEE・信頼の起点 (side channels, TEE & root of trust)

## 今後のテーマ候補

- フォールト注入深掘り (fault injection / glitching) — 電圧・クロックグリッチ、レーザ注入の物理と、センサ・冗長化・定数時間による対策の設計論
- PUF と真性乱数生成器 (PUF & TRNG) — 物理的複製困難性 (physical unclonable function) と鍵生成、エントロピー源の健全性
- サプライチェーンとハードウェアトロイ (supply chain & hardware Trojan) — 設計・製造・実装の各段での改竄と検知、来歴 (provenance) の保証
