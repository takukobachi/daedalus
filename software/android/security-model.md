# Android セキュリティモデル深掘り (Android Security Model: Sandbox, Permissions, Keys & Mitigations)

## 概要 (Overview)

[`platform-architecture.md`](platform-architecture.md) は Android のセキュリティを「Verified Boot から実行時隔離まで一本の鎖」として一段上から眺めた。本ドキュメントはその各輪——**多層サンドボックスの実装 (UID/SELinux/seccomp)、権限モデルの意味論、アプリ署名 (v1〜v4) とアイデンティティ、ストレージの隔離と暗号化 (FBE)、Keystore と鍵アテステーション、Play Integrity、エクスプロイト緩和 (CFI/MTE/Rust)**——に降りる。

主題は「個々の機能」ではなく**権限の代数**である: Android では、UID が主体 (principal) を、パーミッションとケイパビリティ (Binder ハンドル・URI grant) が権限を、SELinux が到達可能性を定義し、それらの合成が「あるアプリに何ができるか」を決める。攻撃はこの代数の隙間（委譲の誤り・境界の混同・検証の不一致）を突き、緩和はバグを前提に**壊れ方を制限する**。防御側の視点で全体を体系化する。分析深度は Level 3。

## 方法論 (Methodology)

- 「主体は誰か・権限はどう表現されるか・誰が検査するか (カーネル/system_server/アプリ自身)」の三問を各機構に適用し、検査点の層 (カーネル MAC / フレームワーク / ハードウェア) を明示する。
- 一次資料は AOSP のセキュリティ公式ドキュメントと Android Security Bulletin、学術側は Mayrhofer らによる Android セキュリティモデルの体系論文に置く。
- 歴史的事件 (Janus・Stagefright・Bad Binder 等) は「どの層の仮定が破れたか」の事例としてのみ扱い、悪用手順には立ち入らない。
- iOS の対応機構（[`../ios/security-architecture.md`](../ios/security-architecture.md)）との対比を随所に置き、設計判断の分岐点を抽出する。

## 発見事項 (Findings)

1. **Android の権限検査は三層に分散しており、層ごとに主体の表現が違う。** カーネル層は UID/GID と SELinux ラベル、フレームワーク層は Binder が刻印する呼び出し元 UID＋パッケージのパーミッション表、アプリ層は URI grant や PendingIntent の**ケイパビリティ**。攻撃面の多く（confused deputy）は**層をまたぐ翻訳点**——UID からパッケージ名への逆引き、PendingIntent の身元借用——に集中する。
2. **アプリのアイデンティティは「署名鍵」である。** 更新の受理・signature パーミッション・sharedUserId はすべて署名鍵の一致で判定される。v1 (JAR) 署名の「検証対象と実行対象のずれ」を突いた Janus (2017) を教訓に、v2 以降は **APK 全体のバイト列に署名**し、v3 が鍵ローテーション（旧鍵から新鍵への系譜の証明）、v4 が fs-verity によるストリーミング検証を足した。**「何に署名するか」の設計ミスが一世代の脆弱性を生んだ**典型例である。
3. **ストレージの隔離は「ファイルシステム」から「メディエータ」へ移行した。** scoped storage (Android 10+) により共有ストレージへの生パス自由アクセスは廃止され、MediaStore/SAF という**仲介 API 越しの、種別と所有に基づくアクセス**になった。ファイルシステムの DAC から「フレームワークが意味論を知った上で仲裁する」モデルへの移行であり、権限のインフレ（READ_EXTERNAL_STORAGE が事実上「全部読める」だった）への構造的応答。
4. **暗号化の単位は「ディスク」から「ユーザ×状態」になった。** FBE (File-Based Encryption) はファイルごとの鍵をユーザごとの **CE (Credential Encrypted) / DE (Device Encrypted)** クラス鍵で包む。CE 鍵はロック解除まで存在しないため、**再起動直後 (BFU) は大半のデータが復号不能**——iOS の Data Protection クラスと同じ「認証状態を鍵の存在に写像する」文法に、両者が独立に到達した。
5. **鍵は「使えるが、見えない」。** Keystore の鍵操作は TEE (KeyMint) / StrongBox (独立 SE) 内で行われ、鍵素材は Android 側に出ない。**鍵アテステーション**は「この鍵がハードウェア内にあり、端末は Verified Boot が green で、OS パッチレベルは X である」ことを Google ルートに連なる証明書チェーンで第三者に証明する——**端末の健全性がリモートに検証可能な述語**になった転換点で、Play Integrity の STRONG 判定の土台でもある。
6. **エクスプロイト緩和は「確率の緩和」から「クラスの根絶」へ進んでいる。** ASLR/W^X（回避可能な確率的防御）→ CFI・ShadowCallStack（制御フローの型強制）→ **MTE (Armv9 メモリタギング, Pixel 8+)** による UAF/オーバーフローの**検出保証**→ 新規コードの Rust 化（クラスの根絶）。Android のメモリ安全脆弱性比率が Rust 導入後に**実測で急減**した事実は、「言語の選択はセキュリティ機構である」ことの最初の大規模な実証になった。
7. **更新の速さそのものがセキュリティ機構である。** Treble/APEX/Mainline（[`platform-architecture.md`](platform-architecture.md)）は「パッチが端末に届くまでの日数」を縮める構造改革であり、月例 Security Bulletin ＋ Play 経由のモジュール更新で、**脆弱性の寿命**を攻撃者の武器化速度と競わせている。緩和・隔離・更新の三位が現代の防御線である。

