# FPGA と再構成可能コンピューティング (FPGA & Reconfigurable Computing)

**製造後に回路そのものを書き換えられるデバイス**の分析。専用化スペクトラム
(CPU → GPU → DSA/ASIC) の中間に位置し、「少量・可変仕様・決定的低レイテンシ」の
谷を埋める FPGA を、ファブリック構造・設計フロー・高位合成・データセンター応用の
順に扱う。

スペクトラムの両隣は `../accelerators/` (GPU/TPU) が、FPGA の存在理由である
NRE・マスク費用は `../semiconductor-manufacturing/` が、SmartNIC/DPU としての
配備先は `../networking/` がそれぞれ扱う。

## ドキュメント

- [`fpga-architecture-flow.md`](fpga-architecture-flow.md) — FPGA のアーキテクチャと
  設計フロー：LUT/CLB と配線ファブリック (「本体は配線」)、ハード化ブロックの系譜
  (BRAM・DSP・SerDes → Zynq/Versal 型異種 SoC)、設計フロー (合成・配置配線・STA・
  タイミングクロージャ)、高位合成 (HLS) と開始間隔、部分再構成、
  Catapult / Azure SmartNIC のデータセンター応用、FPGA vs GPU vs ASIC 比較、
  ビットストリーム保護とマルチテナント FPGA のサイドチャネル

## 今後のテーマ候補

- FPGA での AI 推論 — 空間データフローアクセラレータ、量子化とデバイス内メモリ階層、
  DSL → HLS スタック。`../accelerators/tpu-systolic-arrays.md` のデータフロー
  分類との合流点
- ネットワーク処理の深掘り — P4 とマッチアクションパイプライン、SmartNIC/DPU の
  プログラミングモデル。`../networking/nic-switch-architecture.md` と
  `../../software/operating-systems/` のプログラマブルデータプレーン系譜の各論
- クラウド FPGA のセキュリティ各論 — シェル/ロール隔離、ビットストリーム静的検査と
  電気的サイドチャネルの攻防
