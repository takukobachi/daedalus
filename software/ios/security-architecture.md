# iOS セキュリティアーキテクチャ深掘り (iOS Security: SEP, Data Protection, Code Signing & Mitigations)

## 概要 (Overview)

[`platform-architecture.md`](platform-architecture.md) は iOS のセキュリティを「配布時に閉じる」設計として一段上から眺めた。本ドキュメントはその実装、すなわち**ブートチェーンとパーソナライズ署名、Secure Enclave (SEP) の内部構造、Data Protection の鍵階層と Keychain、コード署名の強制機構 (AMFI・トラストキャッシュ・エンタイトルメント)、サンドボックスと TCC、エクスプロイト緩和の積層 (PAC → PPL → SPTM → MIE)** に降りる。

iOS のセキュリティを貫く一本の糸は、「**検証の根と強制の主体を、侵害されうる層の外へ置き続ける**」という運動である: 署名検証の根は Boot ROM に、鍵は SEP に、ページテーブルの支配権はカーネルの外 (SPTM) に、メモリ安全はシリコン (MIE) に。カーネルすら信頼リストの一員でしかない縦深構造を、攻撃史（Pegasus・checkm8・FORCEDENTRY）との応答関係として読み解く。分析深度は Level 3。

## 方法論 (Methodology)

- 各機構を「何を守るか・誰が強制するか・その強制主体が侵害されたら何が残るか」の三問で解剖し、防御の縦深を層として図式化する。
- 一次資料は Apple Platform Security ガイドと公開 XNU ソースに置く。Apple が仕様を公開しない部分（sepOS 内部・SPTM 詳細）は、公式記述の範囲と、セキュリティ研究者の解析（Project Zero 等）に依る範囲を区別して記す。
- 攻撃事例は「どの層の仮定が破れ、どの構造対策が生まれたか」の因果としてのみ扱い、悪用手順には立ち入らない。
- Android の対応機構（[`../android/security-model.md`](../android/security-model.md)）との対比を随所に置き、章末に対応表で総括する。

## 発見事項 (Findings)

1. **信頼の根は、変更不能な Boot ROM と、署名を発行し続ける Apple のサーバの二重である。** 各ブート段は前段が署名検証するが、その署名は**端末固有 ID とノンスを含めて Apple が都度発行** (パーソナライズ) するため、「他端末用・過去の正規イメージ」は再生できない。ロールバック防止が端末内カウンタ (Android AVB) ではなく**署名発行側の裁量**にある。これは検証をオンラインサービスに接続した設計で、checkm8 (2019) が示した「Boot ROM のバグは修正不能」という root of trust の脆さへの、上流側からの補強でもある。
2. **SEP は「小さな別のコンピュータ」であり、その RAM は AP から暗号学的に見えない。** 専用コア・専用 Boot ROM・独自 OS (sepOS) を持ち、主メモリ上の自領域は **Memory Protection Engine** がインライン暗号化＋認証（リプレイ防止付き）する。工場で融着され**読み出し経路が存在しない UID 鍵**が全鍵階層の根で、パスコード試行の遅延・生体テンプレートの照合・Data Protection のクラス鍵管理はすべて SEP 内で完結する。「AP が完全に侵害されても鍵素材は出ない」が設計目標として明文化されている。
3. **Data Protection は「認証状態を鍵の存在に写像する」体系である。** ファイルごとの鍵をクラス鍵で包み、クラス鍵の可用性がロック状態に連動する: Complete (ロックで鍵消滅)・**Unless Open (非対称鍵により、ロック中でも新規作成だけは可能)**・Until First Unlock (既定)・No Protection。Unless Open が公開鍵暗号で「書けるが読めない」を作る点は、単純な対称鍵設計では得られない状態機械であり、バックグラウンド受信 (メール添付) という実要件から導かれている。
4. **コード署名の強制は「カーネル内の否定既定」である。** 実行可能ページは署名必須 (AMFI)、プラットフォームバイナリは**トラストキャッシュ**（カーネル保持のハッシュ許可リスト）で照合、App Store 外の証明書チェーン検証は **CoreTrust** がカーネル内で行い (iOS 12+、偽署名受理バグ時代の終止符)、**エンタイトルメントは署名ブロブの中にあり実行時改変が不可能**。「どのコードが・どの権限で走りうるか」の台帳全体が、署名という単一の改竄不能媒体に載っている。
5. **サンドボックスは「プロファイル＋コンテナ＋同意 (TCC)」の三点セットで、エンタイトルメントが横串を通す。** 全サードパーティアプリは同一の container プロファイル（拒否既定）で走り、ファイルアクセスはコンテナに閉じ、連絡先・写真・マイク等は tccd がユーザ同意を台帳管理する。プロセス間で権限を分ける Android (UID) に対し、iOS は**同一 UID 上でポリシと同意だけで隔離する**。強制点が多い分、tccd や有資格デーモンの confused deputy が典型的な迂回経路になる。
6. **エクスプロイト緩和は「カーネルの上」へ積み上がり続けている。** KTRR (カーネルテキスト不変, A10) → PAC (ポインタ署名, A12) → PPL (ページテーブル特権分離, A14 世代) → **SPTM/TXM (iOS 17: ページテーブル管理と署名検証を、カーネルより高い例外レベルの別ソフトウェアへ分離)** → **MIE (Memory Integrity Enforcement, A19/iPhone 17, 2025: 同期 EMTE によるメモリタギングの常時有効化)**。方向は一貫して「**カーネル制圧を『ゲームオーバー』から『一つの層の陥落』へ格下げする**」ことにある。
7. **傭兵スパイウェアが脅威モデルを書き換えた。** Pegasus・FORCEDENTRY (2021, ゼロクリック) は「国家級攻撃者は個人を標的にする」を実証し、応答として **BlastDoor**（メッセージ解析の強隔離）・**Lockdown Mode**（攻撃面の選択的遮断）・**MIE**（メモリ安全の原理的検出）が生まれた。iOS セキュリティの近年の進化は、この攻撃者クラスとの応答関係として最もよく説明できる。

