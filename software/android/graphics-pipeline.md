# グラフィックスパイプライン (Android Graphics: SurfaceFlinger, BufferQueue & Frame Pacing)

## 概要 (Overview)

タップから画素までの経路——**アプリの描画 (Choreographer / RenderThread) → バッファの受け渡し (BufferQueue / gralloc) → 合成 (SurfaceFlinger / Hardware Composer) → 表示 (VSYNC)**——を通しで追う。Android のグラフィックスは「GPU で速く描く」話である以上に、**プロセス境界をまたぐ生産者・消費者パイプラインの設計**であり、Binder（制御路）と fd/共有バッファ（データ路)の分業、VSYNC を基準にした全体の位相同期、フレーム落ち（ジャンク）を防ぐパイプライン深度の管理が主題になる。

本ドキュメントが答える問い: なぜ描画は「アプリが画面に描く」のではなく「アプリがバッファを提出し、別プロセスが合成する」構造なのか。16.6ms（60Hz）の予算はどの区間に配分されるのか。トリプルバッファリングは何を買い、何のレイテンシを払うのか。分析深度は Level 3。

## 方法論 (Methodology)

- 1 フレームのライフサイクル（VSYNC 受信 → UI スレッド → RenderThread → キュー → 合成 → 表示）を軸に、各段の担当プロセス・スレッドと受け渡される資源（バッファ・フェンス・トランザクション）を特定する。
- 「制御はどこを通るか（Binder）／画素データはどこを通るか（共有バッファ、コピーなし）」を常に区別する——[`binder-internals.md`](binder-internals.md) の「大きなデータは fd をハンドルとして渡す」の最大の適用例として読む。
- 一次資料は AOSP のグラフィックス公式ドキュメント (source.android.com/docs/core/graphics) に置き、バージョン依存の機構（BLAST 等）は導入版を付記する。
- GPU 内部（シェーダ実行・タイルレンダリング）は本書の対象外とし、GPU を「コマンドを投げるとフェンス付きでバッファを埋める装置」として扱う。

## 発見事項 (Findings)

1. **画素データは一度もコピーされない。** アプリの GPU が描いたバッファ（gralloc 確保・dma-buf fd）は、ハンドルが SurfaceFlinger へ渡るだけで、合成もそのメモリを直接読む。Binder で運ばれるのは**バッファの所有権とメタデータ**であり、[`binder-internals.md`](binder-internals.md) の「1MB 制限、大物は fd で」の帰結が、そのままパイプラインの骨格になっている。
2. **同期の実体は「フェンス」である。** バッファの受け渡しは GPU がまだ描き終わる前に行われ（非同期）、「いつ読んでよいか」は同期フェンス (sync fence, dma-fence の fd) が運ぶ。CPU 側のパイプラインと GPU 側のパイプラインを疎結合に保つ要であり、フェンス待ちの位置がそのままレイテンシ地図になる。
3. **BufferQueue は背圧付きのリングである。** 生産者 (アプリ) は空きバッファを dequeue できなければブロックする——描画が消費 (合成) を追い越せない構造的な流量制御で、oneway Binder の背圧と同じ文法。バッファ数（2〜3）がパイプライン深度＝レイテンシとスループットのトレードオフを決める。
4. **合成は「GPU に戻す」か「ディスプレイに任せる」かの動的な最適化である。** Hardware Composer (HWC) はレイヤをディスプレイのオーバーレイプレーンに直接割り当て（GPU を起こさず合成）、収まらない分だけ SurfaceFlinger が GPU (client composition) で畳む。**電力最適化としての合成戦略**が毎フレーム決定されている。
5. **全員が同じ VSYNC を見ているが、同じ位相では動いていない。** アプリの描画開始・SurfaceFlinger の合成開始は、ハードウェア VSYNC からの**位相オフセット**でずらされ、「アプリが描き終わった直後に合成が始まり、合成が終わった直後に走査が始まる」よう予算が区画割りされる。DispSync/VSYNC モデルはソフトウェアで位相を合成する PLL である。
6. **BLAST (Android 12 頃〜) はバッファ提出をトランザクションに載せ替えた。** BufferQueue の暗黙のキューではなく、`SurfaceControl.Transaction` に「このバッファを・このジオメトリで・この仲間と同時に」を明示して原子的に適用する。ウィンドウリサイズ時の「中身と枠がずれる」古典的な非同期アーティファクトを、**複数プロセスの状態変更の原子コミット**として解いた。
7. **リフレッシュレートは変数になった。** 可変レート (48/60/90/120Hz) 時代の SurfaceFlinger はアプリのフレーム提出ヒント・コンテンツ種別からレートを投票制で選ぶ。フレームペーシング（Swappy / Frame Pacing ライブラリ）は「速く描けたフレームを早く出しすぎない」制御であり、**速さではなく等間隔性がなめらかさ**であることを機構化している。

