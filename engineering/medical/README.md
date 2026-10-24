# 医療機器 (Medical Devices)

医療機器を題材とした工学分析。規制当局の審査を前提とする安全設計と、規制下のソフトウェア開発 (IEC 62304 など) を中心に、事故の歴史と規格・規制の要求の対応を扱う。`../automotive/` の機能安全 (ISO 26262) と並ぶ、安全規格ドメインの一つである。

## ドキュメント

- [`medical-device-safety.md`](medical-device-safety.md) — 医療機器の安全設計と規制下のソフトウェア開発 (medical device safety & regulated software)：薬機法・FDA 510(k)/PMA/De Novo・EU MDR、ISO 14971・IEC 62304 安全クラス・IEC 60601-1・IEC 62366-1・ISO 13485、Therac-25、輸液ポンプと安全保証ケース、ペースメーカの無線攻撃と 2017 年の更新、FD&C Act 524B と SBOM、SaMD・AI 医療機器と PCCP/IDATEN

## 今後のテーマ候補

- **医用電気機器の電気安全** — IEC 60601-1 の絶縁・漏れ電流・単一故障状態、EMC (IEC 60601-1-2)、警報システム。
- **植込み型機器の工学** — ペースメーカのペーシング方式、電池と寿命、MRI 対応、遠隔モニタリング。
- **医用画像と AI** — CT/MRI の信号処理、放射線科 AI の検証と市販後の性能監視。
