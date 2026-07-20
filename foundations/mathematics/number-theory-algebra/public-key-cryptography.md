# 公開鍵暗号の数学 (Mathematics of Public-Key Cryptography)

## 概要 (Overview)

数論・代数シリーズの第3回・最終回。前2回で組み立てた道具——高速冪乗、フェルマー・オイラーの定理、巡回群と離散対数、有限体——が、ここで一つのアイデアに合流する: **順方向は対数時間で計算でき、逆方向は（知る限り）事実上計算できない関数**の上に、「会ったことのない相手と、盗聴者の前で秘密を共有する」機構を建てる。Diffie–Hellman 鍵交換、RSA、楕円曲線暗号は、この一つのアイデアの3つの実装である。

本回は各方式を「なぜ正しいか（代数）」と「なぜ破れないと信じるか（計算量的仮定）」に分けて読む。後者はあくまで**仮定**である——安全性は証明ではなく「最良の既知攻撃のコスト」で測られ、そのコストはアルゴリズムの進歩と計算機の進歩で動く。鍵長の推移、楕円曲線への移行、そして Shor のアルゴリズムが強制した耐量子暗号への移行は、すべて「仮定の地盤が動いた」歴史として読める。数学は不変だが、**その数学に賭けてよい額は時代の関数**である。

## 前提知識 (Prerequisites)

- [`modular-arithmetic-groups.md`](modular-arithmetic-groups.md) — 冪乗の高速計算、フェルマー・オイラーの定理、巡回群・生成元・離散対数
- [`finite-fields-coding.md`](finite-fields-coding.md) — 有限体 GF(p)・GF(2ⁿ)。楕円曲線は有限体の上に定義される

## 本論 (Main Content)

### 1. 一方向性と計算量的安全性 (one-wayness and computational security)

公開鍵暗号の安全性は情報理論的（無限の計算でも破れない）ではなく**計算量的**である: 「破るには最良の既知アルゴリズムで 2¹²⁸ 回相当の演算が要る」という形の主張であり、根拠は数十年にわたり世界中が改良に失敗し続けているという経験である。使われる困難問題は主に2つ:

```text
素因数分解:  n = pq から p, q を求める        (RSA の地盤)
離散対数:    g, gˣ から x を求める            (DH・DSA・楕円曲線の地盤)
```

どちらも「作るのは簡単・戻すのは困難」という非対称を持つ。**128 ビット安全性**（2¹²⁸ 相当）が現在の標準的な目標水準であり、各方式の鍵長はこの水準から逆算される (§4 の表)。

### 2. Diffie–Hellman 鍵交換 (Diffie–Hellman key exchange)

巡回群の可換性だけから鍵共有が出る (Diffie & Hellman 1976):

```text
公開: 群と生成元 g。 Alice: 秘密 a を選び gᵃ を送る。 Bob: 秘密 b を選び gᵇ を送る
双方: (gᵇ)ᵃ = g^(ab) = (gᵃ)ᵇ — 同じ値に到達。盗聴者が見るのは g, gᵃ, gᵇ のみ
```

盗聴者が g^(ab) を得るには離散対数（またはそれと同等と信じられる DH 問題）を解く必要がある。2つの運用上の含意が重要である。第一に、DH は**認証を提供しない**——中間者が両側と別々に鍵交換すれば成立してしまうため、署名（§3, §4)による相手認証と常に組で使われる。TLS のハンドシェイクの本質は「署名で認証された DH」である。第二に、指数 a, b を接続ごとに使い捨てる **ephemeral DH** は、後日サーバの長期鍵が漏れても過去の通信が復号できない**前方秘匿性 (forward secrecy)** を与える——「何を秘密に、どれだけの期間依存するか」の設計であり、TLS 1.3 が静的 RSA 鍵交換を廃止して ephemeral DH に一本化した理由である。

### 3. RSA — オイラーの定理の直接の応用 (RSA)

鍵生成: 大素数 p, q（Miller–Rabin で生成、第1回 §4)を取り n = pq、φ(n) = (p−1)(q−1)。公開指数 e（gcd(e, φ(n)) = 1）に対し、秘密指数 d を **e·d ≡ 1 (mod φ(n))** となるよう拡張互除法で求める (Rivest, Shamir & Adleman 1978)。