## アーキテクチャ (Architecture)

### 全体像 — 二つの経路（制御と画素）

```
 [アプリプロセス]                          [system_server]        [surfaceflinger]
 UI スレッド ── measure/layout/record ─┐    WindowManager ◀──── ウィンドウ状態 (Binder)
 RenderThread ── Skia/HWUI → GL/Vulkan ─┤
      │ gralloc バッファに GPU 描画      │
      ▼                                 ▼
 BufferQueue/BLAST: バッファ + フェンス + Transaction ──(Binder: ハンドルのみ)──▶ 合成へ
                                                                    │
                                                     HWC HAL (ベンダ): レイヤ割当
                                                       ├─ オーバーレイ (DPU 直結)
                                                       └─ あふれ分 → GPU 合成
                                                                    ▼
                                                          ディスプレイ走査 (VSYNC)
```

- **描画と合成は別プロセス**: アプリがクラッシュ・停止しても画面全体は SurfaceFlinger が守る。逆に SurfaceFlinger は「全アプリのバッファを見られる」特権プロセスであり、隔離の要でもある。
- **WindowManager (system_server) は制御面のみ**: どの Surface がどの Z 順・領域にあるかを決め、SurfaceControl として surfaceflinger に指示する。画素は一切通らない。

### 1 フレームの予算配分（60Hz・位相オフセットの考え方）

```
 HW VSYNC ─┬────────────── 16.6ms ──────────────┬─▶
           │ +app offset                         │
           │   Choreographer#doFrame:            │
           │   input → animation → traversal     │
           │   RenderThread: record→GPU submit   │
           │        +sf offset                   │
           │          SurfaceFlinger: latch      │
           │          (acquire fence 待ち) → 合成 │
           │                              走査 ──┘ 表示は 1〜2 VSYNC 後
```

- アプリ描画・合成が各々 1 VSYNC 期に収まる標準構成では、**タッチから表示まで 2〜3 フレーム**のパイプラインレイテンシが構造的に存在する。オフセットを詰めればレイテンシは減るが、締切に間に合わないリスク（ジャンク）が増える——位相設計そのものがレイテンシ/安定性のトレードオフ。

## 技術詳細 (Technical Details)

### 1. アプリ側 — Choreographer と RenderThread

- **Choreographer** は VSYNC イベント（surfaceflinger からアプリへ配られる）を受け、フレームコールバックを **input → animation → traversal (measure/layout/draw)** の固定順で実行する。「VSYNC に揃えて 1 フレーム分の仕事を 1 回だけやる」ための基準クロック配布機構。
- **描画の記録と実行の分離**: UI スレッドは View 階層を**表示リスト (display list) に記録するだけ**で、GPU コマンド発行は **RenderThread** が行う（HWUI、実体は Skia。バックエンドは GL または Vulkan、近年は Vulkan が既定方向、GL は ANGLE で Vulkan 上に実装する方向）。UI スレッドのブロック（＝入力応答性）と GPU 待ちを分離するための二段パイプライン。
- **アニメーションの委譲**: リング進捗のような定型アニメーションは RenderThread 単独で進められ（Animated Vector 等）、UI スレッドがジャンクしても止まらない。

### 2. バッファ管理 — gralloc・BufferQueue・フェンス

