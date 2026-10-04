# Mach IPC / XPC 深掘り (Mach Ports, Messages & XPC Internals)

## 概要 (Overview)

[`platform-architecture.md`](platform-architecture.md) は iOS の IPC を「Mach ポートの上の XPC」として一段上から眺めた。本ドキュメントはその内部——**ポートと権利 (right) のケイパビリティモデル、メッセージの構造と転送、out-of-line メモリの COW 再マップ、死亡・無送信者通知、launchd の bootstrap、XPC 層、voucher による QoS 伝播**——に降りる。

Mach IPC を深掘りする価値は二つある。第一に、これは Binder（[`../android/binder-internals.md`](../android/binder-internals.md)）と**同じ問題（ケイパビリティ・呼び出し元保証・寿命管理・優先度伝播）を 30 年早く解いた原型**であり、両者の対比が IPC 設計の設計空間を照らすこと。第二に、iOS カーネル攻略の歴史がほぼ **Mach ポートのライフサイクルバグの歴史**であり、防御側の構造対策（zone 分離・PAC・SPTM）を理解する前提になること。分析深度は Level 3。

## 方法論 (Methodology)

- 1 回の同期 RPC（送信 → カーネル → 受信 → 応答）を通しで追い、各段の担当（libxpc / MIG / カーネル ipc サブシステム）とデータ・権利の動きを特定する。
- 中核データ構造（タスクのポート名前空間、`ipc_port`、メッセージ本体と記述子）を先に固定し、機能（権利転送・通知・OOL 転送）をその操作として説明する。
- 一次資料は公開 XNU ソース (`osfmk/ipc/`) と Apple Developer の XPC ドキュメントに置く。XPC のワイヤ形式など非公開部分は、公開ヘッダとセキュリティ研究者の解析に依ることを明示する。
- 各節で Binder の対応機構と対比し、共通の文法と設計判断の分岐点を抽出する。

## 発見事項 (Findings)

1. **ポート権利はタスクごとの名前空間に置かれたケイパビリティである。** ポートはカーネル内のメッセージキュー (`ipc_port`)、ユーザ空間が持つのは**名前（整数）→ 権利**の対応表エントリで、fd 表・Binder の ref 表と同じ「テーブル＋不透明ハンドル」文法。名前は偽造できても、**表に権利がなければカーネルが拒否**する。
2. **受信権は常に一人、送信権は何人でも。** 1 ポートの receive right は単一タスクのみが保持し（サービスの実体）、send right は複製・配布できる（クライアント）。応答には**一度だけ使える send-once right** を渡す——「返信の権利」を使い捨てケイパビリティとして型に持つ点は、Binder のトランザクション ID 管理より明示的な設計。
3. **権利はメッセージに載せて移動・複製・鋳造される。** 記述子の指定 (MOVE_RECEIVE / COPY_SEND / MAKE_SEND / MAKE_SEND_ONCE) に従い、カーネルが**送信側の名前を受信側の名前空間の新しい名前へ翻訳**する。Binder のハンドル翻訳と完全に同型で、receive right そのものを移動できる（サービスの実体ごと引っ越せる）点だけがより一般的。
4. **大きなデータはコピーせず COW で再マップする。** out-of-line (OOL) 記述子のメモリは、受信側アドレス空間へ**仮想メモリ操作 (COW) で写像**される。Binder が「受信側 mmap バッファへの 1 コピー＋サイズ上限 1MB」を選んだのに対し、Mach は「コピーゼロ・サイズ無制限・ただし VM 操作のコストと複雑さ」を選んだ——**同じ問題への逆の割り切り**であり、Binder の大物は fd (ashmem) 渡し、という運用が Mach では言語内機能になっている、とも読める。
5. **launchd の bootstrap は「名前サービス＋起動機＋看取り」の三役である。** サービス名 → send right の解決に加え、**登録サービスの receive right を launchd が予め預かる**ため、(a) 最初のメッセージ到着でプロセスをオンデマンド起動、(b) クラッシュ後も**ポートは生きたまま**再起動して透過的に継続、ができる。ServiceManager（解決のみ）＋ ActivityManager（起動）＋ linkToDeath（看取り）の三機能が、ポートの所有権設計ひとつで実現されている。
6. **呼び出し元の同定は audit token、優先度の伝播は voucher が担う。** 各メッセージにはカーネルが刻印する **audit token**（PID・UID に加え世代番号）が付き、受信側はこれで権限判定する——`getCallingUid()` の対応物であり、**PID 再利用攻撃を世代番号で塞ぐ**分だけ精密。**voucher** は QoS・importance を運び、前面アプリの依頼を受けたデーモンが正しい優先度で走る（Binder の nice 継承の一般化）。優先度逆転はカーネルの **turnstile**（優先度継承の汎用機構）が受ける。
7. **Mach ポートは iOS カーネル攻略の主舞台であり続けた。** Pegasus (CVE-2016-4655/4656)・mach_portal・async_wake など、公開された iOS カーネル攻撃の多くがポートの参照カウント・寿命管理の欠陥を「偽ポート製造 → 任意カーネル読み書き」へ組み立てた。応答は kalloc_type による型分離ヒープ・ポインタの PAC 署名・SPTM と、**「ポート構造体を偽造できても使えなくする」方向**の構造対策である。