```text
暗号化/署名検証:  c = mᵉ mod n         (公開鍵で冪乗)
復号/署名生成:    m = cᵈ = m^(ed) = m^(1+kφ(n)) ≡ m (mod n)  (オイラーの定理)
```

正しさの全体がオイラーの定理 1 行に載っている。安全性は「n から φ(n) を求めることは n の素因数分解と等価」という事実に依る——**公開情報 (n, e) と秘密 (p, q) を隔てる壁が素因数分解**である。ただし教科書どおりの RSA は決定的（同じ平文が同じ暗号文になる）で構造的な攻撃（小さい指数、平文の代数関係の悪用）を許すため、実運用は乱択パディング（暗号化は OAEP、署名は PSS）を必須とする——**数学的な芯と、それを安全に包む工学**の分離である。素因数分解の最良法（一般数体ふるい）は準指数時間で、2020 年に 829 ビット (RSA-250) が分解された——2048 ビット以上という現行推奨は、この進捗曲線からの安全マージンである。

### 4. 楕円曲線 — より硬い群への引っ越し (elliptic curves)

有限体上の曲線 y² = x³ + ax + b の点に、「2点を通る直線が曲がりと交わる第3の点」から**点の加法**を定義すると、点全体（＋無限遠点）が可換群になる (Koblitz 1987; Miller 1985)。スカラー倍 kP（P を k 回加える）が冪乗の役を演じ、**楕円曲線離散対数問題 (ECDLP)**——P と kP から k を求める——が困難問題になる。

引っ越しの動機は効率である。mod p の離散対数には指数計算法 (index calculus) という準指数攻撃があるが、楕円曲線群にはこの種の構造的攻撃が知られておらず、最良攻撃が汎用の O(√群位数) に留まる。その結果:

```text
128 ビット安全性の鍵長:  RSA/古典 DH ≈ 3072 ビット  ↔  楕円曲線 ≈ 256 ビット
→ 署名・鍵交換が桁違いに小さく速い。組み込み・モバイル・大規模 TLS の既定へ
```

X25519 (鍵交換)・Ed25519/ECDSA (署名) が現代の主力である。運用の急所は**乱数**にある——ECDSA は署名ごとに一意な乱数 (nonce) を要求し、同じ nonce を2回使うと2本の署名から秘密鍵が連立一次方程式で解けてしまう（PlayStation 3 の署名鍵漏えいはこの初歩的事故である）。決定論的 nonce 生成 (RFC 6979) や、設計段階で事故を防ぐ Ed25519 への移行が対策である——**群の硬さは、群の外(乱数・実装)の弱さを救わない**。

### 5. Shor のアルゴリズムと耐量子移行 (quantum threat and post-quantum migration)

Shor (1997) は、十分大規模な量子計算機が素因数分解と離散対数（楕円曲線を含む）を**多項式時間**で解くことを示した——本シリーズが積み上げた困難性の仮定は、量子計算モデルの上では両方とも崩れる。共通鍵暗号への影響は Grover による全数探索の平方根加速に留まる（鍵長を倍にすれば足りる）ため、危機は公開鍵側に集中する。

移行を急がせるのは **harvest-now, decrypt-later**——今日録画された暗号文が、将来の量子計算機で遡って復号される——であり、長期秘匿を要するデータは既に脅威下にある。標準の応答が**耐量子暗号 (post-quantum cryptography)** で、格子(構造付き格子上の最短ベクトル問題)など量子アルゴリズムの効かない(と信じられる)困難問題への地盤の掛け替えである。NIST は 2024 年に ML-KEM (FIPS 203, 鍵交換)・ML-DSA (FIPS 204, 署名) を標準化し、TLS では従来の X25519 と ML-KEM を**併用するハイブリッド鍵交換**——どちらか一方でも硬ければ安全——が 2026 年現在、主要ブラウザ・CDN で展開が進む。「仮定の地盤が動いたら、複数の地盤に同時に立って渡る」という、計算量的安全性の運用哲学の実演である。

## 工学への接続 (Engineering Connections)

