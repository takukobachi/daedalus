# Observability

可観測性 (observability) と性能工学 (performance engineering) の分析。「システムが今なにをしているかを、どうやって・いくらのコストで知るか」を扱う。[`../security/`](../security/README.md) に続く**第二の横串**である。各分析ドキュメントが末尾に持つ「観測の入口」表（biolatency・SMART・Perfetto・ethtool）を、観測の単価階層（カウンタ→サンプリング→イベント記録）と「症状を下層の事実に接地させる」診断の縦糸として一般化する場である。

## 構成

- `observability-performance-engineering.md` — 可観測性と性能工学：観測の単価階層と
  予算配分、PMU とトップダウン解析 (Yasin)、カーネルトレーシング (tracepoint/kprobe・
  ftrace/perf・eBPF のカーネル内集計)、オフ CPU 解析と PSI、分散トレーシング
  (Dapper・文脈伝搬・head/tail サンプリング)、メトリクス工学 (USE/RED・ヒストグラム・
  カーディナリティ)、SLO/エラーバジェット、測定方法論の落とし穴 (coordinated omission・
  パーセンタイル合成・tail at scale)
- `logging-systems.md` — ログシステム：構造化ログと OpenTelemetry Logs のデータモデル
  (trace_id による相関)、収集パイプライン (Fluent Bit/Vector/OTel Collector のバッファ・
  背圧・at-least-once と重複・ローテーションでの取りこぼし)、索引の前払いと走査の後払い
  (全文の転置インデックス・Loki のラベル索引と圧縮チャンク・ClickHouse の列指向)、
  ラベルの濃度、保持の階層とサンプリング、秘密の混入とログの注入、Log4Shell、
  監査ログの改ざん耐性 (ハッシュ連鎖・WORM)

## 今後のテーマ候補

- 時系列データベース (time-series databases) — Gorilla 圧縮、ダウンサンプリング、
  逆引きインデックス。`../databases/` のストレージエンジン議論の特化形
- ベンチマーキング方法論 (benchmarking methodology) — 統計的有意性、外れ値と分散、
  マイクロベンチの罠 (JMH)、再現性。本書の測定方法論の体系化
- ランタイム観測の各論 (runtime observability) — JFR、V8 inspector、Go pprof/trace。
  `../compilers-runtimes/`・`../browsers/` の観測面との接続