## アーキテクチャ (Architecture)

### 権限検査の三層と検査点

```
 [アプリ層]     URI permission grant / PendingIntent / Binder ハンドルの委譲
                  検査者: 受け取ったアプリ・system_server (grant 表)
 ────────────────────────────────────────────────────────────
 [フレームワーク層] パーミッション (install/runtime/signature/special)
                  検査者: system_server — Binder が刻印した UID で
                  checkPermission(uid, permission) を照合
 ────────────────────────────────────────────────────────────
 [カーネル層]    UID/GID DAC ＋ SELinux MAC ＋ seccomp-bpf
                  検査者: カーネル — プロセス資格情報とラベルで強制
 ────────────────────────────────────────────────────────────
 [ハードウェア層] Verified Boot (AVB) / KeyMint (TEE) / StrongBox (SE)
                  検査者: ブートローダ・隔離実行環境 — OS 侵害後も残る根
```

- **下の層ほど「OS が侵害されても残る」**: フレームワークの検査は system_server の健全性に依存するが、SELinux はカーネルが、鍵の隔離はハードウェアが守る。多層防御とは「各層の破れ方に対する下位層の生存」の設計である。

### SELinux — untrusted_app の実装

- 全アプリは `untrusted_app` 系ドメインで走り、**API レベル別のドメイン分割** (`untrusted_app_25` 等) が「古い targetSdk にだけ許される緩い挙動」を新しいアプリに波及させない。さらにアプリごとに **MLS カテゴリ**が付与され、SELinux レベルでもアプリ間が相互不可視になる（UID 分離の MAC 版二重化）。
- ポリシは Treble に合わせて platform/vendor に分割され、ベンダ拡張が platform の保証を弱められない構造を持つ。**「SELinux は root を信じない」**——root で動くデーモンの侵害が全損にならないのは、ドメインが can-do を列挙しているからである。

## 技術詳細 (Technical Details)

### 1. 権限モデルの意味論

- **保護レベル**: `normal`（インストールで自動付与・低リスク）／`dangerous`（実行時ダイアログ, Android 6+）／`signature`（同一署名鍵のみ・プラットフォーム連携の背骨）／special access（SYSTEM_ALERT_WINDOW 等、設定画面での明示付与）。
- **縮小の系譜**: 権限グループ単位の付与 → 一回限り (one-time) 許可・使用中のみ (while-in-use) の位置情報 (Android 10–11) → **不使用アプリの権限自動失効** (Android 11+) → 近似位置の選択・写真の部分選択 (Android 13–14)。方向は一貫して「**付与を状態ではなくイベントに近づける**」であり、iOS の TCC と収斂している。
- **confused deputy の定番と対策**: (a) exported コンポーネントへの外部 Intent——Android 12+ で `android:exported` の明示が必須化。(b) PendingIntent の身元借用——mutability (`FLAG_IMMUTABLE`) の明示必須化。(c) UID→パッケージの曖昧さ (sharedUserId)——非推奨化。**「暗黙の既定が穴になる」→「明示を必須化する」**が API 進化の反復パターンである。