## アーキテクチャ (Architecture)

### 層構成 — どこまでがライブラリで、どこからがカーネルか

```
 アプリ/デーモン:  NSXPCConnection (型付きプロキシ) / MIG 生成スタブ
                        │  XPC dictionary / MIG メッセージに直列化
 libxpc / libsystem:  xpc_connection_send_message… → mach_msg() ラッパ
                        │  mach_msg トラップ (送受信兼用の単一入口)
 ─────────────────── ユーザ/カーネル境界 ───────────────────
 XNU ipc サブシステム:  task ごとの ipc_space (名前 → 権利エントリの表)
                         ├─ ipc_port … カーネル内オブジェクト (メッセージキュー)
                         ├─ メッセージ copyin/copyout・権利/OOL 記述子の翻訳
                         ├─ 通知 (no-senders / dead-name / send-once 消滅)
                         └─ turnstile / voucher (優先度・文脈の伝播)
```

- **mach_msg は送信と受信を 1 トラップに兼ねる**: 同期 RPC（送信＋応答受信）が 1 回のカーネル進入で済む最適化が最初から入っている。Binder の `BINDER_WRITE_READ` ioctl（書き＋読みを 1 回で）と同じ着想。
- **ハンドル 0 に相当する特別扱いはタスクの special port 群**: 各タスクは bootstrap ポート（launchd への send right）を継承し、そこから世界が始まる——ServiceManager のハンドル 0 と同じ「最初のケイパビリティ」問題の解。

### メッセージの構造

```
 mach_msg_header_t: 宛先ポート名 / 応答ポート名 (+権利処理の指定 disposition)
 (complex の場合) 記述子列:
    ├─ PORT 記述子   … 権利の move/copy/make (カーネルが名前を翻訳)
    ├─ OOL 記述子    … メモリ領域 (カーネルが COW で受信側へ再マップ)
    └─ OOL_PORTS     … ポート権利の配列
 インラインデータ (小さな本文はここへコピー)
 受信側にはトレイラが付加: audit token (カーネル刻印の送信者情報) 等
```

- Parcel との対比: Binder はフラットなバイト列に `flat_binder_object` を埋めて走査するのに対し、Mach は**記述子を先頭に分離**する。どちらも「カーネルが書き換える部分」を型で区別する必要から来た形。

## 技術詳細 (Technical Details)

### 1. 同期 RPC の一生