## アーキテクチャ (Architecture)

### 強制主体の縦深 — 誰が侵害されたら何が残るか

```
 [アプリ層]      サンドボックスプロファイル / TCC 同意 / エンタイトルメント
                   強制者: sandbox.kext + tccd + AMFI
 ─────────────────────────────────────────────────────────────
 [カーネル層]    XNU — MAC フック・コード署名検査・ゾーン分離 (kalloc_type)
                   強制者: カーネル自身 (PAC・KTRR で自衛)
 ─────────────────────────────────────────────────────────────
 [カーネルの上]  SPTM: ページテーブル書き換えの唯一の主体
                 TXM:  コード署名ポリシの検証主体
                   強制者: より高い特権レベルの独立コンポーネント
                   → カーネル制圧後も「実行コードの追加・改変」が成立しない
 ─────────────────────────────────────────────────────────────
 [シリコン層]    Boot ROM (検証の根) / SEP (鍵と生体) / MIE (タグ検査)
                   強制者: ハードウェア — ソフトウェア全損でも残る底
```

- Android の対応構造（[`../android/security-model.md`](../android/security-model.md) の三層図）と比べると、iOS は **「カーネルとハードウェアの間」に独自の層を増設**している点が特徴的である（Android 側の対応物は TEE/pKVM が担いつつある）。

### ブートチェーンとパーソナライズ

```
 Boot ROM (SoC 焼き込み・Apple CA 公開鍵内蔵)
   └─ 検証 → iBoot ──検証──▶ カーネルキャッシュ + トラストキャッシュ
        │                      └─ SSV (署名済みシステムボリューム) をマウント
        └─ 復元/更新時: 端末 ECID + ノンス を含む要求を Apple 署名サーバへ
             → 応答 (APTicket) がこの端末・この 1 回にだけ有効な署名になる
 SEP は並行して専用 Boot ROM から sepOS を検証起動 (AP と独立)
```

## 技術詳細 (Technical Details)

### 1. Secure Enclave — 内部構造

- **構成**: 専用 CPU コア（AP とキャッシュ非共有）・専用 Boot ROM・TRNG・鍵導出/AES エンジン。主記憶の専有領域は Memory Protection Engine が**暗号化＋完全性タグ＋リプレイ防止カウンタ**で覆い、AP からは暗号文しか見えない。
- **UID / GID 鍵**: UID は端末固有・製造時融着・**いかなるソフトウェアからも読み出し不能**で、AES エンジンに「使う」ことしかできない。全ユーザデータ鍵はここに錨を下ろすため、チップ外への鍵持ち出し・別端末での復号が原理的に成立しない。
- **パスコードガード**: 試行はすべて SEP 内でカウント・遅延され、UID 絡みの鍵導出が**意図的に低速** (数十 ms 級) に調整されているため、オフライン総当たりも端末内総当たりも時間コストが物理で決まる。これは Android の Gatekeeper/Weaver と同型の「試行制限のハードウェア化」である。
- **生体認証**: Face ID のテンプレートと照合は SEP 側（保護されたニューラルエンジン経路）で行われ、AP には成否と鍵解放イベントだけが渡る。**生体データは端末外に出ない**ことが構造で保証される。

