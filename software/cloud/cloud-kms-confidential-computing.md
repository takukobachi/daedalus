# クラウドの鍵管理と機密計算 (Cloud Key Management & Confidential Computing: KMS, Envelope Encryption & Nitro Enclaves)

## 概要 (Overview)

クラウドの暗号化は「アルゴリズムの問題」ではない。AES は 20 年以上破られておらず、破綻は常に**鍵の扱い**——配布・保管・認可・失効——で起きる。**KMS (Key Management Service)** はこの鍵管理を API 化したサービスであり、その第一原理はただ一つ：**鍵そのものは決して外に出さず、鍵を使った演算だけを売る**。アプリケーションが受け取るのは暗号化・復号・署名の**結果**であって鍵ではなく、したがって鍵の窃取は「ファイルのコピー」から「認可された API 呼び出しの獲得」へと問題の形を変える——つまり鍵管理は [`cloud-iam-authorization.md`](cloud-iam-authorization.md) の認可・監査・失効の土俵に載る。

その先にあるのが**機密計算 (confidential computing)** である。保存時暗号化と通信路暗号化が普及した現在、残る露出は**処理中のデータ**——メモリ上の平文と、それを覗きうるホスト・運用者・同居プロセスである。AWS Nitro Enclaves は、インスタンスから CPU とメモリを切り出した隔離実行環境と、**構成証明 (attestation)**——「いま動いているのはこのハッシュのコードである」という署名付き証明——を組み合わせ、KMS のポリシー条件に「コードの同一性」を書けるようにした。認可の主語が「誰が呼んだか」から「**何のコードが呼んだか**」へ拡張される——[`cloud-iam-authorization.md`](cloud-iam-authorization.md) が「暗黙の文脈を偽造不能な明示的文脈へ置き換える歴史」と要約した進化の、現時点の先端である。本ドキュメントは KMS の鍵階層とエンベロープ暗号化を機構から追い、Nitro Enclaves の隔離と attestation を読み、最後に「機密計算は誰を信頼から外せるのか」という保証等級の正直な評価を行う。分析深度は Level 3。

## 方法論 (Methodology)

- 暗号プリミティブ (AES-GCM, HMAC, 公開鍵暗号) は [`../../foundations/mathematics/number-theory-algebra/public-key-cryptography.md`](../../foundations/mathematics/number-theory-algebra/public-key-cryptography.md) と [`finite-fields-coding.md`](../../foundations/mathematics/number-theory-algebra/finite-fields-coding.md) に、TEE のハードウェア機構とサイドチャネル史は [`../../hardware/security/side-channels-tee-secure-boot.md`](../../hardware/security/side-channels-tee-secure-boot.md) に委ね、本書は「鍵とコードの認可アーキテクチャ」に集中する。
- KMS 内部は AWS の暗号設計文書 (*KMS Cryptographic Details*)、Nitro の隔離設計は *The Security Design of the AWS Nitro System*——いずれも**ベンダー一次資料**であり、自社仕様の規範記述として扱いつつ、独立評価 (NCC Group の公開監査報告——AWS 委託だが独立機関による) と学術側の TEE 研究で相互検証する。
- 鍵管理の一般原則 (鍵の分離・ローテーション・危殆化対応) は NIST SP 800-57 を規範とし、ベンダー機構をこの標準語彙へ写像して評価する。

## 発見事項 (Findings)

1. **KMS の本体は「鍵を持たないことを API 契約にした」ことである。** KMS の鍵 (KMS key / CMK) は FIPS 検証済み HSM 群の中で生成され、平文のままエクスポートする API が**存在しない**。呼び出せるのは Encrypt/Decrypt/Sign/GenerateDataKey といった演算だけで、全呼び出しが SigV4 認証・IAM/キーポリシー評価・CloudTrail 監査を通る ([`cloud-iam-authorization.md`](cloud-iam-authorization.md))。この設計の帰結として、鍵の危殆化シナリオから「サーバ侵入による鍵ファイルの持ち出し」が消え、残るのは「認可の獲得」だけになる——つまり**鍵の安全性が認可の安全性に還元される**。逆に言えば、KMS を使っても IAM が緩ければ何も守られていない。