```
 [クライアント]                        [カーネル]                      [サーバ]
 応答用ポートを用意 (通常は再利用)
 mach_msg(送信+受信):
   宛先 = サービスの send right ──▶ 権利検査・記述子翻訳
   応答 = MAKE_SEND_ONCE            インライン copyin / OOL は COW 予約
                                    宛先キューへ enqueue、待機スレッド起床 ─▶ mach_msg 受信
 応答ポートで受信待ちブロック                                              処理 (audit token で権限判定)
                       ◀────────── send-once right へ応答を配送 ◀──────── mach_msg 送信
 受信・send-once は消滅 (一度きり)
```

- **send-once right の消滅は保証付き**: サーバがクラッシュしても、カーネルが send-once 消滅通知を応答として配送するため、クライアントは**応答か死亡通知のどちらかを必ず受け取る**。Binder が transaction 失敗コードで返す保証の、権利システムに織り込まれた版。
- **MIG** (Mach Interface Generator) は .defs から送受信スタブを生成する古典 IDL で、カーネル・デーモンの多くの境界は今も MIG。ユーザランドの新規界面はほぼ XPC に移った。

### 2. 権利転送の意味論

| disposition | 効果 | Binder での対応物 |
|---|---|---|
| COPY_SEND | send right を複製して渡す | ハンドルを Parcel に入れて渡す (ref 増設) |
| MOVE_SEND | 自分の send right を手放して渡す | （なし——Binder は常に複製） |
| MAKE_SEND | receive right から send right を鋳造 | 自オブジェクトの初回送信 (node から ref 生成) |
| MAKE_SEND_ONCE | 一度きりの送信権を鋳造 | 応答経路 (transaction の reply) |
| MOVE_RECEIVE | **受信権そのもの**を移す | 対応物なし (BBinder は所有プロセスに固定) |

- MOVE_RECEIVE の存在が Mach の一般性を示す: サービスの実体をプロセス間で引っ越せる。launchd がサービスの receive right を預かって配る運用は、この一般性の上に成り立つ。
- 参照カウント: send right の総数はカーネルが把握し、最後の send right が消えると **no-senders 通知**が receive 側へ届く——「リモート参照が全部消えた」を所有者に伝える Binder のドライバ集計と同じ仕事。

### 3. OOL 転送と仮想メモリ

