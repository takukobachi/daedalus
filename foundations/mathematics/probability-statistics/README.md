# 確率・統計 (Probability & Statistics)

不確実性を定量的に扱う数学。分岐予測のヒット率、製造歩留まり、故障率、
テイルレイテンシ——Daedalus の分析に現れる「率」「分布」「期待値」という
言葉の正確な意味を、確率空間の公理から積み上げて文書化する。

## 文書一覧 (Documents)

1. [`probability-random-variables.md`](probability-random-variables.md) —
   確率空間の公理、条件付き確率と独立、確率変数、期待値・分散、主要分布
   (ベルヌーイ・二項・幾何・ポアソン・指数・正規)、大数の法則と中心極限定理。
   シリーズの起点
2. [`markov-chains-queueing.md`](markov-chains-queueing.md) —
   マルコフ性と遷移行列(線形代数第2回の確率行列と合流)、定常分布、
   ポアソン過程、M/M/1 と利用率–レイテンシの双曲線、リトルの法則。
   分岐予測器・キャッシュ・サーバ性能解析の言語

## ロードマップ (Roadmap)

番号は推奨する読み書きの順序(依存順)。

3. 推定と仮説検定 (estimation & hypothesis testing) — 最尤推定、信頼区間、
   ベイズ推定。測定・ベンチマーク・A/B テストを「何回測れば足りるか」から
   設計するための道具
4. 信頼性理論 (reliability theory) — 故障率、バスタブ曲線、MTBF/MTTF、
   直列・並列系の信頼度。機能安全 (ISO 26262) の確率的メトリクスの土台