- **TLS ハンドシェイクの正体** (§2–4): 「証明書チェーンの署名検証で相手を認証し、ephemeral (EC)DH で前方秘匿性つきのセッション鍵を作る」——本回の部品の直列である。暗号スイートの選択とは、この各部品をどの群・どの署名で実装するかの選択にほかならない。
- **セキュアブートとコード署名** (§3–4): ブートチェーン・パッケージ・コンテナイメージの署名検証は、ハードウェアに焼かれたルート公開鍵を信頼の錨とする署名の連鎖である。署名鍵の保護 (HSM) と失効設計が、数学の外側の本体になる。
- **実装攻撃との境界** (§3–4): 秘密指数・nonce・スカラー倍は、タイミング・電力・故障で漏れる（第1回 §3–4）。定数時間スカラー倍・ブラインディング・検算は、「代数的に正しい」を「物理的に安全」へ翻訳する層である。
- **鍵長と性能の予算** (§4): 署名サイズ・ハンドシェイク往復のバイト数・検証 CPU は、証明書チェーン全体で積み上がる。楕円曲線への移行と PQC ハイブリッドの追加バイトは、レイテンシ予算の中の実費として現れる。
- **暗号アジリティ** (§5): プロトコル・フォーマットに「アルゴリズムを差し替える継ぎ目」を最初から設計しておくこと——PQC 移行の速度は、数学より継ぎ目の有無で決まっている。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`software/networking/tls-internals.md`](../../../software/networking/tls-internals.md) | 署名で認証された ephemeral (EC)DH、前方秘匿性、PQC ハイブリッド鍵交換 (§2, §4–5) |
| [`hardware/peripherals-connectivity/bluetooth-pairing-protocol.md`](../../../hardware/peripherals-connectivity/bluetooth-pairing-protocol.md) | ペアリングの ECDH と中間者対策（認証つき鍵交換の必然性） (§2, §4) |
| [`hardware/hardware-security/side-channels-tee-secure-boot.md`](../../../hardware/hardware-security/side-channels-tee-secure-boot.md) | セキュアブートの署名連鎖、スカラー倍・冪乗のサイドチャネルと対策 (§3–4) |
| [`software/security/software-supply-chain-security.md`](../../../software/security/software-supply-chain-security.md) | アーティファクト署名・検証の数学的土台と鍵管理 (§3–4) |
| [`software/specifications/authentication-tokens-specification.md`](../../../software/specifications/authentication-tokens-specification.md) | RS256/ES256 署名の代数、nonce 事故と決定論的署名 (§3–4) |
| [`hardware/automotive/cybersecurity-iso21434.md`](../../../hardware/automotive/cybersecurity-iso21434.md) | 車載での署名検証・鍵管理と、長寿命製品ゆえの PQC 移行圧力 (§5) |

## 参考文献 (References)

- Diffie, W., & Hellman, M. (1976). New Directions in Cryptography. *IEEE Transactions on Information Theory*, 22(6), 644–654. DOI: 10.1109/TIT.1976.1055638 — 公開鍵の概念と鍵交換の原典
- Rivest, R. L., Shamir, A., & Adleman, L. (1978). A Method for Obtaining Digital Signatures and Public-Key Cryptosystems. *Communications of the ACM*, 21(2), 120–126. DOI: 10.1145/359340.359342 — RSA の原典
- Koblitz, N. (1987). Elliptic Curve Cryptosystems. *Mathematics of Computation*, 48(177), 203–209. DOI: 10.1090/S0025-5718-1987-0866109-5 — 楕円曲線暗号の原典（Miller 1985 と独立同時）
- Shor, P. W. (1997). Polynomial-Time Algorithms for Prime Factorization and Discrete Logarithms on a Quantum Computer. *SIAM Journal on Computing*, 26(5), 1484–1509. DOI: 10.1137/S0097539795293172 — 量子多項式時間攻撃の原典
- Katz, J., & Lindell, Y. (2020). *Introduction to Modern Cryptography* (3rd ed.). CRC Press. — 計算量的安全性の定義体系の主参照
- [NIST FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard](https://csrc.nist.gov/pubs/fips/203/final) — Accessed: 2026-07-20。ML-KEM の標準文書