- **gralloc HAL** がバッファを確保する。フォーマット・用途フラグ（GPU 書込/合成読出/カメラ書込…）から**ベンダ固有の最適レイアウト**（タイル・圧縮 AFBC 等）を選び、dma-buf fd として各プロセスへ渡す。「用途の集合で確保する」API 設計が、後続の全段（GPU・DPU・カメラ ISP）でのゼロコピー共有を可能にしている。
- **BufferQueue の状態機械**: 各バッファは FREE → DEQUEUED（アプリが取得）→ QUEUED（提出）→ ACQUIRED（合成中）→ FREE を巡る。dequeue できなければ生産者はブロック——これが**背圧**。ダブルバッファは低レイテンシだが締切ミスに弱く、トリプルバッファは 1 フレームのレイテンシで安定性を買う。
- **同期フェンス**: 「GPU がこのバッファを描き終えた」(acquire fence)・「合成側がこのバッファを読み終えた」(release fence) を dma-fence の fd で運ぶ。CPU はフェンスを**待たずに**ハンドルだけ先に渡せるため、CPU/GPU/DPU の三者が最大限並行に走る。フェンス待ちの計測（どの段で締切を割ったか）が Systrace/Perfetto のフレーム分析の中身である。
- **BLAST**: バッファを `SurfaceControl.Transaction` に添付し、ジオメトリ変更・親子関係・他ウィンドウの変更と**原子的に**コミットする (Android 12 頃に BufferQueue から移行)。複数プロセス（アプリ＋IME＋システム UI）にまたがる画面遷移の同期が、トランザクションのマージとして表現できるようになった。

### 3. 合成 — SurfaceFlinger と HWC

- SurfaceFlinger は VSYNC (+sf offset) で起き、各レイヤの**最新の到着済みバッファを latch** し、可視レイヤのリストを HWC HAL に提示する。
- **HWC の割当判断**: ディスプレイコントローラ (DPU) のオーバーレイプレーン数・スケーラ・回転能力の範囲でレイヤを直接割当（`DEVICE` 合成）し、収まらないレイヤ群は SurfaceFlinger が GPU で 1 枚に畳む（`CLIENT` 合成）。典型画面（壁紙＋アプリ＋ステータスバー＋ナビバー）が DPU だけで合成できるよう、プレーン数はハードウェア設計段階で決められている——**電力の議論がハードウェア構成に遡る**好例。
- **仮想ディスプレイ・画面録画**も同じ機構の別出口（合成結果をバッファへ書く）であり、外部ディスプレイ・Cast は「もう一つの消費者」として自然に増設される。
- **VSYNC モデル**: ハードウェア VSYNC は較正時だけ有効化し、通常はソフトウェアモデル (DispSync) が位相・周期を推定してアプリ/合成用のオフセット付き VSYNC を配る——割り込み削減（電力）と位相制御の両立。

### 4. 可変リフレッシュレートとフレームペーシング

- SurfaceFlinger はレイヤごとの**希望フレームレート**（アプリの `setFrameRate` ヒント、動画のネイティブ fps、操作中か否か）を集めて表示モードを投票で決める。60fps 動画を 120Hz パネルで再生する類の**整数倍関係の管理**が中心課題。
- **フレームペーシング**: 描画が速く終わったフレームを即提出すると、次の重いフレームとの表示間隔が不均一になりカクつく。Swappy (Android Game SDK) はフレームの**提出タイミングを意図的に遅らせて**表示間隔を等しくする。「レイテンシを少し払って等間隔性を買う」——スループット・レイテンシ・ジッタの三者では**ジッタが知覚品質を支配する**というディスプレイ系の経験則の機構化。
- **ジャンクの分類**: 締切ミスは (a) UI スレッド起因（layout・GC・Binder 同期呼び出しの混入）、(b) RenderThread/GPU 起因、(c) 合成側起因に分けられ、Perfetto のフレームタイムラインが各フレームの実際の律速段を記録する。[`art-runtime-internals.md`](art-runtime-internals.md) の GC ポーズ設計（サブミリ秒）は、この予算の (a) を守るためにある。

## セキュリティ考察 (Security Considerations)

- **GPU ドライバはアプリから直接届くカーネル攻撃面**: あらゆるアプリが GPU を叩けるため、ベンダ GPU ドライバ（Mali・Adreno）の脆弱性は Binder と並ぶ権限昇格の常連であり、実際に in-the-wild 悪用が繰り返し報告されている。ドライバの ioctl 面の広さ・メモリ管理の複雑さ（マッピングの寿命）が原因で、[`binder-internals.md`](binder-internals.md) の「中核機構＝攻撃面の中核」と同じ構図。GPU ドライバの一部をユーザ空間へ出す・Rust 化するといった構造的対策が進行中の領域である。
- **dma-buf ハンドルは権限である**: gralloc バッファの fd を渡すことは画素メモリへのアクセス権の委譲であり、誤って強力なバッファ（他アプリの画面内容）を渡す経路はサンドボックス迂回になる。SurfaceFlinger が「全画面を見られる」単一プロセスである以上、そこへの到達性 (SELinux) と、スクリーンショット・録画 API の権限ゲートが実質的な壁になる。
- **保護コンテンツ (DRM)**: `SECURE` フラグ付きバッファは GPU/CPU から読めないメモリ（TEE 管理）に置かれ、HWC が直接合成する。スクリーンショットには黒く写る——「OS 自身にも見せない」合成経路が、通常経路と並走している（Widevine L1 の要件）。
- **画面からの情報漏洩はバッファ以外からも**: 描画時間の差・GPU の共有状態を測るサイドチャネル、オーバーレイを使った UI 偽装 (tapjacking) など、パイプラインの性質そのものが攻撃面になる系譜がある。オーバーレイ表示中のタッチ制限・`FLAG_SECURE` はその対症である。

