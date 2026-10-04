# Core Animation とレンダーサーバ (Layer Trees, the Render Server & Frame Pacing)

## 概要 (Overview)

[`../android/graphics-pipeline.md`](../android/graphics-pipeline.md) は Android の描画を「プロセス境界をまたぐゼロコピー生産者・消費者系」として読んだ。本ドキュメントはその iOS 版として、**CALayer の三つの木 (モデル/プレゼンテーション/レンダー)、CATransaction によるレイヤツリーのコミット、レンダーサーバ (backboardd 内 CARenderServer) でのサーバサイドアニメーション、IOSurface によるゼロコピーバッファ共有、CADisplayLink と ProMotion 可変レート、ヒッチ (hitch) の分類**を同じ切り口で読む。

対比の焦点は**分業の境界線**にある。Android はアプリ内 RenderThread が GPU コマンドまで発行し、SurfaceFlinger は「完成したバッファの合成」だけを担う。iOS はさらに一歩踏み込み、**アニメーションの補間・レイヤの描画までをレンダーサーバに移した**。そのため、アプリのメインスレッドが完全に固まっていても、進行中のアニメーションは滑らかに動き続ける。「アプリをどこまで信用しないか」の設計差が、パイプラインの切断位置として現れている。分析深度は Level 3。

## 方法論 (Methodology)

- Android 版と同じ 1 フレームのライフサイクル(表示リンク受信 → 変更の記録 → コミット → サーバ側処理 → 合成 → 表示)を軸に、各段の担当プロセスと受け渡される資源(レイヤツリー差分・IOSurface・フェンス相当)を特定する。
- 「制御(レイヤツリーのコミット)はどこを通り、画素 (IOSurface) はどこを通るか」を常に区別し、[`mach-ipc-xpc.md`](mach-ipc-xpc.md) の「大物は OOL/ポートで」の最大の適用例として読む。
- 一次資料は Apple Developer ドキュメントと WWDC セッション(特にヒッチ分類・ProMotion 最適化の各回)に置く。レンダーサーバ内部は非公開のため、公式説明の範囲と挙動からの推定を区別する。
- Metal によるゲーム系直接描画は CAMetalLayer の接続点までを扱い、GPU API 自体の詳説はしない。

## 発見事項 (Findings)

1. **CALayer は「描画命令」ではなく「宣言的なシーングラフのノード」である。** 位置・変形・内容・不透明度を属性として持つレイヤの木が UI の真実であり、描画とは「この木をレンダーサーバに複製し、サーバが毎フレーム評価する」ことである。Android の View が「invalidate → 再記録 → 再描画」のたびにアプリの仕事を要するのに対し、**iOS は木を送った後の変化(アニメーション)をサーバの自律動作にできる**。つまり宣言性がプロセス分界を越える通貨になっている。
2. **木は三つある。** アプリ内の**モデル木**(コードが触る値・アニメーションの終点)、**プレゼンテーション木**(いまこの瞬間の見かけの値)、そしてレンダーサーバ内の**レンダー木**(実際に評価・合成される複製)。「タップ判定は見かけの位置で行いたい」(プレゼンテーション木を読む)のような実務は、この三重構造の直接の帰結である。
3. **コミットはトランザクションであり、暗黙に束ねられる。** 1 回のランループで行った全レイヤ変更は CATransaction に自動集約され、ランループ末尾で**差分としてレンダーサーバへ一括送信**される。「同一イベント処理内の変更は必ず同一フレームに乗る」という原子性を、アプリに意識させず与える設計である。Android が BLAST トランザクション([`../android/graphics-pipeline.md`](../android/graphics-pipeline.md))で**後付けした**原子コミットを、iOS は初期設計 (2007) から持っていた。
4. **アニメーションは「終点と曲線」だけがサーバへ渡り、補間はサーバが行う。** CAAnimation は from/to/duration/タイミング関数の記述であり、毎フレームの中間値計算はレンダーサーバの仕事である。ゆえに**メインスレッドがブロックしてもアニメーションは止まらない**。これは Android で「UI スレッドが固まるとアニメも止まる」(RenderThread 委譲された一部を除く)のと対照的で、体感品質の既定値を機構が保証している。代償は、サーバが解釈できる語彙(プロパティアニメーション)に表現が縛られることである。毎フレームアプリが介入する UIScrollView のスクロールやゲームは、この恩恵の外にある。
5. **画素の共有は IOSurface、会計はケイパビリティ。** GPU が描くバッファは IOSurface(プロセス間共有・参照カウント付き)として確保され、Mach ポート([`mach-ipc-xpc.md`](mach-ipc-xpc.md))で受け渡される。これは gralloc バッファ+dma-buf fd の正確な対応物である。CAMetalLayer の drawable 取得はダブル/トリプルバッファのプールからの dequeue で、BufferQueue と同じ背圧(枯渇時のブロック)を持つ。**ゼロコピー・ハンドル渡し・背圧**という基本構造は両 OS で完全に一致する。
6. **CADisplayLink は Choreographer であり、ProMotion では「次の締切」が交渉制になる。** 表示リンクはリフレッシュに同期してコールバックを配り、ProMotion (10–120Hz) では `preferredFrameRateRange` とコンテンツの要求から**フレームごとに次の締切が決まる**。低速化(静止画面で 10Hz)による省電力と、タッチ時の即時 120Hz 昇格の両立は、Android の可変レート投票と同じ問題への解である。
7. **Apple は「ジャンク」を commit hitch と render hitch に公式分類した。** 締切ミスの原因を、(a) アプリ側の準備遅延(コミットがフレーム締切に間に合わない = commit hitch)と、(b) サーバ側の描画過負荷(オフスクリーンレンダリング・重いブラー等 = render hitch)に二分する。**パイプラインの切断位置(コミット境界)が、そのまま性能責任の分界線**になっている。Perfetto のフレームタイムラインが app/SF どちらの律速かを記録するのと同じ思想で、切断位置だけが違う。