2. **エンベロープ暗号化は性能と権限の分業である。** 大量データを KMS の API 越しに暗号化するのはレイテンシ・スループット・費用の全てで成立しない。そこで**データ鍵 (DEK)** をローカルで使い (AES-GCM で GB 級を高速に)、DEK 自体を KMS の鍵 (KEK) で包んで暗号文と一緒に保存する。復号には「包まれた DEK を KMS に解いてもらう」1 回の呼び出しだけが要る——**バルク暗号は分散、認可判定は集中**という分業であり、S3・EBS・DynamoDB の保存時暗号化 ([`cloud-block-storage.md`](cloud-block-storage.md), [`cloud-databases-aurora-dynamodb.md`](cloud-databases-aurora-dynamodb.md)) はすべてこの構造である。副産物として **crypto-shredding**——KEK を消せば、散らばった暗号文が物理削除なしで一斉に読めなくなる——が手に入る。ブロックストレージ編で見た「鍵の無効化 = リモート消去」の一般形である。

3. **encryption context は「暗号文の取り違え」を認可で殺す。** エンベロープの弱点は、正しく復号できる暗号文を**別の場所に差し替える**攻撃 (ciphertext substitution) が暗号的には検出できないことである。KMS の encryption context は、暗号化時に指定したキー値組 (例: `{"volume-id": "vol-123"}`) を AAD (authenticated additional data) として AES-GCM の認証タグに焼き込み、**復号時に同じ文脈を要求する**。文脈が合わなければ復号は失敗し、さらに文脈は IAM ポリシーの条件・CloudTrail のログにも現れる——「この鍵はこのボリュームのためにしか解かれない」という束縛を、暗号 (完全性) と認可 (条件) の両面で同時に張る機構である。

4. **鍵階層は信頼の階層であり、どこまで登るかが主権の選択になる。** HSM のドメイン鍵 → KMS キー → データ鍵という階層は、NIST SP 800-57 の鍵分離原則 (用途・寿命・露出の異なる鍵を混ぜない) の実装である。既定では階層の根は AWS の HSM にあるが、**BYOK** (鍵素材の持ち込み)・**外部キーストア** (根を顧客の HSM に置き、KMS は都度問い合わせる) と、根を顧客側へ引き上げる選択肢が連続的に用意されている。ただし根を外に出すほど、可用性 (顧客 HSM の停止 = 全データ読めず) と運用責任も顧客へ移る——[`multi-region-global-architecture.md`](multi-region-global-architecture.md) の主権要件と同じく、**主権と可用性の交換**が価格表になっている構図である。

5. **Nitro Enclaves の隔離は「機能の削除」で買われている——Firecracker と同じ制約の勝利。** Enclave は親インスタンスから vCPU とメモリを割いて作られる隔離 VM で、**ネットワークなし・永続ストレージなし・対話的アクセスなし**、外界との接続は親インスタンスとの vsock 1 本だけである。SSH で入れず、ディスクに何も残らず、外に direct に繋がらない——攻撃面と漏えい経路を列挙可能な最小集合まで削った設計であり、[`serverless-faas-internals.md`](serverless-faas-internals.md) で見た「要らない自由度を捨てるほど守りやすい」原理の再演である。隔離の実体は Nitro ハイパーバイザによるメモリ・CPU の分離 (ベンダー一次資料) で、親インスタンスの root でも enclave のメモリは読めないと主張される。