### 2. Data Protection と Keychain

| クラス | 鍵の可用性 | 典型用途 |
|---|---|---|
| Complete | ロック中は鍵消滅 | ヘルスデータ等の高機微 |
| Unless Open | ロック中は**新規作成のみ**可 (ECDH) | 受信中の添付ファイル |
| Until First Unlock | 初回解錠後は保持 (**既定**) | 大半のアプリデータ |
| No Protection | 常時 (UID でのみ包む) | ロック前に要る構成情報 |

- ファイル鍵 ← クラス鍵 ← (パスコード導出鍵 × UID) の階層で、**Effaceable Storage** の鍵を消すだけで全データが瞬時に復号不能になる (リモートワイプの実体)。Android FBE の CE/DE が「ユーザ×2 状態」なのに対し、iOS は**4 クラス＋非対称の書き込み専用状態**を持つ、より細粒度の状態機械である。
- **Keychain** は同じ文法の資格情報版で、項目ごとにクラス（`WhenUnlocked` 等）と **ACL（生体必須・この端末のみ `ThisDeviceOnly`）** を持ち、検査は SEP と securityd が分担する。
- **キーバッグ**: ユーザ・デバイス・バックアップ・エスクロー (MDM)・iCloud の各キーバッグがクラス鍵の束を文脈別に管理する。「バックアップに含まれるか」「新端末へ移行できるか」はキーバッグ設計の帰結として決まる。

### 3. コード署名の強制機構

- **検証の分業**: AMFI (kext) が実行時のページハッシュ照合を、`amfid` がユーザ空間での署名評価を、**CoreTrust** が証明書チェーンの検証をカーネル内で行う。プラットフォームバイナリは**静的トラストキャッシュ**（ブートイメージと共に署名されたハッシュ表）で照合され、amfid の判断すら要らない。**検証者自身への依存を最小化する**再帰的な設計である。
- **エンタイトルメント**: 署名ブロブ内の plist として運ばれ、カーネル・デーモンが XPC 接続相手の資格として参照する（[`mach-ipc-xpc.md`](mach-ipc-xpc.md) の audit token と組で「誰が・何を許されて」が決まる）。強力なエンタイトルメント (`task_for_pid` 系・JIT 許可) は Apple 自身のバイナリと特別なプロビジョニングにしか発行されない。
- **配布経路**: App Store 申請時に Apple が**再署名**し、バイナリは FairPlay 暗号化で包まれる。エンタープライズ/TestFlight/AltStore 系はプロビジョニングプロファイル（エンタイトルメント＋端末リスト＋期限の署名文書）で範囲を限定する。EU の代替配布 (2024+) でも**公証 (notarization) による Apple の検査・署名は残る**。「署名主体としての Apple」はモデルの不変条件である。
- **App Attest / DeviceCheck**: SEP 生成鍵で「正規署名アプリが正規端末で動いている」ことをサーバへ証明する。Play Integrity（[`../android/security-model.md`](../android/security-model.md)）の対応物で、両者とも根はハードウェア鍵アテステーションに収斂した。

### 4. サンドボックス・TCC・プロセス隔離

- **Sandbox.kext** は TrustedBSD MAC フックに載ったポリシエンジンで、プロファイル（SBPL を編纂したもの）がファイル・Mach サービス名 lookup・IOKit クラス・syscall の許可を列挙する。サードパーティアプリは全員同一の `container` プロファイル＋ UUID コンテナパスで走る。**bootstrap 名の lookup 制限**が実質の IPC 到達性制御である点は SELinux の ServiceManager 制限と正確に対応する（[`mach-ipc-xpc.md`](mach-ipc-xpc.md)）。
- **TCC**: 「OS の権限」ではなく「ユーザの同意の台帳」であり、tccd が管理しダイアログの文言・頻度・リセット（不使用アプリの権限失効, iOS 17+ 相当機能）まで含めて設計される。Android の実行時権限と完全に収斂した領域。
- **システム側の隔離**: メディア解析 (BlastDoor)・Web コンテンツ (WebContent プロセス)・ドライバ (DriverKit) と、「**不信入力のパーサを最小権限の別プロセスへ**」が横展開されている。Stagefright 後の Android と同じ処方であり、プラットフォーム横断の定石。

### 5. 緩和の積層 — PAC から MIE へ

