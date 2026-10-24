# ハードウェア信頼性 (Hardware Reliability)

フリート規模で見たハードウェアの信頼性。ECC やバスタブ曲線のような部品単位の議論は
[`../cpu/dram-memory-controllers.md`](../cpu/dram-memory-controllers.md)・
[`../storage/nand-flash-ftl.md`](../storage/nand-flash-ftl.md)・
[`../../foundations/mathematics/probability-statistics/reliability-theory.md`](../../foundations/mathematics/probability-statistics/reliability-theory.md)
が扱う。ここでは、数十万台のサーバや数万基のアクセラレータを運用したときに、
ハードウェアの誤りがどの経路でソフトウェアに届き、どこで検出・隔離できるかを扱う。

## ドキュメント

- [`fleet-hardware-reliability.md`](fleet-hardware-reliability.md) — フリート規模のハードウェア信頼性：
  故障・誤り・障害の分類、ソフトエラーと摩耗とマージナル欠陥、メモリ誤りのフィールド実測、
  CPU のサイレントデータ破損 (mercurial cores)、大規模学習の故障内訳、フリート検査・ABFT・
  隔離と RMA、誤り注入と暗号
- [`ras-machine-check.md`](ras-machine-check.md) — RAS アーキテクチャとマシンチェック：
  CE・UCNA・SRAO・SRAR の分類とポイズン、x86 MCA と AMD の遅延誤り、Arm RAS 拡張と SDEI、
  ファームウェア優先と OS 優先 (APEI・CPER)、スクラブ・スペアリング・ミラーリング、
  Linux の EDAC・memory_failure・SIGBUS、PCIe AER と DPC、CE の嵐、予測交換

## 今後のテーマ候補

- 電源・冷却と故障率 — 温度・電圧変動・電源品質がフィールド故障率に与える影響の実測。
  [`../../engineering/datacenters/`](../../engineering/datacenters/) の施設設計と対で読む