6. **Attestation は「コードへの認可」を可能にする——認可の主語の拡張。** Enclave の起動時、Nitro システムはイメージの測定値 (PCR: イメージ全体・カーネル・アプリのハッシュ、親インスタンスの ID 等) を計測し、AWS の認証局に連なる鍵で署名した **attestation document** を発行する。KMS はポリシー条件 (`kms:RecipientAttestation:PCR0` 等) でこの測定値を検査し、**「特定ハッシュのコードとして起動した enclave にだけ」復号結果を返す**——復号結果自体も attestation document 内の公開鍵で暗号化されて返るため、経路上の親インスタンスにも読めない。ここで起きているのは認可の主語の交代である：IAM ロール (誰) では「同じロールを持つ侵害されたプロセス」を区別できないが、attestation (何のコード) はそれを区別する。秘密は「人と役割」ではなく「検証されたコード」に紐づく——最小権限の極限形と言える。

7. **「誰を信頼から外せたか」は保証の等級を分けて評価しなければならない。** Nitro Enclaves が外すのは「親インスタンスの侵害者・同僚テナント・インスタンス上のマルウェア」であり、**クラウド事業者そのものは信頼の根に残る**——attestation の署名鍵も HSM も Nitro ハイパーバイザも AWS の管理下にあり、「AWS 運用者はテナントデータにアクセスする手段を持たない」という主張は設計・組織統制による保証 (ベンダー一次資料が明言し、NCC Group の独立評価が設計主張の整合性を確認) であって、数学的保証ではない。一方 Intel SGX 系は「事業者すら信頼しない」ことを目標に掲げたが、マイクロアーキテクチャのサイドチャネルで実装が繰り返し破られた ([`../../hardware/security/side-channels-tee-secure-boot.md`](../../hardware/security/side-channels-tee-secure-boot.md))。**脅威モデルの野心と実装の頑健さは別の軸**であり、「TEE だから安全」「事業者を信頼しなくてよい」という短絡は、どちらの軸の評価も誤る。実務の問いは常に「私の脅威モデルで外したい相手は誰で、この機構はその相手をどの等級の保証で外すか」である。

## アーキテクチャ (Architecture)

### エンベロープ暗号化 — 分業の構造

```
 暗号化 (書き込み時)                          復号 (読み出し時)
 ─────────────────                          ─────────────────
 App ──GenerateDataKey──▶ KMS (HSM 内で     App ──Decrypt(包まれたDEK,
      ◀── DEK平文 + DEK暗号文── KEK は不出)        encryption context)──▶ KMS
  │                                              ◀── DEK 平文 ──  (ポリシー評価+監査)
  ├─ DEK で AES-GCM 暗号化 (ローカル・高速)    │
  ├─ DEK 平文を即座にメモリから消去            └─ DEK でローカル復号
  └─ 保存: [暗号文 ‖ DEK暗号文 ‖ context]
                                             認可判定は KMS 呼び出しに集中。
                                             バルク暗号はローカルに分散。
```

### Nitro Enclaves — 隔離と attestation の経路

```
 ┌─ EC2 インスタンス ────────────────────────────────────────┐
 │  親プロセス (ネットワークあり・信頼しない)                    │
 │     │ vsock のみ (唯一の通信路)                             │
 │  ┌──▼────────── Enclave ─────────────┐                    │
 │  │ 割かれた vCPU + メモリ (親から不可視) │                    │
 │  │ ネットワークなし・ディスクなし・SSHなし │                   │
 │  │ ① 起動時: Nitro が PCR を計測        │                   │
 │  └──┬────────────────────────────────┘                    │
 └─────┼──────────────────────────────────────────────────────┘
       │ ② attestation document (PCR + enclave 公開鍵, AWS 署名)
       ▼
     KMS: ③ ポリシー条件 kms:RecipientAttestation:PCR0 == <既知ハッシュ> を検査
          ④ 復号結果を enclave 公開鍵で暗号化して返す
             → 経路上の親インスタンスにも平文は見えない
   「このハッシュのコードとして起動した実行体にだけ、この鍵を使わせる」
```

## 技術詳細 (Technical Details)

### 1. KMS の鍵階層と HSM ドメイン

