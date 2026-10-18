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
- `llm-application-security.md` — LLM アプリケーションのセキュリティ（命令とデータが
  同じ経路を流れる層）。直接・間接プロンプトインジェクションと知識汚染・ツールポイズニングの
  分類、Markdown 画像などの情報流出の経路、EchoLeak と GitHub MCP の事例、防御の 5 層
  （モデル・入力の区別・検出・設計・実行環境）と適応的な攻撃による検出の突破、CaMeL と
  設計パターン、Rule of Two、ジェイルブレイクとの区別、MCP に固有の攻撃面。
  [`../ai/`](../ai/README.md) のエージェント・RAG 文書のセキュリティ考察を束ねる
- `perimeter-defense-zero-trust.md` — 境界防御とゼロトラスト（通信を検査し止める層）。
  パケットフィルタからステートフル・次世代ファイアウォールへの判断材料の拡大、プロキシと DMZ、
  IDS/IPS (シグネチャと異常検知、基準率の誤謬、回避、暗号化の影響)、WAF と仮想パッチ、
  NIST SP 800-207 のゼロトラスト (PE/PA/PEP) と BeyondCorp、SASE/SSE、
  侵入の入口になった境界の機器と TLS インスペクションの代償
- `detection-incident-response.md` — 検知とインシデント対応（突破された後の層）。
  ログの収集 (時刻・保存期間・改ざん防止)、SIEM の正規化と相関と detection as code、
  EDR/XDR/MDR とエージェントの危うさ、痛みのピラミッドと STIX/TAXII/TLP、SOC と CSIRT、
  NIST SP 800-61 Rev. 3 の CSF 2.0 にもとづくライフサイクル、RFC 3227 の揮発性の順序と
  証拠保全の連鎖、個人情報保護法・GDPR・SEC の報告期限
- `attack-lifecycle.md` — 攻撃のライフサイクル（攻撃者の側から見た工程）。
  サイバーキルチェーンと ATT&CK (v18)・ダイヤモンドモデル、APT と初期侵入 (脆弱性の悪用・
  インフォスティーラー・ヘルプデスクへの電話)、環境寄生と横展開、ランサムウェアの分業 (RaaS・
  二重恐喝) と支払いの動向、DDoS の 3 層 (反射と増幅・ボットネット・HTTP/2 Rapid Reset)
- `vulnerability-management.md` — 脆弱性管理（見つかった脆弱性を組織として扱う層）。
  CVE と CNA・NVD の 2026 年の方針変更・CWE、CVSS v4.0 の指標群と限界、EPSS・CISA KEV・
  SSVC による優先順位付け、協調的な開示 (ISO/IEC 29147/30111・JVN・EU CRA の報告義務)、
  修正の期限と直せないときの緩和、脆弱性スキャン・ペネトレーションテスト・レッドチーム
- `web-application-vulnerabilities.md` — サーバ側の Web アプリケーション脆弱性（データが命令に
  なる層と、誰の要求かを確かめない層）。インジェクションの定式化と対策 (SQL・OS コマンド・
  テンプレートと式言語・XXE・デシリアライズ)、パストラバーサルとアップロード、IDOR/BOLA と
  認可の設計、SSRF と Capital One、セッション管理、リクエストスマグリング、競合状態、
  セキュアコーディングの原則と OWASP Top 10:2025

読む順の目安: 見つける (`fuzzing-program-analysis.md`) → exploit させない
(`memory-safety-exploit-mitigations.md`) → 閉じ込める (`sandbox-architectures.md`)
→ 出所を検証する (`software-supply-chain-security.md`)。四層で「単一の失敗で
崩れない防御」を構成する。

なお、ブラウザ側のウェブセキュリティ（同一生成元ポリシー・XSS/CSRF・
COOP/COEP・権限モデル）は [`../browsers/web-security-model.md`](../browsers/web-security-model.md)
がプラットフォーム文脈ごと扱う。
