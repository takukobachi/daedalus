# Binder 深掘り (Binder Internals: Driver, Transactions & Object Model)

## 概要 (Overview)

[`platform-architecture.md`](platform-architecture.md) は Binder を「カーネルが呼び出し元を保証するケイパビリティ型 IPC」として一段上から眺めた。本ドキュメントはその内部——**カーネルドライバのオブジェクトモデル、トランザクションの一生、スレッドプール、参照カウント、死亡通知**——に降りる。

Binder を深掘りする価値は、それが単なる「速い IPC」ではなく、**OS の設計判断が詰まった総合機構**である点にある: ハンドルの翻訳（ケイパビリティ）、1コピー転送（mmap 受信バッファ）、呼び出し元スレッドでの再入（デッドロック回避）、参照カウントのプロセス横断化（分散 GC の縮小版）。Android の権限モデル・プロセスライフサイクル・HAL 境界 (Treble) はすべてこの機構の性質の上に組まれている。

歴史的には BeOS/Palm の OpenBinder に由来し、Android で全面採用ののち Linux メインラインに入った (`drivers/android/binder.c`)。分析深度は Level 3。

## 方法論 (Methodology)

- 1回の同期呼び出し (`transact`) を**送信側ユーザ空間 → カーネル → 受信側ユーザ空間 → 応答**まで通しで追い、各段の担当（libbinder / ドライバ）とデータの動きを特定する。
- ドライバの中核データ構造（node / ref / buffer / transaction stack）を先に固定し、機能（ハンドル翻訳・参照カウント・死亡通知）をその操作として説明する。
- 一次資料は AOSP ドキュメントと Linux メインラインのドライバソースに置く。数値（バッファサイズ・スレッド数）は AOSP の既定値であり、ベンダ構成で変わりうることを明示する。

## 発見事項 (Findings)

1. **Binder の名前解決・権限・寿命管理は「ハンドル翻訳」という一つの機構から生えている。** プロセス A のオブジェクトポインタは、カーネル内で `binder_node`（実体、所有プロセスに1つ）と `binder_ref`（参照、リモートプロセスごと）に対応づけられ、Parcel 中の Binder はトランザクション通過時に**ドライバがポインタ⇄ハンドルを書き換える**。ユーザ空間はハンドルを偽造できても、**対応する ref がなければドライバが解決を拒否**する——ケイパビリティの実体はこの参照表である。
2. **データコピーは1回だけ。** 受信側は `/dev/binder` を mmap した**読み取り専用バッファ**を持ち、ドライバは送信側ユーザ空間から受信側バッファへ直接コピーする（送信→カーネル→受信の2コピーではない）。共有メモリの複雑さなしに、パイプ/ソケット（2コピー）より軽い転送を実現する設計点。
3. **同期呼び出しは「呼び出し元スレッドの又貸し」である。** クライアントスレッドは応答までブロックし、その間サーバのスレッドプールが処理する。さらに**トランザクションスタック**により、サーバが同じ呼び出しの文脈でクライアントへ呼び返すと、**ブロック中の元スレッドで実行される**——再帰的な相互呼び出しがデッドロックしない理由であり、スレッド数の爆発も防ぐ。
4. **oneway は「順序付き・流量制限付きの非同期」。** 同一オブジェクトへの oneway 呼び出しは FIFO が保たれ、受信側の非同期バッファ（mmap 領域の半分）を使い尽くすと送信側がブロック/失敗する。「投げっぱなし」に見えて、バックプレッシャが仕込まれている。
5. **参照カウントはプロセス横断の分散 GC の縮小版。** strong/weak 参照をドライバが集計し、全リモート参照が消えた時点で所有プロセスに通知する。プロセスが死ねばその refs は一括清算され、**死亡通知 (linkToDeath)** が購読者に飛ぶ——「相手がいつでも殺される」プラットフォーム（lmkd）で、リーク無しに参照を配れる基盤。
6. **Binder は3ドメインに分割されている。** フレームワーク/アプリ用 `/dev/binder`、HAL 用 `/dev/hwbinder`、ベンダ内部用 `/dev/vndbinder`——それぞれ独立した ServiceManager を持つ。Treble の system/vendor 分離が、IPC の名前空間レベルでも貫かれている。
7. **中核ゆえに、カーネル攻撃面の一級品でもある。** 全アプリから ioctl で直接叩ける複雑な状態機械であり、参照カウントや競合の誤りは即 UAF＝権限昇格になる（CVE-2019-2215 が実例）。AOSP が AIDL の Rust バックエンドを整備した動機の一半はここにある。

## アーキテクチャ (Architecture)

### 層構成 — どこまでがライブラリで、どこからがカーネルか

