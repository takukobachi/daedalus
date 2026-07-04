# eBPF

eBPF (extended Berkeley Packet Filter) の分析。カーネルを再コンパイルせず、
カーネルモジュールも書かずに、検証済みバイトコードをカーネル内で安全に実行する
「カーネル内の安全な実行基盤 (in-kernel execution platform)」を扱う。
`operating-systems/` が扱うカーネル機構そのものと、`networking/`（XDP・tc による
高速データプレーン）やセキュリティ（LSM/BPF, seccomp）といった応用面を
つなぐ層であり、「カーネルの拡張点はどこにあるべきか」という設計問題への
Linux の回答である。

## 構成

- `ebpf-kernel-programmability.md` — eBPF：カーネル内プログラマビリティの基盤。
  カーネルモジュール vs ユーザ空間の永遠のトレードオフと第三の道、classic BPF
  (1993) からの系譜、検証器 (verifier) の静的解析と「検証器と戦う」開発体験、
  JIT と実行時ハードニング、プログラムタイプとフック分類 (XDP/tc/kprobe/LSM/
  sched_ext)、マップとデータプレーン、BTF/CO-RE による可搬性、エコシステム比較

## 今後のテーマ候補

- XDP と高速データプレーン深掘り — Katran / Cilium の実装分析、DPDK・カーネル
  バイパスとの対比、`networking/` のトランスポート層との合流点
- sched_ext によるスケジューラの eBPF 化 — `../operating-systems/process-thread-scheduling.md`
  の CFS/EEVDF を「差し替え可能なポリシー」として読み直す
- Windows eBPF と移植性 — eBPF ISA の標準化 (IETF)、PREVAIL 検証器、
  「Linux 固有機構」から「汎用ランタイム仕様」への転換の行方
