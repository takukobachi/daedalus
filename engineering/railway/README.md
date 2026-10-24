# 鉄道 (Railway)

鉄道を題材とした工学分析。信号保安と列車制御を中心に、閉塞・連動・ATS/ATC・無線式列車制御のアーキテクチャと、「故障したら止まれば安全」という安全側を持つ系の設計原理を扱う。

自動車の機能安全（[`../automotive/functional-safety-iso26262.md`](../automotive/functional-safety-iso26262.md)）とは安全状態の位置が異なる。その違いを比べるための、もう 1 つの輸送ドメインでもある。

## ドキュメント

- [`railway-signaling-control.md`](railway-signaling-control.md) — 鉄道信号と列車制御 (railway signaling & train control)：固定閉塞と在線検知（閉電路式軌道回路・車軸カウンタ型）、連動装置の鎖錠、フェールセーフ（落下リレー・2oo2/2oo3）、ATS-P・アナログ/デジタル ATC、ATACS・ETCS（Level 3 の Level 2 への統合）・CBTC と移動閉塞、EN 50126/50129/50716・IEC 62278/62279・SIL、信楽・福知山線・サンティアゴ・テンピの事故、CLC/TS 50701・IEC 63452 と Radio-Stop 事件、自動車の機能安全との対比

## 今後のテーマ候補

- 踏切と障害物検知 (level crossings & obstacle detection) — 踏切制御の論理、障害物検知装置、無線踏切。鉄道で知覚の問題が残る場所
- 鉄道の自動運転 (GoA & ATO) — GoA 2〜4、ETCS 上の ATO、無人運転の地下鉄
- 列車ダイヤと運行管理 (timetabling & traffic management) — 運転間隔の計算、遅延の伝播、運行管理システム
- 高速鉄道の車両と軌道 (high-speed rolling stock & track) — 新幹線の車両・軌道・地震時の早期検知と停止