## アーキテクチャ (Architecture)

### 全体像 — 切断位置の違い

```
 [アプリプロセス]                                  [backboardd (レンダーサーバ)]
 メインスレッド:                                    レンダー木 (レイヤツリーの複製)
   レイヤ属性の変更・アニメーション記述               ├─ 毎フレーム: アニメーション補間
   ランループ末尾: CATransaction commit              ├─ レイヤ評価 → GPU で合成
     └── 木の差分 + アニメーション記述 ──(Mach IPC)──▶│    (Metal によるタイルベース合成)
   CAMetalLayer / drawRect の内容は                  └─ Display へ走査出力
     IOSurface として GPU が描画 ────(ポート渡し)────▶ 合成の入力に
                                                    [SpringBoard 等の他プロセスのレンダー木も
                                                     同一サーバが合成 = SurfaceFlinger 相当]
```

- **Android との対応**: Choreographer ⇔ CADisplayLink、RenderThread の描画 ⇔ (アプリ側では) Core Graphics/Metal による IOSurface への描画、BLAST トランザクション ⇔ CATransaction、SurfaceFlinger+HWC ⇔ レンダーサーバ。**決定的な差は「アニメーション補間と多くのレイヤ描画がサーバ側にある」こと**で、iOS のアプリはより「何を表示したいか」の宣言者に近く、Android のアプリはより「画素の生産者」に近い。
- レンダーサーバはすべてのアプリ・SpringBoard のレンダー木を見る単一特権プロセスであり、SurfaceFlinger と同じ「全画面が見える」隔離上の要衝である。

### 1 フレームの予算 (120Hz なら ~8.3ms)

```
 表示リンク発火 ─┬─ アプリ: イベント処理・レイアウト・commit ──┤ commit 締切
                │                                            │
                └─ サーバ: 差分適用・補間・合成 ──────────────┤ 表示
 commit が締切を割る → commit hitch (フレームは前の内容のまま)
 合成が締切を割る   → render hitch
```

## 技術詳細 (Technical Details)

### 1. レイヤの内容はどこで描かれるか — 三つの経路

- **(a) 属性だけのレイヤ**(色・角丸・影・変形): 内容の実体はなく、レンダーサーバが直接評価・描画する。最も軽く、アニメーションも完全にサーバ内で完結する。
- **(b) drawRect / Core Graphics**: アプリの CPU がバッキングストア(実体は IOSurface)にラスタライズし、レイヤの contents として送る。Android の Canvas ソフトウェア描画に対応するが、iOS では「静的な内容を一度描いて、あとは属性アニメーションで動かす」使い方が規範で、毎フレーム再描画は例外である。
- **(c) CAMetalLayer**: アプリの GPU コマンドが drawable (IOSurface) に直接描く。ゲーム・カメラ・地図の経路で、Android の SurfaceView + Vulkan/GL に正確に対応する。ここではアプリが画素の生産者となり、サーバは合成のみを担う。**両 OS の分業がこの経路では一致する**。
- SwiftUI はこの機構の上の宣言層であり、ビュー差分の評価後は同じレイヤ木・同じコミットに落ちる。そのため描画基盤の議論は SwiftUI 時代でも不変である。

