# Toolchain

ツールチェーン (toolchain) の分析。コンパイル後の成果物（オブジェクトファイル）が
実行中のプロセスになるまで——**リンク (linking) とロード (loading)**——を扱う層。
`../../hardware/hw-sw-interface/abi-syscalls.md` が定めた ABI という「見えない契約」を
**誰が守らせているか**への答えであり、将来のコンパイラ・ランタイム分析
（IR, 最適化, GC, JIT）の土台となる。

## 構成

- `linkers-loaders-elf.md` — リンカとローダ：シンボル解決と再配置という2つの
  基本操作、ELF の二重ビュー（セクション vs セグメント）、静的リンクの規則と
  失敗モード、PLT/GOT と遅延束縛、シンボル介入と LD_PRELOAD、ld.so の
  ブートストラップ、IFUNC と xz 事件、ASLR/PIE/RELRO、LTO とリンカの進化
  (BFD ld → gold → lld → mold)、dyld / Android linker / PE との対比

## 今後のテーマ候補

- コンパイラバックエンド (compiler backend) — 命令選択 (instruction selection)、
  レジスタ割り当て (register allocation)、命令スケジューリング。
  `hardware/cpu-architecture/` の ISA・パイプライン分析との合流点
- ビルドシステムと再現可能ビルド (build systems & reproducible builds) —
  依存グラフとパッケージ依存解決 (go modules の MVS, cargo の semver, pip)、
  キャッシュと分散ビルド、ビット単位で同一な成果物が
  サプライチェーン検証に持つ意味
- デバッグ情報とアンワインド (debug info & unwinding) — DWARF、
  スタックアンワインド (CFI)、シンボリケーション。プロファイラ・例外処理・
  クラッシュレポートの共通基盤