### 2. アプリ署名 — v1 から v4、鍵の運用

| 版 | 対象 | 要点 |
|---|---|---|
| v1 (JAR) | エントリごと | ZIP 構造の非署名領域が残る——Janus (DEX 前置) の温床 |
| v2 (Android 7) | APK 全体のバイト列 | 「検証対象＝インストール対象」を回復。Merkle 的一括ハッシュ |
| v3 (Android 9) | v2 + 系譜 | **鍵ローテーション**: 旧鍵が新鍵を裏書きする proof-of-rotation |
| v4 (Android 11) | fs-verity 木 | ページ単位の遅延検証——**ストリーミングインストール**を可能に |

- **Play App Signing**: 配布鍵を Google が保管・代行署名し、開発者はアップロード鍵のみ持つ。鍵紛失・漏洩リスクの移転と引き換えに、**ストアが署名主体になる**——「開発者署名が唯一の身元」だった初期モデルからの静かな転換であり、iOS の「Apple が再署名する」App Store モデルとの距離が縮まった点として読める。
- v4 の fs-verity は dm-verity（ブロックデバイス全体）のファイル版で、`/data` 上の APK という「Verified Boot 圏外の弱い環」（[`art-runtime-internals.md`](art-runtime-internals.md)）に読み出し時検証を持ち込む。

### 3. ストレージ — scoped storage と FBE

- **scoped storage**: アプリ固有ディレクトリ（無条件）／自分が作ったメディア（MediaStore 経由・無条件）／他アプリのメディア（READ_MEDIA_* 権限）／任意ファイル（SAF のユーザ選択＝**ユーザ操作そのものが権限付与**）。「どの API で触るか」が権限の粒度を規定する。
- **FBE の鍵階層**: ファイル鍵 ← クラス鍵 (CE/DE, ユーザごと) ← ユーザ資格情報（PIN/パターン）と TEE 内の端末固有鍵の合成。Gatekeeper/Weaver が**試行回数の制限とスロットリングを TEE/SE 内で**強制し、オフライン総当たりを端末内の物理的制約に変える。metadata encryption がファイル名・レイアウトも覆う。
- **BFU/AFU の境界**: 再起動直後 (Before First Unlock) は CE 鍵が不存在。直接ブート対応アプリだけが DE 領域で動ける——「アラームは鳴るが、メッセージは読めない」の実装。フォレンジクスと防御の攻防はこの境界線上で行われる。

### 4. Keystore・アテステーション・Play Integrity

- **KeyMint (TEE)**: 鍵生成・署名・復号を TEE 内で実行し、鍵は端末外・OS 側に出ない。使用条件（ユーザ認証後 N 秒のみ・生体必須・特定用途のみ）を**鍵のメタデータとして TEE が強制**する。**StrongBox** は専用セキュアエレメント上の実装で、TEE より狭い攻撃面（ただし低速・容量制約）を選べる。
- **鍵アテステーション**: 鍵ペア生成時に、鍵の属性・Verified Boot 状態・OS/パッチレベル・端末識別子を含む証明書チェーン（Google ハードウェアアテステーションルート起点）を発行。リモートサーバは**端末とアプリの健全性を暗号学的に検証**できる。
- **Play Integrity API**: 旧 SafetyNet の後継。app integrity（署名一致）・device integrity（MEETS_DEVICE_INTEGRITY / 上位の MEETS_STRONG_INTEGRITY はハードウェアアテステーション必須）・アカウント正当性の判定を返す。**「改造端末をどう扱うか」をアプリ開発者のリスク判断として API 化**したもので、iOS の App Attest/DeviceCheck の対応物。root 検出とのいたちごっこを、ハードウェアの根に錨を下ろすことで打ち切る方向にある。

### 5. エクスプロイト緩和の現在地