## 結論 (Conclusions)

- Android グラフィックスの本質は「プロセス境界をまたぐゼロコピー生産者・消費者系」であり、制御（Binder トランザクション）と画素（dma-buf ハンドル＋フェンス）の分離、BufferQueue の背圧、VSYNC 位相による予算区画が骨格をなす。個々の機構は Binder・mmap 共有・cgroup と同じ「共有はハンドルで、同期は明示で」という Android 全体の設計文法の適用である。
- 合成の HWC/GPU 動的振り分けと VSYNC のソフトウェアモデルは、**電力が第一級の設計変数**であることの現れで、プレーン数というハードウェア構成にまで遡って最適化されている。
- BLAST・可変リフレッシュレート・フレームペーシングという近年の変化は、いずれも「非同期パイプラインの見えないずれ（ジオメトリ・表示間隔）を明示的な同期点に変える」方向であり、なめらかさの正体が等間隔性＝ジッタ制御であることを機構として認めた設計である。
- SurfaceFlinger の特権性・GPU ドライバの到達性・バッファハンドルの委譲は、そのままセキュリティ境界の地図である。グラフィックスは性能の話であると同時に、Android の隔離モデルの主要な検証対象である。

### 次の接続先 (Related Topics)

- [`platform-architecture.md`](platform-architecture.md) — SurfaceFlinger・system_server を含む全体像。「1 タップの旅」の描画区間の詳細が本書。
- [`binder-internals.md`](binder-internals.md) — 制御路の実体。fd 翻訳・oneway 背圧・1MB 制限が本書のバッファ受け渡し設計を規定する。
- [`art-runtime-internals.md`](art-runtime-internals.md) — フレーム予算の中で GC が守るべき制約の出どころ。
- [`../../hardware/hw-sw-interface/mmio-dma-interrupts.md`](../../hardware/hw-sw-interface/mmio-dma-interrupts.md) — dma-buf・フェンス・DPU の下にある DMA と割り込みの一般論。
- [`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) — RenderThread・surfaceflinger の優先度 (RT/cgroup) 設計の基盤。

## 参考文献 (References)

**一次資料 (Primary — AOSP)**

- AOSP, *Graphics architecture*（BufferQueue, SurfaceFlinger, HWC, VSYNC, Fence の公式体系）. — [source.android.com/docs/core/graphics](https://source.android.com/docs/core/graphics) — Accessed: 2026-07-03。
- AOSP, *Hardware Composer HAL* ／ *Implementing gralloc*. — [source.android.com/docs/core/graphics/implement-hwc](https://source.android.com/docs/core/graphics/implement-hwc) — Accessed: 2026-07-03。
- Android Developers, *Frame pacing library (Swappy)* ／ *setFrameRate*. — [developer.android.com/games/sdk/frame-pacing](https://developer.android.com/games/sdk/frame-pacing) — Accessed: 2026-07-03。可変レート時代のペーシングの規範。
- Perfetto, *Frame timeline / jank classification*（フレーム律速段の計測法）. — [perfetto.dev/docs](https://perfetto.dev/docs/data-sources/frametimeline) — Accessed: 2026-07-03。

**二次資料 (Secondary)**

- Google グラフィックスチームのカンファレンス発表（"Drawn out: how Android renders" Google I/O 2018 ほか、RenderThread・BLAST の解説）. — [youtube.com/@AndroidDevelopers](https://www.youtube.com/@AndroidDevelopers) — Accessed: 2026-07-03。
- Google Project Zero / ベンダセキュリティ告示（Mali・Adreno GPU ドライバ脆弱性の in-the-wild 事例分析）. — [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/) — Accessed: 2026-07-03。