- **PAC (arm64e, A12+)**: 戻りアドレス・関数ポインタ・一部カーネルデータ構造のポインタに鍵付き MAC を埋め、偽造ポインタの行使を防ぐ。カーネルと platform バイナリで全面採用され、JOP/ROP と「偽 Mach ポート」系の組み立てコストを恒常的に引き上げた（[`mach-ipc-xpc.md`](mach-ipc-xpc.md)）。
- **PPL → SPTM/TXM**: PPL はページテーブル操作を APRR による特権窓の内側へ隔離した。iOS 17 の **SPTM** はこれを独立コンポーネントへ昇格させ、**TXM** がコード署名ポリシ判断を持つ。ページの「実行可能化」はカーネルの権能ではなくなり、**カーネル任意読み書きを得ても署名外コードの導入が成立しない**世界を目指す。
- **kalloc_type / ゾーン分離**: 型ごとのヒープ分離で UAF の型混同を封じる (iOS 15 世代)。Android の kmalloc 型分離・Scudo と同方向。
- **MIE (A19/iPhone 17, 2025)**: Enhanced MTE (EMTE) を**同期モードで常時有効**にし、タグの機密性保護 (Tag Confidentiality Enforcement) を含めてカーネル・主要ユーザ空間に展開。Android の MTE (Pixel 8+, 選択的有効) に対し、**既定で全ユーザに有効**である点が現在の最大差分で、「メモリ安全違反の検出をシリコンの既定にする」段階への移行を画する。
- **運用系の防御**: Lockdown Mode（攻撃面の選択遮断）・Stolen Device Protection（場所と時間遅延を認証条件に加える）・Rapid Security Response（OS 全体更新を待たない緊急パッチ）がある。技術的緩和と**運用・時間軸の緩和**が並走する。

### 6. Android との対応表

| 領域 | iOS | Android |
|---|---|---|
| ブート検証 | Boot ROM + パーソナライズ署名 (APTicket) | AVB + ロールバックカウンタ |
| 鍵の隔離 | SEP (sepOS, UID 鍵) | TEE KeyMint / StrongBox SE |
| 保存データ | Data Protection 4 クラス + キーバッグ | FBE CE/DE + metadata encryption |
| 試行制限 | SEP パスコードガード | Gatekeeper / Weaver (TEE/SE) |
| 実行コード統制 | 署名必須・JIT 原則禁止・トラストキャッシュ | 署名 v2–v4 + fs-verity (JIT は許容) |
| アプリ隔離 | 同一 UID + サンドボックスプロファイル + TCC | per-UID + SELinux + 実行時権限 |
| IPC 到達性 | bootstrap lookup のプロファイル制限 | ServiceManager への SELinux 制限 |
| 健全性証明 | App Attest / DeviceCheck | Play Integrity / 鍵アテステーション |
| メモリタギング | MIE: 既定有効 (A19+) | MTE: 選択的有効 (Pixel 8+) |
| カーネル外の層 | SPTM / TXM | (pKVM・TEE が部分的に対応) |
| パーサ隔離 | BlastDoor / WebContent | メディアスタック分割 (post-Stagefright) |

## セキュリティ考察 (Security Considerations)

本書全体がセキュリティ分析だが、モデル自体の残余リスクを挙げる。

- **Boot ROM は修正不能な根である**: checkm8 が示した通り、根の脆弱性は当該世代のハードウェア全数に恒久残存する。パーソナライズ署名・SEP の独立性・後段の緩和は「根が割れても鍵と実行統制の一部が残る」ための縦深だが、根の交換は買い替えでしか行えない。**root of trust の不変性は強さであり脆さである**。
- **有資格プロセスの confused deputy**: 同一 UID モデルでは、強いエンタイトルメントを持つデーモンの入力検証不備が、そのままサンドボックス脱出に等しい。XPC 境界の手書きバリデーションが Android の Parcel と同じく最頻の弱点であり続ける（[`mach-ipc-xpc.md`](mach-ipc-xpc.md)）。
- **閉鎖性は検証可能性と交換されている**: sepOS・SPTM の内部は非公開で、第三者検証は攻撃者と同じくリバースエンジニアリングに依る。「Apple を信頼する」ことがモデルの公理であり、Android の（相対的な）検証可能性・改造の自由との根源的なトレードオフ。EU DMA 下の代替配布圏でも公証が残る構造は、この公理の防衛線と読める。
- **ゼロクリック面は縮んだが消えていない**: BlastDoor・Lockdown 以後も、傭兵スパイウェアは露出面（画像コーデック・無線スタック・クラウド連携）を移動しながら継続している。MIE が悪用コストを一段引き上げた直後の攻撃側の適応が、次の設計課題を定義するはずである。
- **物理アクセスと AFU**: 初回解錠後の端末はクラス鍵の多くがメモリに存在し、ロック画面バイパスや有線経路の脆弱性と組むと抽出リスクが残る。Until First Unlock が既定である以上、この境界の攻防は Android の BFU/AFU と同じ地形で続く。