- **制御フロー**: LLVM CFI（前方エッジ: 関数ポインタの型集合強制）＋ ShadowCallStack（後方エッジ: 戻りアドレスの別スタック退避, AArch64）がカーネル・主要ユーザ空間に適用。ARMv8.5+ では BTI/PAC がハードウェア支援を足す。
- **メモリアロケータ**: Scudo（ユーザ空間既定・ヘッダ検証と隔離）、GWP-ASan（本番端末で低頻度サンプリングされる UAF/OOB 検出——**フィールドをファザーにする**発想）、カーネルは kmalloc の型分離等。
- **MTE (Pixel 8+/Armv9)**: ポインタ上位ビットとメモリのタグ一致をハードウェアが検査し、UAF・線形オーバーフローを**確率でなく原理で**検出する。開発者向け・高リスクユーザ向け (Advanced Protection) に有効化が進む——iOS の MIE（[`../ios/security-architecture.md`](../ios/security-architecture.md)）と同じ方向で、既定有効化の範囲が両者の現在の差分。
- **Rust**: 新規ネイティブコードの主要言語として採用され、AOSP のメモリ安全脆弱性の比率は導入後に大きく低下した（Google の公開統計）。Binder ドライバの Rust 再実装・Rust 製システムコンポーネント（[`binder-internals.md`](binder-internals.md)）はその実践。**緩和（壊れても耐える）と根絶（壊れなくする）の役割分担**が明確になった。

## セキュリティ考察 (Security Considerations)

本書全体がセキュリティ分析だが、モデル自体の残余リスク——**どの仮定が破れると何が失われるか**——を挙げる。

- **system_server は単一障害点である**: フレームワーク層の権限検査は system_server 内で行われるため、その侵害は「全アプリの権限表の書き換え」に等しい。SELinux とハードウェア層（鍵・Verified Boot）は残るが、ユーザデータの機密性は大きく失われる。巨大単一プロセスという設計（[`platform-architecture.md`](platform-architecture.md)、[`system-server-services.md`](system-server-services.md)）のセキュリティ上の代償であり、iOS の「デーモンの連邦」との構造的差異が最も効く点。
- **ベンダコードは品質の分散が大きい**: SoC ベンダのドライバ・HAL（GPU・モデム・カメラ）は Security Bulletin の常連であり、Treble の境界はあっても**カーネル空間のベンダドライバは依然カーネル権限**で走る。エコシステムの開放性（多ベンダ）がそのまま攻撃面の分散になる——Android モデル固有の宿命。
- **ユーザの同意は最弱の輪であり続ける**: サイドロードの許可・アクセシビリティサービスの悪用・オーバーレイによる UI 偽装は、権限モデルの外側（ユーザの判断）を突く。Android 13+ の「サイドロードアプリへのアクセシビリティ制限」等、**同意経路そのものへの摩擦の設計**が対策の主戦場になっている。
- **物理アクセスと BFU/AFU**: AFU 状態の端末からのデータ抽出（ロック画面バイパスや lockscreen 脆弱性経由）は、鍵がメモリに存在する以上、原理的に消せない残余リスクである。防御は「BFU に戻す（再起動）」「認証必須鍵（Keystore の auth-bound）」の利用拡大に向かう。
- **アテステーションの中央集権性**: Play Integrity への依存は、カスタム ROM・自由なブートローダ解錠との緊張を生む。「健全性の証明」と「所有者の改造の自由」のトレードオフは技術でなくガバナンスの問題として残る。

## 結論 (Conclusions)

- Android のセキュリティモデルは「UID という主体・パーミッション/ケイパビリティという権限・SELinux という到達可能性」の三層代数であり、歴史的な脆弱性の多くは層間の翻訳点 (confused deputy) と「暗黙の既定」に集中した。API の進化は一貫して**明示の必須化と付与のイベント化**に向かっている。
- 署名 (v2+/ローテーション/fs-verity)・FBE (CE/DE)・鍵アテステーションは、それぞれ「アイデンティティ・保存データ・端末健全性」を暗号学的な述語に変えた。**検証可能性がモデルの通貨**になり、その根は一貫してハードウェア（TEE/SE/AVB）に置かれる。
- 緩和の現在地は「確率的緩和 → 型・タグによる強制 (CFI/MTE) → 言語による根絶 (Rust)」の三段推移で示せる。Rust 導入後のメモリ安全脆弱性の実測減は、プラットフォームセキュリティにおける言語選択の重みを確定させた。
- 残余リスクは構造に根差す: system_server の単一性、ベンダカーネルコード、ユーザ同意経路、AFU 状態の物理アクセス。次の設計課題はこれらの**構造そのもの**への手当てであり、iOS との対比（[`../ios/security-architecture.md`](../ios/security-architecture.md)）は互いの盲点を照らす最良の鏡である。

