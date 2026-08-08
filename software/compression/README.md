# データ圧縮とコーデック (Compression & Codecs)

データを縮める機構の分析。可逆圧縮（gzip/zstd/lz4 — バイト列を完全に復元する）と
非可逆圧縮（JPEG/H.264/AV1/Opus — 知覚品質と引き換えにビットを捨てる）は別技術に
見えるが、どちらも**確率モデル＋エントロピー符号**という同じ骨格の上に立ち、
その理論的下限は情報理論
([`../../foundations/information-theory/`](../../foundations/information-theory/))
が与える。ストレージエンジンのページ圧縮、時系列メトリクスの差分符号化、CDN の
転送量、動画配信のビットレートラダー——既存の分析文書が暗黙に依存してきた
「縮める層」を、ここで明示的に文書化する。

## 文書一覧 (Documents)

1. [`lossless-compression.md`](lossless-compression.md) —
   可逆圧縮。「確率モデル＋エントロピー符号」への分解、LZ77 と辞書法の普遍性、
   Huffman → 算術符号 → ANS の系譜、DEFLATE/zstd/brotli/lz4 の設計対比、
   列指向・時系列の専用変換 (delta, Gorilla XOR, bit-packing)、
   圧縮とセキュリティ (CRIME/BREACH, zip bomb)
2. [`media-codecs.md`](media-codecs.md) —
   映像・音声コーデック。予測→変換→量子化→エントロピー符号の共通スケルトン、
   DCT とエネルギー集中、動き補償とフレーム間予測、レート制御、
   JPEG → H.264 → HEVC → AV1 の世代進化、音響心理モデルと Opus、
   ハードウェアコーデック、コーデックパーサの攻撃面 (Stagefright)

## 今後のテーマ候補

- 画像の次世代 (AVIF/JPEG XL/WebP) — 静止画をめぐる動画イントラ符号化系と
  専用設計系の競合、後方互換の経済
- ニューラル圧縮 (neural compression) — 学習ベースのエントロピーモデル・
  autoencoder 型コーデック。「圧縮＝予測」
  ([`../../foundations/information-theory/entropy-source-coding.md`](../../foundations/information-theory/entropy-source-coding.md) §5)
  の直接の実装で、`../ml-systems/` との合流点
- ストレージスタックの透過圧縮 — ZFS/Btrfs/ZNS の圧縮、NVMe 計算ストレージ、
  圧縮とアラインメント・torn write の相互作用。`../databases/`・
  `../../hardware/storage-devices/` との合流点