```
 アプリ/サービス:  AIDL 生成コード (Stub / Proxy)
                       │  Parcel に直列化
 libbinder (C++/Java/Rust):  IBinder / BBinder(実体) / BpBinder(プロキシ)
                       │  ioctl(BINDER_WRITE_READ) — コマンド列の読み書き
 ─────────────────── ユーザ/カーネル境界 ───────────────────
 binder ドライバ:  binder_proc (プロセスごと)
                    ├─ binder_node   … 自プロセスが公開するオブジェクト
                    ├─ binder_ref    … 他プロセスのオブジェクトへの参照 (= ハンドル)
                    ├─ mmap 受信バッファ … 着信トランザクションの置き場 (1コピーの終点)
                    ├─ スレッドプール管理 (looper 登録・待機・起床)
                    └─ transaction stack … 同期呼び出しの文脈チェーン
```

- **BBinder**（サーバ側実体）と **BpBinder**（クライアント側プロキシ、中身はハンドル番号）が libbinder の中心対。AIDL はこの上に型安全な Stub/Proxy を生成するだけで、輸送はすべて `transact`/`onTransact` に落ちる。
- ハンドル 0 は予約されており **ServiceManager** を指す。名前→ハンドルの辞書自体が、一つの Binder サービスとして実装されている（自己ホスト構造）。

### ハンドル翻訳 — ケイパビリティの実装

```
 プロセス A が Parcel に「自分のオブジェクト X」を入れて送信:
   A 内では X はポインタ。ドライバは flat_binder_object を検出し、
   ① X に対応する binder_node がなければ作る
   ② 受信者 B 用の binder_ref を作り、ハンドル番号 h を採番
   ③ Parcel 内の表現を「ポインタ」→「ハンドル h」へ書き換えて B に渡す
 以後 B が h を使うと、ドライバが ref 表で node へ解決して A に届ける。
 B が h を第三者 C へ転送すれば、ドライバが C 用の ref を新設する — 権限の委譲。
```

- **渡されない限り呼べない**（ref が存在しない）、**渡せば委譲になる**（ref が増える）、**偽造できない**（表はカーネル内）。ファイルディスクリプタと同じ文法であり、実際 fd も同様に翻訳して送れる（ashmem/memfd と組み合わせて大容量データを渡す常套手段）。
- SELinux はこの上に重なる: トランザクション自体・ServiceManager への登録/検索が MAC でチェックされ、「どのドメインがどのサービスと話せるか」がポリシで縛られる。

## 技術詳細 (Technical Details)

### 1. 同期トランザクションの一生

```
 [クライアント]                          [ドライバ]                      [サーバ]
 Proxy: Parcel 構築
 ioctl(WRITE_READ, BC_TRANSACTION) ──▶ 送信元 UID/PID を記録
                                       Parcel を検証・オブジェクト翻訳
                                       サーバの mmap バッファへ 1 コピー
                                       待機スレッドを起床 ─────────────▶ BR_TRANSACTION 受領
 呼び出しスレッドは応答待ちで停止          (いなければ spawn 要求)          Stub::onTransact 実行
                                                                        (getCallingUid で権限判定)
                        ◀───────────── 応答 Parcel を同経路で逆向きに ── BC_REPLY
 BR_REPLY 受領・Parcel 解読
```