### 2. コミットの中身 — 差分・原子性・入れ子

- commit は「変更されたレイヤの属性差分＋追加/削除＋アニメーション記述」のエンコードで、木全体の再送ではない。明示的 `CATransaction` で複数の変更のアニメーション時間・完了ハンドラを束ねられ、入れ子も可能。
- **フラッシュのタイミングがランループ結合**である点が重要で、メインスレッドの長い処理はコミット自体を遅らせる。これが commit hitch の主因である。Android で「UI スレッドを塞ぐと traversal が遅れる」のと同型だが、iOS は描画(サーバ)が止まらない分、「**アニメは動くのに操作に反応しない**」という特有の壊れ方をする。
- 複数プロセスの協調(アプリ+キーボード+システム UI の同時遷移)は、レンダーサーバが各クライアントの fence/コミットを揃えて適用することで実現される。BLAST が解いた「複数プロセスの原子的画面遷移」と同じ問題が、サーバ集中型では自然に解ける。

### 3. サーバサイドアニメーションの境界

- サーバが補間できるのは**プロパティの時間関数**(位置・変形・不透明度・角丸等、および spring 等のタイミング)であり、レイアウト再計算・テキスト再組版・ビューの生成は含まれない。ゆえに「画面遷移の開始と終点を宣言してサーバに任せる」UIKit の遷移は滑らかだが、**毎フレームアプリが値を決めたい**場合(インタラクティブ遷移・物理シミュレーション)は CADisplayLink でアプリ側駆動に切り替える。恩恵と統制の交換が API 設計に明示されている。
- スクロールは指に追随する連続的なアプリ側駆動(UIScrollView がメインスレッドでオフセット更新)だが、減速アニメーションは慣性曲線としてサーバに委譲できる形になっており、「**指が触れている間はアプリ、離れたらサーバ**」という分担が体感の頑健性を作る。

### 4. ProMotion・省電力・熱

- 可変レートの決定入力は、進行中アニメーションの要求・CADisplayLink の希望範囲・タッチ状態・熱状態で、静止時は 10Hz まで下げる。これは「等間隔性が知覚品質」([`../android/graphics-pipeline.md`](../android/graphics-pipeline.md))に「**静止時は描かないことが最大の省電力**」を加えた二本柱である。
- レート遷移時の「間隔の乱れ」を避けるため、整数比の隣接レート間で遷移する等の設計が公式に文書化されており、フレームペーシング(Swappy)の問題意識と完全に共通する。
- レンダーサーバ側の負荷要因として公式に挙げられるのがオフスクリーンレンダリング(マスク・影・グループ不透明度が招く中間バッファへの迂回)で、render hitch の代表原因である。「宣言が重い描画を暗黙に招く」という、宣言的 API の古典的な罠である。

## セキュリティ考察 (Security Considerations)

- **レンダーサーバは全画面を見る単一特権プロセスである**: SurfaceFlinger と同じく、ここへの到達性(サンドボックスのサービス名 lookup 制限)とプロセス自体の堅牢性が壁になる。レイヤツリーのデコード(不信クライアントからの複雑な直列化入力)は、[`mach-ipc-xpc.md`](mach-ipc-xpc.md) で述べた「XPC 境界の手書きバリデーション」問題の大型事例であり、実際にレンダーサーバ境界の脆弱性はサンドボックス脱出の研究対象であり続けている。
- **IOSurface はケイパビリティであり、カーネル攻撃面でもある**: バッファのプロセス間受け渡しは権限の委譲(gralloc と同型)であり、同時に IOSurface カーネル拡張は「全アプリから直接叩ける複雑なオブジェクト管理」として、GPU ドライバと並ぶ権限昇格の常連だった(async_wake 等が IOSurface を経由。詳細は [`mach-ipc-xpc.md`](mach-ipc-xpc.md))。近年のカーネル側の型分離・PAC はここにも適用されている。
- **画面内容の保護**: スクリーンショット・画面収録からの秘匿(セキュアテキスト入力・DRM 動画の合成経路)、背面遷移時のスナップショット制御(アプリスイッチャに機微画面を残さない)は、レンダーサーバが合成の最終地点であることを利用した集中実装である。Android の FLAG_SECURE・SECURE バッファと同じ問題への、同じ位置での解である。
- **描画時間のサイドチャネル**: 合成時間・オフスクリーン迂回の有無が内容に依存することを測る攻撃系譜(クロスオリジンのピクセル推定等)は、サーバ集中型でも消えない。GPU 時間の割当粒度・タイマの粗視化が緩和の主軸である点は両 OS 共通。

