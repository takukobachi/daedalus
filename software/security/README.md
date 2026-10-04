# Security

ソフトウェアセキュリティ (software security) の分析。メモリ安全性・exploit 緩和・
サンドボックス・応用暗号・サプライチェーンといった、「攻撃者を前提に壊れ方を設計する」
機構を横断的に扱う。`hardware/security/`（サイドチャネル・TEE・信頼の起点）の
**ソフトウェア側の対**であり、シリコンが守る底の上に積み上がるソフトウェア防御の層を
担当する。

同時にこの領域は**横串 (cross-cutting)** である。各分析文書の「セキュリティ考察」
セクションには、`compilers-runtimes/` の ASLR/RELRO と xz 事件、`ios/`・`android/` のプラットフォーム
緩和、`operating-systems/` の eBPF 検証器と Spectre、`compilers-runtimes/` の JIT 型混同、`networking/` の
TLS が、それぞれの文脈で断片的に現れる。本領域はそれらを機構ごとに束ね直し、
「攻撃と防御の軍拡競争」という一本の系譜として一般化する場である。

## 構成

- `memory-safety-exploit-mitigations.md` — メモリ安全性と exploit 緩和の系譜。
  C/C++ に構造的に生じるメモリ破壊（スタック/ヒープオーバーフロー・use-after-free・
  型混同・整数オーバーフロー）と未定義動作 (UB) の関係、code injection →
  DEP/canary → ROP/JOP → ASLR → CFI → shadow stack (CET) → PAC/BTI → MTE →
  CHERI という軍拡競争、AddressSanitizer とファジングの検出系、Rust の所有権・
  マネージドランタイム (GC) による根絶と CISA/ONCD の政策転換・Android の Rust 実測、
  iOS/Android への積層。攻撃と防御を必ず対で扱う防御的分析
- `fuzzing-program-analysis.md` — ファジングとプログラム解析（見つける層）。
  カバレッジ誘導ファジングの進化的ループ (AFL) とサニタイザの内部機構
  (ASan/UBSan/TSan)、シンボリック実行と concolic、静的解析と抽象解釈
  （WCET 解析と同根）、Rice の定理が強いる健全性/完全性の取引、
  OSS-Fuzz が示した「検出はインフラ」という運用論
- `sandbox-architectures.md` — サンドボックス設計比較（閉じ込める層）。
  「隔離境界をどこに引くか」の一軸で WASM (SFI) / seccomp + namespaces /
  gVisor / microVM を TCB・攻撃面・脱出の意味・コストの4軸比較、
  Chrome の積層とサイト分離（Spectre 後の境界設計）、検証器ベース隔離
  (eBPF/WASM) の賭け
- `software-supply-chain-security.md` — サプライチェーンセキュリティ（出所を検証する層）。
  パイプライン上の攻撃分類（xz・SolarWinds・依存関係かく乱）、lockfile・
  再現可能ビルド・SLSA 来歴・Sigstore・SBOM を「答えられる問い」で対応づけ、
  理論的な底として Trusting Trust と多様二重コンパイル
- `applied-cryptography.md` — 応用暗号工学（暗号を正しく使う層）。
  AEAD (AES-GCM / ChaCha20-Poly1305 / AES-GCM-SIV)・HKDF・Argon2/scrypt・
  Ed25519/ECDSA (RFC 6979)・X25519 の選択指針、nonce 再利用 (PS3・GCM)・
  タイミングサイドチャネル (Lucky13・Minerva)・パディングオラクル
  (Bleichenbacher・Vaudenay・ROBOT)・無効曲線攻撃・乱数の失敗 (Debian・
  Mining Your Ps and Qs) という実装の落とし穴、誤用耐性 API (libsodium・Tink)、
  鍵管理とローテーション、PKI/CT、耐量子移行 (FIPS 203/204/205・
  X25519MLKEM768・暗号の俊敏性)。[`../networking/tls-internals.md`](../networking/tls-internals.md)
  の TLS 各論を暗号プリミティブの一般論へ引き上げる

読む順の目安: 見つける (`fuzzing-program-analysis.md`) → exploit させない
(`memory-safety-exploit-mitigations.md`) → 閉じ込める (`sandbox-architectures.md`)
→ 出所を検証する (`software-supply-chain-security.md`)。四層で「単一の失敗で
崩れない防御」を構成する。

なお、ウェブ・アプリケーションセキュリティ（同一生成元ポリシー・XSS/CSRF・
COOP/COEP・権限モデル）は [`../browsers/web-security-model.md`](../browsers/web-security-model.md)
がプラットフォーム文脈ごと扱う。