- OOL 記述子のメモリは `vm_map_copyin`/`copyout` により**物理コピーなしで受信側へ COW 写像**される。どちらかが書いた時点で初めてページ単位のコピーが起きる——読み共有が多い大型データ（辞書・画像）で効く。
- 代償は VM 操作のコスト（マップ書き換え・TLB）と、**受信側アドレス空間に「送信側由来の領域」が出現する**複雑さ。断片化・アドレス予測（セキュリティ）の面倒はこの設計の影であり、Binder が固定 mmap バッファ＋1 コピーで単純化した理由の裏返しでもある（[`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) の COW の応用と限界）。
- ファイルは **fileport**（fd をポート化して送る）で渡せる。fd・ポート・メモリのすべてが「メッセージに載るケイパビリティ」に統一されている。

### 4. 通知 — 寿命管理の機構群

- **dead-name 通知**: 相手の receive right が破壊される（プロセス死を含む）と、手持ちの send right は dead name に変わり、登録者へ通知が届く——`linkToDeath` の対応物。カーネルがプロセス解体と原子的に処理する点も同じで、**「殺される前提の世界」で参照を配る**ための必需品が両 OS で同型なのは偶然ではない（Jetsam / lmkd）。
- **no-senders 通知**: クライアントが全員去ったことをサービスが知り、アイドル退出 (idle exit) する契機になる。launchd のオンデマンド起動と対にして、「**使われていないサービスはメモリに居ない**」を平常運転にする——プロセス＝キャッシュ（Android）に対する、サービス＝オンデマンド（iOS）の実装基盤。

### 5. XPC — 現代の界面

- **xpc オブジェクト**: 辞書・配列・データ・fd・**endpoint**（接続をケイパビリティとして再送可能にしたもの）を持つ型付きコンテナで、シリアライズと mach_msg 詳細を隠蔽する。`NSXPCConnection` はさらにプロトコル（インタフェース宣言）から型付きプロキシを作る——AIDL 相当の層。
- **接続モデル**: XPC 接続はポート対の上の恒常的な双方向チャネルで、切断イベント（相手の死・アイドル退出）がハンドラに届く。**再接続すれば launchd が透過的に再起動**するため、クライアントは「サービスが常に居る」ように書ける——Binder の死亡通知＋手動リトライを、プラットフォームの既定動作に昇格させた形。
- **importance donation**: 前面アプリからのメッセージが背景デーモンを一時的に高優先度へ引き上げる voucher 機構が XPC に統合されており、「重要な仕事の依頼が背景優先度で詰まる」逆転をフレームワーク層で防ぐ。

### 6. Binder との設計比較

| 観点 | Mach IPC (+XPC) | Binder |
|---|---|---|
| ケイパビリティ | ポート権利 (send/receive/send-once の型) | ハンドル (strong/weak 参照) |
| 名前解決 | launchd bootstrap (起動・看取り込み) | ServiceManager (解決のみ) |
| 大データ | OOL: COW 再マップ、0 コピー | 1 コピー + 1MB 上限、大物は fd |
| 呼び出し元同定 | audit token (PID 世代付き) | UID/PID 刻印 |
| 死亡検知 | dead-name / no-senders 通知 | linkToDeath / ref 清算 |
| 優先度伝播 | voucher + turnstile (汎用) | nice/RT 継承 (専用) |
| スレッドモデル | 受信側が自前で待つ (GCD が管理) | ドライバ管理のスレッドプール+又貸し |
| 再入・相互呼び出し | 設計上の保証なし (デッドロックは設計責任) | トランザクションスタックで元スレッドへ配送 |

- 最大の分岐は**スレッドと再入**: Binder は「同期 RPC の相互呼び出し」をカーネルが解く（[`../android/binder-internals.md`](../android/binder-internals.md)）のに対し、Mach/XPC は非同期メッセージ＋GCD を基本形とし、同期の入れ子は作らないのが作法。IPC の意味論がプログラミングモデル（コールバック中心）にまで浸透している。

## セキュリティ考察 (Security Considerations)

- **ポートの寿命バグ＝カーネル制圧、の系譜**: Pegasus/Trident (CVE-2016-4655/4656) はカーネル情報漏洩＋OSUnserializeXML の欠陥、mach_portal (2016) はポート名の混同と launchd の再利用を、async_wake (2017) は IOSurface 経由の参照カウント誤りを、それぞれ「**カーネルに任意のポート構造体を信じ込ませる**」ことへ組み立てた。ケイパビリティ表がカーネル内にある以上、その整合性バグは即・全権限である——Binder の Bad Binder と同じ構図が、より長い歴史で反復されている。
- **構造的対策の積層**: (a) **kalloc_type / zone 分離**——型ごとにヒープを分け、UAF の「別型を重ね置く」定石を封じる。(b) **PAC**——ポート等のカーネルポインタ偽造をポインタ署名で無効化。(c) **SPTM/TXM**——ページテーブルと署名検証をカーネルより上位の権限層へ隔離し、「カーネルを取っても任意物理書き込みに至らない」。攻撃の各段（偽造→読み書き→永続化）に 1 枚ずつ壁を増やす方針が明瞭である。
- **ユーザランド側の作法**: XPC 受信データは任意プロセスが構築しうる不信入力であり、**権限判定は PID でなく audit token で**（PID は再利用される）、**接続の相手はエンタイトルメント・署名要件で検証する**のが規範。サービス側の手書きバリデーション不備は、そのまま「サンドボックス内 → デーモン権限」の昇格経路になる——Parcel の手書き read 列の危険（Binder）と同一の教訓。
- **bootstrap 到達性の制限**: サンドボックスプロファイルは「どのサービス名を lookup できるか」を絞る。ケイパビリティモデルは「渡されなければ呼べない」を保証するが、**名前サービスが誰にでも配るなら意味がない**——SELinux による ServiceManager 到達性制限と正確に対応する、名前解決層の MAC。
- **可用性**: 受信キューの深さ・OOL メモリの吊り上げ（受信側に大量の COW 領域を積ませる）は DoS 面であり、XPC はメッセージサイズ・キュー長の制限で受ける。背圧の所在がドライバ (Binder) ではなくライブラリ＋キュー上限 (XPC) にある点も、両者の分業の差を映す。

## 結論 (Conclusions)

- Mach IPC の見かけの多機能（RPC・名前解決・寿命・優先度・大データ転送）は、「**ポート権利という型付きケイパビリティ＋カーネルの名前翻訳**」という単一の土台から派生している。Binder が参照表とハンドル翻訳から全機能を生やしたのと同じ構造であり、ケイパビリティ IPC の文法が OS 系譜を超えて収斂することを示す。
- 分岐点は割り切りの選択にある: 1 コピー＋上限 (Binder) 対 COW 再マップ (Mach)、カーネルの再入解決 (Binder) 対 非同期基本 (XPC/GCD)、解決のみの名前サービス (ServiceManager) 対 起動・看取り込み (launchd)。**どちらが優れているかではなく、何を単純化の犠牲にしたか**が対比の読みどころである。
- launchd がサービスの receive right を預かる設計は、「オンデマンド起動・透過再起動・アイドル退出」を一つの所有権配置から導いた美しい例であり、iOS の「サービスは居ないのが既定」というメモリ哲学の実装基盤になっている。
- 中核機構は攻撃面の中核でもある——この命題は Binder と Mach で全く同じに成立し、応答（型分離ヒープ・PAC・上位権限層・到達性 MAC）は現代カーネル防御のカタログになっている。IPC の深掘りはプラットフォームセキュリティの深掘りとほぼ同義である。

### 次の接続先 (Related Topics)

- [`platform-architecture.md`](platform-architecture.md) — Mach/XPC が支える iOS 全体像（launchd・凍結・Jetsam・SEP）。
- [`../android/binder-internals.md`](../android/binder-internals.md) — 全節の対比対象。同じ問題への Android の解。
- [`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) — OOL 転送が使う COW・VM remap の一般論。
- [`../concurrency/lock-free-data-structures.md`](../concurrency/lock-free-data-structures.md) — カーネル内参照カウント・寿命管理バグが昇格になる構造の一般論。