## 結論 (Conclusions)

- iOS の描画は「レイヤ木という宣言をサーバに複製し、変化(アニメーション)の評価ごと委譲する」体系であり、Android より一段深くアプリから独立している。**メインスレッドが死んでもアニメーションが動く**という体感の頑健性は、この切断位置の直接の配当である。
- 一方で基本構造(表示リンク同期・トランザクションの原子コミット・ゼロコピーのバッファハンドル・背圧・可変レート・ジャンクの二分類)は Android と完全に収斂しており、**モバイル描画の「正解の形」は切断位置を除いてほぼ一意**に定まったと言える。CATransaction (2007) と BLAST (2021) の年差は、サーバ集中型が最初から持っていた原子性を、分散型が後から獲得した歴史として読める。
- 宣言的アニメーションの恩恵は語彙の制約と表裏であり、「毎フレーム介入したい」用途(ゲーム・インタラクティブ遷移)では両 OS とも同じアプリ駆動モデルに合流する。**分業の境界は能力ではなく信頼と統制の設計**である。
- レンダーサーバ・IOSurface・表示経路はそのまま隔離モデルの要衝であり、graphics は両 OS で「性能の話であると同時にセキュリティ境界の話」である。[`../android/graphics-pipeline.md`](../android/graphics-pipeline.md) と本書で、その対称性が確認できる。

### 次の接続先 (Related Topics)

- [`../android/graphics-pipeline.md`](../android/graphics-pipeline.md) — 全節の対比対象。BufferQueue・SurfaceFlinger・BLAST・フレームペーシング。
- [`mach-ipc-xpc.md`](mach-ipc-xpc.md) — コミットと IOSurface 受け渡しの輸送層。OOL 転送・ポート渡し。
- [`app-lifecycle-runningboard.md`](app-lifecycle-runningboard.md) — 前面/背面遷移と描画権の接続(背面スナップショット・凍結)。
- [`platform-architecture.md`](platform-architecture.md) — backboardd を含むデーモンの連邦の全体像。
- [`../../hardware/hw-sw-interface/mmio-dma-interrupts.md`](../../hardware/hw-sw-interface/mmio-dma-interrupts.md) — IOSurface・ディスプレイパイプの下にある DMA・割り込みの一般論。

## 参考文献 (References)

**一次資料 (Primary — Apple 公式)**

- Apple Developer, *Core Animation Programming Guide*(三つの木・トランザクション・レイヤ描画経路の規範). — [developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html) — Accessed: 2026-07-03。
- Apple WWDC 2020, *Explore UI animation hitches and the render loop*(commit hitch / render hitch の公式分類とレンダーループの解説). — [developer.apple.com/videos](https://developer.apple.com/videos/) — Accessed: 2026-07-03。
- Apple Developer, *Optimizing ProMotion refresh rates for iPhone and iPad* ／ *CADisplayLink*. — [developer.apple.com/documentation/quartzcore/optimizing-promotion-refresh-rates](https://developer.apple.com/documentation/quartzcore/optimizing-promotion-refresh-rates-for-iphone-13-pro-and-ipad-pro) — Accessed: 2026-07-03。
- Apple Developer, *IOSurface*(プロセス間共有バッファの公式 API). — [developer.apple.com/documentation/iosurface](https://developer.apple.com/documentation/iosurface) — Accessed: 2026-07-03。

**二次資料 (Secondary)**

- Levin, J. (2017–). **OS Internals* Vol. I*. Technologeeks Press. — backboardd・レンダーサーバの位置づけの解析。
- Google Project Zero (2017–). async_wake ほか IOSurface 経由のカーネル攻撃の技術分析(セキュリティ考察の一次分析). — [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/) — Accessed: 2026-07-03。