## 結論 (Conclusions)

- iOS のセキュリティは「検証の根と強制の主体を侵害可能な層の外へ移し続ける」運動として一貫している: 署名の根は Boot ROM とサーバへ、鍵は SEP へ、ページテーブルは SPTM へ、メモリ安全はシリコン (MIE) へ。**カーネルはもはや最終防衛線ではなく中間層**である。
- Data Protection・コード署名・サンドボックスはそれぞれ「認証状態・実行資格・到達可能性」を暗号学的/宣言的な台帳に変えた体系であり、エンタイトルメント＝署名内 plist という一点に権限台帳を集約した設計が全体を単純化している。
- 近年の進化 (BlastDoor → Lockdown → SPTM → MIE) は傭兵スパイウェアという攻撃者クラスへの応答として最もよく説明でき、「侵害を前提に、悪用の各段に壁を 1 枚ずつ増やす」方針が明瞭である。Android と比べると、**既定で全員に有効化する速度** (MIE) と**カーネル外の層の厚さ** (SPTM/TXM) が iOS の現在の強みであり、検証可能性・改造の自由がその代価である。
- 両 OS の対応表が示す通り、保存データ暗号化・試行制限・健全性証明・パーサ隔離は完全に収斂した。残る本質的差異は「主体の表現 (UID 対 エンタイトルメント)」と「エコシステムの開放度」であり、セキュリティモデルの比較は結局、**プラットフォームのガバナンス設計の比較**に帰着する。

### 次の接続先 (Related Topics)

- [`../android/security-model.md`](../android/security-model.md) — 同じ切り口の Android 版。対応表の各行の詳細。
- [`platform-architecture.md`](platform-architecture.md) — 署名・サンドボックス・SEP の全体像での位置づけ。
- [`mach-ipc-xpc.md`](mach-ipc-xpc.md) — audit token・エンタイトルメント検査・ポート寿命バグと PAC の攻防。
- [`../../hardware/hw-sw-interface/boot-initialization.md`](../../hardware/hw-sw-interface/boot-initialization.md) — 検証チェーンの一般構造。パーソナライズ署名はその拡張。
- [`../../hardware/cpu/virtual-memory-mmu.md`](../../hardware/cpu/virtual-memory-mmu.md) — SPTM/PPL が操作を独占する対象 (ページテーブル) の一般機構。

## 参考文献 (References)

**一次資料 (Primary — Apple 公式 / 公開ソース)**

- Apple, *Apple Platform Security*（Secure Enclave・Data Protection・コード署名・BlastDoor・Lockdown Mode の公式体系。本書の主参照）. — [support.apple.com/guide/security](https://support.apple.com/guide/security/welcome/web) — Accessed: 2026-07-03。
- Apple Security Research Blog (2025). *Memory Integrity Enforcement*（EMTE 同期モード・Tag Confidentiality Enforcement の公式解説）. — [security.apple.com/blog/memory-integrity-enforcement](https://security.apple.com/blog/memory-integrity-enforcement/) — Accessed: 2026-07-03。
- Apple, XNU 公開ソース（MAC フック・AMFI 連携・kalloc_type）. — [github.com/apple-oss-distributions/xnu](https://github.com/apple-oss-distributions/xnu) — Accessed: 2026-07-03。
- Apple Developer, *DeviceCheck / App Attest*. — [developer.apple.com/documentation/devicecheck](https://developer.apple.com/documentation/devicecheck) — Accessed: 2026-07-03。

**二次資料 (Secondary — 第三者分析)**

- Google Project Zero (2021). *A deep dive into an NSO zero-click iMessage exploit* (FORCEDENTRY) ／ *A Look at iMessage in iOS 14* (BlastDoor). — [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/2021/12/a-deep-dive-into-nso-zero-click.html) — Accessed: 2026-07-03。ゼロクリック攻撃と隔離設計の応答関係の一次分析。
- Citizen Lab (2016–). *Pegasus / Trident 以降の傭兵スパイウェア追跡報告*. — [citizenlab.ca](https://citizenlab.ca/) — Accessed: 2026-07-03。脅威モデル変化の実証記録。
- Levin, J. (2017–). **OS Internals* Vol. III: Security & Insecurity*. Technologeeks Press. — AMFI・サンドボックス・KTRR/PPL の非公式解析の標準文献。
