# Security

ソフトウェアセキュリティ (software security) の分析。メモリ安全性・exploit 緩和・
サンドボックス・応用暗号・サプライチェーンといった、「攻撃者を前提に壊れ方を設計する」
機構を横断的に扱う。`hardware/hardware-security/`（サイドチャネル・TEE・信頼の起点）の
**ソフトウェア側の対**であり、シリコンが守る底の上に積み上がるソフトウェア防御の層を
担当する。

同時にこの領域は**横串 (cross-cutting)** である。各分析文書が持つ「セキュリティ考察」
セクション——`toolchain/` の ASLR/RELRO と xz 事件、`ios/`・`android/` のプラットフォーム
緩和、`ebpf/` の検証器と Spectre、`compilers-runtimes/` の JIT 型混同、`networking/` の
TLS——は、それぞれの文脈で断片的に現れる。本領域はそれらを機構ごとに束ね直し、
「攻撃と防御の軍拡競争」という一本の系譜として一般化する場である。

## 構成

- `memory-safety-exploit-mitigations.md` — メモリ安全性と exploit 緩和の系譜。
  C/C++ に構造的に生じるメモリ破壊（スタック/ヒープオーバーフロー・use-after-free・
  型混同・整数オーバーフロー）と未定義動作 (UB) の関係、code injection →
  DEP/canary → ROP/JOP → ASLR → CFI → shadow stack (CET) → PAC/BTI → MTE →
  CHERI という軍拡競争、AddressSanitizer とファジングの検出系、Rust の所有権・
  マネージドランタイム (GC) による根絶と CISA/ONCD の政策転換・Android の Rust 実測、
  iOS/Android への積層。攻撃と防御を必ず対で扱う防御的分析

## 今後のテーマ候補

- サンドボックス設計比較 (sandbox architectures) — seccomp / namespaces / WASM /
  gVisor を「隔離境界をどこに引くか」で比較。`../ebpf/`（seccomp-BPF・検証器）と
  `../virtualization/`（namespaces/cgroups・gVisor・WASM サンドボックス）の
  セキュリティ議論を束ね直す横串
- 応用暗号工学 (applied cryptographic engineering) — 鍵交換・AEAD・署名・
  前方秘匿性・PKI の実装上の落とし穴（サイドチャネル・nonce 再利用・実装脆弱性）。
  `../networking/tls-internals.md` の TLS 各論を暗号プリミティブの一般論へ引き上げる
- サプライチェーンセキュリティ (software supply chain security) — 依存関係の注入・
  reproducible builds・SBOM・署名検証・SLSA。`../toolchain/`（xz/liblzma 事件と
  ロード時実行点）を「コード内の欠陥」ではなく「コードの出所」の問題として展開する
- ファジングとプログラム解析 (fuzzing & program analysis) — カバレッジ誘導ファジング・
  シンボリック実行・静的解析・サニタイザの内部機構。本領域の検出系を深掘りし、
  `../compilers-runtimes/` のコンパイラ解析基盤と接続する
- ウェブ・アプリケーションセキュリティ (web & application security) — 同一生成元
  ポリシー・XSS/CSRF・サイト分離・権限モデル。ブラウザ内部（`../` の今後の候補）と
  `../networking/` の HTTP 各論の合流点