KMS の暗号設計文書 (ベンダー一次資料) によれば、リージョンの HSM 群は**ドメイン**を成し、ドメイン鍵で各 KMS キーの鍵素材を包んで耐久ストレージに置く——HSM 自体はステートレスに近く、鍵素材は使用時にドメイン鍵で解かれて HSM メモリ内でのみ平文になる。ローテーションは「鍵素材の世代追加」であり、古い世代は復号のために保持される (暗号文の再暗号化は強制されない)。マルチリージョンキー ([`multi-region-global-architecture.md`](multi-region-global-architecture.md)) は同一鍵素材を別リージョンのドメインへ複製する明示的な例外であり、既定では鍵素材はリージョンを出ない。この階層は「上位の鍵ほど露出回数が少なく、寿命が長く、守りが厚い」という NIST SP 800-57 の原則の実装である——DEK は 1 オブジェクトごと、KMS キーは呼び出しごとに HSM 内、ドメイン鍵は HSM 群の外に平文で出ない。

### 2. Encryption context と AAD の意味論

AES-GCM の AAD は「暗号化はしないが認証タグには含める」入力であり、KMS はここに encryption context を正規化して投入する。設計上の要点は 3 つ。(a) **束縛**: 同じ DEK 暗号文でも、異なる context では復号できない——S3 ならオブジェクトの ARN、EBS ならボリューム ID が焼き込まれ、暗号文の「持ち場」が固定される。(b) **認可条件**: IAM ポリシーは `kms:EncryptionContext:<key>` 条件で「この context のときだけ Decrypt を許す」と書ける——復号の認可がデータの論理的帰属と結び付く。(c) **監査**: CloudTrail に context が平文で記録されるため、「どのボリュームのための復号か」が監査ログの語彙になる (context に秘密を入れてはならない理由でもある)。暗号の完全性・認可・監査の 3 平面が 1 つのキー値組で貫通する、小さいが模範的な設計である。

### 3. データ鍵キャッシュと呼び出し経済

KMS 呼び出しはネットワーク往復と課金を伴うため、高頻度の暗号化には DEK の再利用・キャッシュが要る (AWS Encryption SDK のキーキャッシュ等)。ここには明示的なトレードオフがある：DEK を使い回すほど呼び出しは減るが、1 つの DEK が守る範囲 (爆発半径) と平文鍵の滞在時間が延びる。上限 (メッセージ数・バイト数・寿命) を設けたキャッシュはこの交換の定量管理であり、[`cloud-iam-authorization.md`](cloud-iam-authorization.md) の「失効は期限で保証する」と同じ構図——**即時性を諦めて上限で縛る**——が鍵の粒度で再演される。なお AES-GCM は同一鍵での nonce 再利用が破滅的であるため、DEK 再利用の実装は nonce 管理と使用回数上限を暗号学的要件として持つ (SDK はこれを強制する)。

### 4. Enclave のライフサイクルと通信

Enclave イメージ (EIF) はコンテナイメージから変換され、変換時に PCR (PCR0: イメージ全体、PCR1: カーネル+ブート、PCR2: アプリケーション) が決定的に計算される——「どのハッシュを KMS ポリシーに書くか」はビルド時に分かる。実行時は親インスタンスの Nitro CLI が起動を依頼し、Nitro ハイパーバイザが資源を割いて隔離を確立する。通信は vsock 上にアプリが自分でプロトコルを組む (TLS を張るのが定石——親インスタンスは経路として信頼しない)。外部 API (KMS 等) への到達は親側のプロキシ経由になるが、ペイロードは enclave 内で暗号化されてから通るため、プロキシは盲目の配達人である。この「親を配達人扱いする」構造は、[`../../hardware/security/side-channels-tee-secure-boot.md`](../../hardware/security/side-channels-tee-secure-boot.md) の TEE 一般論——ホストを信頼せず、暗号と証明で経路を作る——の実装形である。