- 往復のレイテンシは無競合で**数十マイクロ秒**の桁（プロセス間の起床＋1コピー＋復帰）。同一プロセス内なら libbinder がドライバを介さず直接呼ぶ最適化が効く。
- **優先度の継承**: 呼び出し元の nice 値（構成によっては RT 優先度も）をサーバ側処理スレッドへ伝播させる仕組みを持ち、「前面アプリの依頼がバックグラウンド優先度で処理されて詰まる」逆転を防ぐ（[`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) の優先度逆転の Binder 版対処）。

### 2. 受信バッファとサイズの制約

- 受信側 `ProcessState` は初期化時に `/dev/binder` を mmap する（既定 ~1MB 弱）。**トランザクションの実データはこの領域に置かれ、ユーザ空間は読み取り専用で直接参照**する——受信側での追加コピーが不要な理由。
- ゆえに **1MB がプロセス単位の共有上限**（全並行トランザクションの合計）であり、超えると `TransactionTooLargeException` になる。「Binder で画像を送るな、fd を送れ」という実務則の根拠。oneway はさらに厳しく、**非同期用に半分だけ**が割り当てられる。
- 大きな Parcel 向けにはスキャッタギャザー転送（バッファ断片の列として送る）が導入され、HAL 境界の大型構造体コピーを減らしている。

### 3. スレッドプールと再入

- サーバの Binder スレッドは `ProcessState` が管理し、既定の上限は**15 本＋メインスレッド**程度（構成可能）。ドライバは「待機スレッドが尽きた」ときにプロセスへ増員を要求する。
- **トランザクションスタック**: スレッドごとに「いま誰の依頼を処理中か」の連鎖を持つ。A→B 処理中に B が A を呼ぶと、ドライバは連鎖を辿り**A のブロック中スレッドに配送**する。これで (a) 相互呼び出しのデッドロック回避、(b) 文脈の一貫性（同一の論理呼び出しは同一スレッド連鎖で進む）を同時に得る。
- 帰結としての設計則: **Binder スレッドは枯渇する**。上限本数しかないので、onTransact 内で長時間ブロックする実装はプロセス全体の応答性を殺す（ANR の一因）。「Binder ハンドラは短く、重い仕事はワーカへ」——ISR の top/bottom half（[`../../hardware/hw-sw-interface/mmio-dma-interrupts.md`](../../hardware/hw-sw-interface/mmio-dma-interrupts.md)）と同じ規律が現れる。

### 4. oneway の意味論 — 非同期・順序・背圧

- `oneway` 呼び出しは応答を待たずに戻るが、**同一ノード宛ての oneway はドライバのキューで直列化**され、到着順が保たれる。
- 非同期バッファが満杯になると後続の oneway は失敗/ブロックする——雑に投げ続けるクライアントへの**背圧**であり、「oneway だから軽い」とは限らない。通知の洪水はバッファ枯渇として跳ね返る。
- 同期呼び出しとの混在時の順序は保証されないため、状態遷移の通知列に oneway を使う場合はシーケンス番号等の自衛が要る。

### 5. 参照カウントと死亡通知

- ドライバは node ごとに「どのプロセスがいくつ strong/weak 参照を持つか」を集計し、増減をユーザ空間の refcount（BBinder の寿命）と同期させる。**リモートの全参照が消えた**ことを所有側に通知するのはドライバにしかできない仕事である。
- `linkToDeath`: プロキシに死亡受信者を登録すると、**相手プロセスの binder_proc が解体された時点**（クラッシュ・lmkd による kill を含む）で通知が届く。system_server はこれで全アプリの死を検知してサービス登録や WakeLock を清算する——「殺して回収する」プラットフォームの整合性は、この機構が下支えしている。
- ポーリング不要・取りこぼしなし（カーネルがプロセス解体と原子的に処理）という性質は、ユーザ空間の生存監視（ping）では原理的に得られない。

### 6. 3つの Binder ドメインと Treble

| デバイス | 用途 | ServiceManager | 主な話者 |
|---|---|---|---|
| `/dev/binder` | フレームワーク ⇄ アプリ | `servicemanager` | system_server、全アプリ |
| `/dev/hwbinder` | フレームワーク ⇄ HAL | `hwservicemanager` | HAL プロセス (HIDL 期) |
| `/dev/vndbinder` | ベンダプロセス間 | `vndservicemanager` | vendor デーモン |

- ドメイン分離により、**アプリの Binder 流量が HAL 呼び出しを飽和させない**（バッファ・スレッドプールが独立）し、SELinux ポリシも境界ごとに書ける。AIDL for HALs への移行後もこの分離構造は維持されている。
- Treble の「安定インタフェース」の実体は、この hwbinder/AIDL 境界の**バージョン管理された .aidl 定義**であり、VTS が互換性を機械検証する。

### 7. なぜソケットではないのか — 設計比較

| 観点 | Binder | UNIX ソケット/パイプ |
|---|---|---|
| コピー回数 | 1（送信元→受信 mmap） | 2（送信→カーネル→受信） |
| 呼び出し元の同定 | ドライバが UID/PID を刻印 | `SO_PEERCRED` で接続単位（メッセージ単位でない） |
| オブジェクト/fd の受け渡し | ハンドル翻訳で委譲・第三者転送 | fd は SCM_RIGHTS で可、オブジェクト概念なし |
| RPC 形状 | 同期呼び出し＋スレッド又貸しが基本 | バイトストリーム。RPC は自作 |
| 生存監視 | linkToDeath（カーネル保証） | EOF 検知（半端な状態が残りうる） |
| 順序・背圧 | ノード単位 FIFO・バッファ背圧 | ストリーム順・ソケットバッファ |

「RPC・ケイパビリティ・生存監視・優先度伝播を**一つのカーネル機構**に束ねた」ことが Binder の本質であり、その分だけドライバは複雑になった——次節のセキュリティ史はその代償である。

## セキュリティ考察 (Security Considerations)

- **CVE-2019-2215 ("Bad Binder")**: binder_thread の解放をめぐる UAF が、実際に**野生の攻撃 (in-the-wild) で権限昇格に使われた**代表例。全アプリが ioctl で直接叩ける複雑な参照カウント機構、という組み合わせは、[`../concurrency/lock-free-data-structures.md`](../concurrency/lock-free-data-structures.md) で述べた「カーネル内の寿命管理バグ＝即昇格」の最前線である。epoll との相互作用という**機構の交差点**で起きた点も教訓的（単体テストでは出ない）。
- **対策の系譜**: ドライバの継続的な fuzzing/監査に加え、AOSP は AIDL の **Rust バックエンド**と Rust 製サービスを拡大している——ユーザ空間側の Parcel 解析・寿命管理から一クラスのバグを型で排除する方針。ドライバ本体の複雑さは残るため、SELinux による「誰がどのサービスへ届くか」の到達性制限が第二の防壁になる。
- **Parcel は信頼できない入力**: onTransact に届くデータは任意アプリが構築できる。生成コード外の手書き `readXxx` 列の不整合（型混同・境界）は、そのままサービス側の脆弱性になる。「デバイスから来るディスクリプタを検証する」ドライバの規律（[`../../hardware/hw-sw-interface/mmio-dma-interrupts.md`](../../hardware/hw-sw-interface/mmio-dma-interrupts.md)）と同じ構図が、ここではアプリ→システムサービスの向きで成立する。
- **ケイパビリティ漏れ**: Binder/fd を Parcel に入れて渡すことは権限の委譲である。エクスポートされたサービスが強力なオブジェクトをうっかり返すと、UID サンドボックスを迂回する経路になる。「何を返しているか」はインタフェース設計のセキュリティレビュー項目そのもの。
- **可用性**: スレッドプール枯渇（遅い onTransact への同期呼び出しの集中）や oneway バッファの飽和は、アプリから誘発可能な DoS 面である。タイムアウトを持たない同期 transact をシステム側が無防備に受けない、という設計が要る。

## 結論 (Conclusions)

- Binder の見かけの多機能（RPC・名前解決・権限・寿命・優先度）は、**カーネル内の参照表（node/ref）とハンドル翻訳**という単一の土台から派生している。ケイパビリティを IPC の一次概念にしたことが、Android の権限モデルの実装可能性を決めた。
- 1コピー転送・スレッド又貸し・トランザクションスタック・oneway 背圧は、いずれも「モバイルの資源制約でプロセス境界を多用する」ための性能・正しさ両立の工夫であり、1MB 制限やスレッド枯渇といった**運用上の掟はすべて内部構造の直接の影**である。
- linkToDeath と参照清算は「プロセスがいつでも殺される」lmkd の世界観を成立させる対の機構であり、プラットフォーム設計は IPC の性質と不可分であることを示す。
- 中核機構は攻撃面の中核でもある。Bad Binder 以後の Rust 化・SELinux 到達性制限は、「複雑さを消せないなら、書き手のミスのクラスを消す」という現代的な防御の実例である。

### 次の接続先 (Related Topics)

- [`platform-architecture.md`](platform-architecture.md) — Binder が支える全体像（Zygote・lmkd・Treble・サンドボックス）。
- [`../operating-systems/process-thread-scheduling.md`](../operating-systems/process-thread-scheduling.md) — 優先度継承・スレッドプールの一般論。
- [`../concurrency/lock-free-data-structures.md`](../concurrency/lock-free-data-structures.md) — カーネル内寿命管理バグ（UAF）が昇格になる構造の一般論。
- [`../../hardware/hw-sw-interface/abi-syscalls.md`](../../hardware/hw-sw-interface/abi-syscalls.md) — その下の syscall 境界。Binder は ioctl 一本の上に築かれた第二の特権境界。

## 参考文献 (References)

**一次資料 (Primary — AOSP / カーネルソース)**

- AOSP, *AIDL for HALs* ／ *Binder IPC*（Treble・hwbinder/vndbinder ドメインの公式解説）. — [source.android.com/docs/core/architecture/aidl](https://source.android.com/docs/core/architecture/aidl) — Accessed: 2026-07-02。
- Linux mainline, `drivers/android/binder.c`（ドライバ実装本体: node/ref、トランザクション、mmap バッファ）. — [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c) — Accessed: 2026-07-02。
- Android Developers, *AIDL (Android Interface Definition Language)*. — [developer.android.com](https://developer.android.com/develop/background-work/services/aidl) — Accessed: 2026-07-02。oneway・TransactionTooLargeException を含むアプリ側の規範。
- Google Project Zero (2019). *Bad Binder: Android In-The-Wild Exploit* (CVE-2019-2215). — [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/2019/11/bad-binder-android-in-wild-exploit.html) — Accessed: 2026-07-02。UAF の機構と実悪用の一次分析。

**二次資料 (Secondary)**

- Yaghmour, K. (2013). *Embedded Android*. O'Reilly. — ServiceManager・libbinder の層構成の詳説。
- AOSP, *Android Rust*（Rust 製プラットフォームコンポーネントと AIDL Rust バックエンド）. — [source.android.com/docs/setup/build/rust/building-rust-modules/overview](https://source.android.com/docs/setup/build/rust/building-rust-modules/overview) — Accessed: 2026-07-02。