## 参考文献 (References)

**一次資料 (Primary — Apple 公式 / 公開ソース)**

- Apple, XNU 公開ソース `osfmk/ipc/`（ipc_port・ipc_space・mach_msg・通知の実装本体）. — [github.com/apple-oss-distributions/xnu](https://github.com/apple-oss-distributions/xnu) — Accessed: 2026-07-03。
- Apple Developer, *XPC* ／ *Creating XPC services*（xpc オブジェクト・接続・NSXPCConnection の規範）. — [developer.apple.com/documentation/xpc](https://developer.apple.com/documentation/xpc) — Accessed: 2026-07-03。
- Apple, *Apple Platform Security*（kalloc_type・PAC・SPTM 等カーネル防御の公式解説）. — [support.apple.com/guide/security](https://support.apple.com/guide/security/welcome/web) — Accessed: 2026-07-03。

**二次資料 (Secondary)**

- Levin, J. (2017–). **OS Internals* Vol. I / III*. Technologeeks Press. — Mach IPC・launchd・XPC の体系的解析。
- Google Project Zero (2016–). *mach_portal* (Ian Beer)、*async_wake* 解説、ならびに Lookout/Citizen Lab (2016). *Trident/Pegasus 技術分析* (CVE-2016-4655/4656). — [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/) — Accessed: 2026-07-03。ポート寿命バグの攻撃組み立ての一次分析。
- Apple WWDC, *Modernizing Grand Central Dispatch Usage* ほか QoS/voucher 関連セッション. — [developer.apple.com/videos](https://developer.apple.com/videos/) — Accessed: 2026-07-03。