### 5. 用途の型 — 何を enclave に入れるか

実用の型は「**秘密に触れる最小の演算だけを切り出す**」に収束する：(a) 秘密鍵の保管と署名 (エンクレーブ内の署名オラクル——鍵は親にも出ない)、(b) 個人情報のトークン化・突合 (平文 PII に触れるのは enclave 内だけ、結果は仮名化 ID)、(c) マルチパーティ計算 (相互不信の当事者が、検証済みコードにだけデータを渡す——attestation が「第三者の中立サーバ」を代替する)。逆に、アプリ全体を enclave に入れる設計は、vsock 境界の複雑化と TCB (信頼計算基盤) の肥大でこの機構の利点を食い潰す。「enclave は金庫であって実行環境ではない」という運用感覚が、機能削除による安全性 (発見事項 5) を保つ側の規律である。

## セキュリティ考察 (Security Considerations)

- **KMS の最重要リスクは鍵ではなく認可である。** `kms:Decrypt` を広く許すポリシー、`kms:*` を持つ管理ロール、キーポリシーの `"Principal": "*"`——実際の漏えいは HSM の破りではなく、この種の誤設定と資格情報窃取 ([`cloud-iam-authorization.md`](cloud-iam-authorization.md) の Capital One 型連鎖) から起きる。防御は同文書の結論と同じ：機械検証 (公開・過剰権限の継続監査)、ガードレール (SCP でキー削除・ポリシー変更を制限)、そして encryption context 条件による復号の文脈束縛である。
- **鍵削除は最強かつ最も危険な操作である。** crypto-shredding の裏面として、KMS キーの削除は配下の全暗号文を不可逆に無効化する——誤操作・悪意の削除はランサムウェアと同型の被害になる。削除の強制待機期間 (7–30 日)、削除権限の分離、および「鍵の無効化 (可逆) と削除 (不可逆) の使い分け」が運用の要になる。[`multi-region-global-architecture.md`](multi-region-global-architecture.md) の隔離バックアップの議論は、バックアップの鍵を本番と分けて初めて完結する。
- **Attestation は「ビルドの信頼」に依存する。** PCR で束縛できるのは「起動したコードのハッシュ」までであり、そのハッシュのコードが安全かどうかは別問題である——依存ライブラリの汚染や悪意あるビルドは attestation を素通りする。機密計算の保証は [`../security/software-supply-chain-security.md`](../security/software-supply-chain-security.md) のビルド来歴 (provenance) と再現可能ビルドに接続して初めて端から端まで繋がる。「何のコードか」を証明する機構は、「そのコードは信頼に値するか」を証明しない。
- **サイドチャネルは TEE の歴史的弱点である。** SGX に対する投機実行系攻撃の系譜が示す通り、マイクロアーキテクチャ状態の共有は論理的隔離を透過しうる ([`../../hardware/security/side-channels-tee-secure-boot.md`](../../hardware/security/side-channels-tee-secure-boot.md))。Nitro Enclaves は専有 vCPU (コア単位の割当) と最小ファームウェアで面を狭めているが、「同一物理ホスト上の共有資源」という前提条件そのものは消えない。高保証が要る用途では、専有ハードウェア (dedicated host, CloudHSM) との使い分けが最終的な緩和である——ここでも密度 (経済性) と隔離の交換 ([`../virtualization/hardware-virtualization-containers.md`](../virtualization/hardware-virtualization-containers.md)) が最後に現れる。

## 結論 (Conclusions)

