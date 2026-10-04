# 組み込み・リアルタイム OS (Embedded & Real-Time OS)

「平均を速くする」汎用 OS に対し、「最悪を保証する」側の実行基盤。ハード/ソフト
リアルタイムの理論（スケジューリング可能性・WCET）、RTOS カーネルの設計空間
（フラットアドレスの小型カーネルからマイクロカーネル・検証済みカーネルまで）、
そして産業実装としての AUTOSAR を扱う。`operating-systems/` の対照編であり、
`engineering/automotive/`（ECU）・`engineering/robotics/`（制御計算機）と software 側を
束ねる橋になる領域。

## ドキュメント

- [`realtime-fundamentals.md`](realtime-fundamentals.md) — リアルタイムシステム基礎：
  ハード/ソフトリアルタイムの定義、タスクモデルと固定優先度/EDF スケジューリング
  （Liu & Layland 限界・応答時間解析）、優先度逆転と継承（Mars Pathfinder）、
  レイテンシの解剖、WCET 解析とマルチコアの困難
- [`rtos-architectures.md`](rtos-architectures.md) — RTOS アーキテクチャ：カーネル設計の
  スペクトラム（ベアメタル → FreeRTOS/Zephyr → QNX マイクロカーネル → seL4 →
  ARINC 653 パーティショニング）、PREEMPT_RT Linux との使い分け、混合クリティカリティ
  と AMP 構成、安全規格認証（IEC 61508 / DO-178C / ISO 26262）
- [`autosar-vehicle-software.md`](autosar-vehicle-software.md) — AUTOSAR と車載
  ソフトウェア基盤：OSEK の系譜、Classic Platform（BSW/RTE/SWC・静的構成）、
  Adaptive Platform（POSIX・SOME/IP・実行管理）、干渉からの自由と E2E 保護、
  SDV 化と内製 Vehicle OS の潮流

## 今後のテーマ候補

- リアルタイムネットワーキングの各論 — TSN/EtherCAT のソフトウェアスタック側
  （`engineering/automotive/automotive-ethernet-tsn.md` のホスト実装側）
- 検証とスケジューラビリティ解析の実務 — トレースベース検証、形式手法の適用範囲
- 省電力とリアルタイムの両立 — DVFS とデッドライン保証の相互作用
  （`operating-systems/power-management.md` との合流点）