### 次の接続先 (Related Topics)

- [`../ios/security-architecture.md`](../ios/security-architecture.md) — 同じ切り口の iOS 版。Data Protection・App Attest・MIE との対比。
- [`platform-architecture.md`](platform-architecture.md) — Verified Boot・SELinux・Keystore の全体像での位置づけ。
- [`binder-internals.md`](binder-internals.md) — フレームワーク層検査の土台（呼び出し元 UID 刻印）と Rust 化の実践。
- [`art-runtime-internals.md`](art-runtime-internals.md) — 検証器・JIT・JNI という実行系側の型安全境界。
- [`../../hardware/automotive/cybersecurity-iso21434.md`](../../hardware/automotive/cybersecurity-iso21434.md) — 多層防御・HSM・アテステーションの車載版設計文法。
- [`package-management.md`](package-management.md) — 権限検査の第一層 (UID) を割り当てる側。installd・packages.xml・パッケージ可視性。
- [`boot-init-properties.md`](boot-init-properties.md) — ブートローダーのアンロック・fastboot・init による SELinux ポリシー読み込み。
- [`system-server-services.md`](system-server-services.md) — 単一障害点としての system_server の内部構造。
- [`process-thread-model.md`](process-thread-model.md) — UID/プロセスと SELinux ドメインの対応。

## 参考文献 (References)

**一次資料 (Primary — AOSP / Google 公式)**

- AOSP, *Security overview*（サンドボックス・SELinux・Verified Boot・暗号化の公式体系）. — [source.android.com/docs/security](https://source.android.com/docs/security) — Accessed: 2026-07-03。
- AOSP, *Application signing* ／ *APK Signature Scheme v2/v3/v4*. — [source.android.com/docs/security/features/apksigning](https://source.android.com/docs/security/features/apksigning) — Accessed: 2026-07-03。
- AOSP, *File-Based Encryption* ／ *Gatekeeper* ／ *Key and ID Attestation*. — [source.android.com/docs/security/features/encryption/file-based](https://source.android.com/docs/security/features/encryption/file-based) — Accessed: 2026-07-03。
- Android Developers, *Play Integrity API* ／ *Permissions on Android*. — [developer.android.com/google/play/integrity](https://developer.android.com/google/play/integrity) — Accessed: 2026-07-03。
- Google Security Blog, *Memory Safe Languages in Android 13*（Rust 導入とメモリ安全脆弱性比率の実測統計）. — [security.googleblog.com](https://security.googleblog.com/2022/12/memory-safe-languages-in-android-13.html) — Accessed: 2026-07-03。

**査読論文・二次資料 (Peer-reviewed / Secondary)**

- Mayrhofer, R., Stoep, J. V., Brubaker, C., & Kralevich, N. (2021). The Android Platform Security Model. *ACM Transactions on Privacy and Security*, 24(3), 1–35. DOI: 10.1145/3448609 — プラットフォーム設計者自身による脅威モデルと設計原則の体系化。
- Google Project Zero (2017–). Janus (CVE-2017-13156)・Bad Binder ほか Android 事例の技術分析. — [googleprojectzero.blogspot.com](https://googleprojectzero.blogspot.com/) — Accessed: 2026-07-03。
- Arm, *Memory Tagging Extension (MTE) 仕様・ホワイトペーパー*. — [developer.arm.com](https://developer.arm.com/documentation/108035/latest/) — Accessed: 2026-07-03。
- 有野和真 (2017).『Androidを支える技術〈II〉真のマルチタスクに挑んだモバイルOSの心臓部』技術評論社. ISBN 978-4-7741-8861-4 — 第2章 2.1〜2.4（アプリをユーザーに見立てる UID モデル・installd の権限・packages.xml）。Android 7.0 Nougat 基準。書籍内コードは著者による簡略化であり、現行 AOSP との対応は [`../../references/books/book-android-internals-arino.md`](../../references/books/book-android-internals-arino.md) を参照。