- KMS の設計は一行に要約できる：**鍵の問題を認可の問題に変換せよ**。鍵は HSM から出さず、演算を API として売り、全呼び出しを IAM と監査に通す——これにより暗号の運用が、クラウドが最も得意とする「認可・監査・失効の機械」([`cloud-iam-authorization.md`](cloud-iam-authorization.md)) に載る。エンベロープ暗号化と encryption context は、この変換を性能と意味論の両面で成立させる補助線である。
- 機密計算は認可の主語を「誰」から「何のコード」へ拡張する。attestation を KMS ポリシーの条件に落とした点が Nitro Enclaves の設計上の要諦であり、秘密が人・役割ではなく検証済みコードに紐づく世界は、最小権限の論理的な到達点である。
- ただし保証の等級は冷静に読む必要がある。Nitro Enclaves が信頼から外すのはインスタンス側の敵であり、事業者は設計・組織統制という等級で信頼に残る。事業者すら外す野心を持った SGX 系は実装の頑健さで苦しんだ。「誰を・どの等級の保証で外すか」を明示することが、機密計算を評価する唯一のまともな方法である。
- 本領域を貫く原理がここでも反復される：機能の削除が安全を買い (enclave の無機能性)、間接層が柔軟性を買い (鍵階層)、暗黙の文脈の明示化が事故を殺す (encryption context, attestation)。クラウドの設計パターンは少数であり、暗号と信頼の層でも同じ道具が使われている。

### 次の接続先 (Related Topics)

- [`cloud-iam-authorization.md`](cloud-iam-authorization.md) — 認可・監査・失効の機械。KMS が鍵の問題を還元する先。
- [`../../hardware/security/side-channels-tee-secure-boot.md`](../../hardware/security/side-channels-tee-secure-boot.md) — TEE のハードウェア機構・SGX の系譜・サイドチャネル史。
- [`../security/software-supply-chain-security.md`](../security/software-supply-chain-security.md) — attestation の上流にあるビルド来歴の信頼。
- [`multi-region-global-architecture.md`](multi-region-global-architecture.md) — マルチリージョンキーと主権要件、隔離バックアップ。
- [`../apis/authentication-tokens-specification.md`](../apis/authentication-tokens-specification.md) — 「秘密の配布からトークン発行へ」という同型の系譜のトークン側。

## 参考文献 (References)

**規範・標準 (Standards)**

- NIST (2020). *SP 800-57 Part 1 Rev. 5: Recommendation for Key Management*. DOI: 10.6028/NIST.SP.800-57pt1r5 — 鍵分離・寿命・危殆化対応の規範。
- NIST (2007). *SP 800-38D: Galois/Counter Mode (GCM) and GMAC*. DOI: 10.6028/NIST.SP.800-38D — AAD の意味論と nonce 一意性要件。

**ベンダー一次資料 (Vendor — 自社設計の規範記述)**

- AWS. *AWS Key Management Service Cryptographic Details*. [docs.aws.amazon.com/kms/latest/cryptographic-details/](https://docs.aws.amazon.com/kms/latest/cryptographic-details/intro.html) — Accessed: 2026-07-30。HSM ドメイン・鍵階層・ローテーションの一次記述。
- AWS. *The Security Design of the AWS Nitro System* (whitepaper). [docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/](https://docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/security-design-of-aws-nitro-system.html) — Accessed: 2026-07-30。Nitro の隔離設計と「運用者アクセス排除」の主張。
- AWS 文書: *Nitro Enclaves attestation process*; *KMS condition keys for Nitro Enclaves*. [docs.aws.amazon.com/enclaves/](https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave-concepts.html) — Accessed: 2026-07-30。PCR・attestation document・`kms:RecipientAttestation` の規範記述。

**独立評価・学術 (Independent / Academic)**

- NCC Group (2023). *Public Report: AWS Nitro System API & Security Claims*. [nccgroup.com](https://www.nccgroup.com/us/research-blog/public-report-aws-nitro-system-api-security-claims/) — Accessed: 2026-07-30。AWS 委託による独立機関の設計主張評価 (委託関係を明示した上で参照)。
- Costan, V., & Devadas, S. (2016). Intel SGX Explained. *IACR Cryptology ePrint Archive*, 2016/086. — 事業者を信頼から外す側の TEE の詳細な学術分析。SGX 系の脅威モデルとの対比軸。
