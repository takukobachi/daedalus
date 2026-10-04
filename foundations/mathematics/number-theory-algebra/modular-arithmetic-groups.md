# 合同算術と群 (Modular Arithmetic and Groups)

## 概要 (Overview)

数論・代数シリーズの第1回。計算機の整数は数学の整数ではない——64 ビットレジスタの加算は 2⁶⁴ を法とする**合同算術 (modular arithmetic)** であり、ハッシュテーブルの添字計算も、暗号の鍵交換も、「割った余りの世界」の算術である。本回はこの世界を、割り算の基礎（Euclid の互除法）から積み上げ、フェルマー・オイラーの定理と高速冪乗——**公開鍵暗号を計算可能にしている2つの道具**——まで到達する。

後半では、これらの構造に共通する骨格として**群 (group)** を導入する。群は「演算が1つあり、逆操作が常にできる」ことだけを公理化した構造であり、ラグランジュの定理という1本の定理がフェルマー・オイラーの定理を統一的に説明する。そして巡回群の上で「冪乗は簡単、その逆（離散対数）は難しい」という非対称が現れる——次々回の公開鍵暗号はこの非対称の上に建つ。抽象化は一般化のためではなく、**どの性質が本質かを特定するための道具**として使う。

## 前提知識 (Prerequisites)

- [`../discrete-logic/propositional-boolean-logic.md`](../discrete-logic/propositional-boolean-logic.md) — 集合と論理の記法、有限構造の全数検査という作法
- [`../probability-statistics/probability-random-variables.md`](../probability-statistics/probability-random-variables.md) — §4 の確率的素数判定で誤り確率の議論に使う

## 本論 (Main Content)

### 1. 整除と Euclid の互除法 (divisibility and the Euclidean algorithm)

最大公約数 gcd(a, b) は `gcd(a, b) = gcd(b, a mod b)` の繰り返しで求まる——記録に残る最古のアルゴリズムであり、桁数に対し線形（値に対し対数）回で停止する (Knuth 1997, §4.5.2)。決定的に重要なのは拡張版が返す **Bézout の等式**である。

```text
gcd(a, b) = ax + by  を満たす整数 x, y が存在し、互除法の逆算で求まる
特に gcd(a, n) = 1 なら ax ≡ 1 (mod n) — x が a の「逆数」(モジュラ逆元)
```

割り算のない世界に「掛けて 1 になる相手」を与えるこの構成が、以降のすべて——合同式を「解く」こと、RSA の秘密鍵の計算——の土台になる。

### 2. 合同算術 — 有限の世界の四則 (arithmetic mod n)

a ≡ b (mod n) を「n で割った余りが等しい」と定め、余りの集合 Z/nZ = {0, 1, …, n−1} の上で加減乗を定義する。整合性（代表元の取り方に依らないこと）が成り立つので、**有限集合の上で普通の代数法則（結合・分配・交換）がそのまま使える**。機械整数のラップアラウンドは n = 2⁶⁴ の場合にすぎず、「オーバーフロー」とは mod 2⁶⁴ の算術を無限の整数と誤解した時に見える現象である——符号なし整数の演算が言語仕様上「ラップする」と定義されるのは、この代数の採用宣言である。

割り算だけは特別で、a の逆元は **gcd(a, n) = 1 のときに限り**存在する (§1)。従って n が素数 p なら 0 以外の全元が逆元を持ち、Z/pZ は四則の揃った**体 (field)** になる——有限体の最初の例であり、次回の主役である。逆に n が合成数だと零因子（掛けて 0 になる非零元）が現れる。この差が「素数を法に取る」理由のすべてである。

### 3. 中国剰余定理 (Chinese Remainder Theorem)

n = n₁n₂…n_k（どの2つも互いに素）のとき、次が成り立つ。

```text
x mod n を知ることと、(x mod n₁, …, x mod n_k) を知ることは等価
— 大きな法の算術は、小さな法の算術の並列実行に分解できる
```

対応は両方向とも構成的（逆向きは Bézout から組み立てる）である (Shoup 2009, Ch. 2)。工学的な顔は**計算の分割**である: RSA の復号は法 n = pq の冪乗だが、CRT で mod p と mod q の2つの冪乗に分解すると約4倍速くなる (RSA-CRT)。ただしこの最適化は故障攻撃の名所でもある——片方の枝だけ計算を誤らせると、誤った署名と正しい署名の差から p が漏れる (Bellcore 攻撃)。**数学的に等価な計算経路が、物理的な攻撃面としては等価でない**、という教訓の古典例である。

### 4. フェルマー・オイラーの定理と高速冪乗 (Fermat–Euler and fast exponentiation)

```text
フェルマーの小定理:  p が素数, gcd(a,p)=1 ⇒ a^(p−1) ≡ 1 (mod p)
オイラーの定理:      gcd(a,n)=1 ⇒ a^φ(n) ≡ 1 (mod n)
  φ(n) = n 以下で n と互いに素な数の個数。φ(pq) = (p−1)(q−1)
```

指数の世界が「φ(n) を法として回る」——この周期性が、次々回の RSA の正しさ（e·d ≡ 1 (mod φ(n)) と選べば冪乗が元に戻る）の全根拠である。

計算面の相棒が**二進冪乗法 (square-and-multiply)**: a^e を「二乗を繰り返し、e のビットが 1 の位置でだけ掛ける」ことで、乗算回数を e 回から **O(log e) 回**に落とす。2048 ビットの指数でも数千回の乗算で済む——**公開鍵暗号が実用になるのはこの対数化のおかげ**である。同時に、「ビットが 1 のときだけ余計な乗算をする」という素朴な実装は、実行時間・電力にビット列を漏らす——タイミング攻撃の教科書的な標的であり、定数時間実装（モンゴメリラダー等）が要求される理由である。

素数の入手も冪乗に依る。**Miller–Rabin 判定**は「合成数なら、ランダムな底の少なくとも 3/4 が合成の証拠を出す」ことを使う確率的判定で、k 回の独立試行で誤り確率を 4^(−k) 以下に抑える (Rabin 1980)——巨大素数を「証明抜きで、しかし任意の信頼度で」量産できる。暗号鍵の裏には確率・統計シリーズの誤り確率の議論が常に張り付いている。

### 5. 群 — 構造の公理化とラグランジュの定理 (groups and Lagrange's theorem)

**群**は、集合と演算の組で、結合律・単位元・逆元を満たすものである（可換なら加えて交換律）。(Z/nZ, +)、逆元を持つ余りの積 (Z/nZ)^×、そして次々回の楕円曲線の点——見た目の異なるこれらが同じ公理を満たす。有限群の基本定理が**ラグランジュの定理**である。

```text
有限群 G の任意の元 a の位数 (a^k = e となる最小の k) は |G| を割り切る
⇒ a^|G| = e — フェルマー・オイラーの定理は |(Z/nZ)^×| = φ(n) の特殊例
```

個別に証明した2つの定理が、群の言葉では1行の系になる——抽象化の威力の実演である。1つの元の冪 {g, g², g³, …} が群全体を尽くすとき**巡回群**、g を**生成元**と呼ぶ。素数 p に対し (Z/pZ)^× は必ず巡回群であり、ここに根本的な非対称が現れる: g と x から g^x を計算するのは O(log x) 回の乗算（§4）だが、g と g^x から x を求める**離散対数問題 (discrete logarithm problem)** には、既知の最良アルゴリズムでも指数的に近い時間がかかる。**順方向は対数時間、逆方向は（知る限り）準指数時間**——この片道性が公開鍵暗号の存在根拠であり、第3回の主題である。

## 工学への接続 (Engineering Connections)

- **機械整数の意味論** (§2): 符号なし演算のラップアラウンドは mod 2⁶⁴ の代数であり、「オーバーフローバグ」の多くは無限整数と合同算術の混同である。ハッシュ・チェックサムが加算と XOR を自由に混ぜられるのも、どちらも良い代数の上にいるからである。
- **ハッシュと分配** (§2): `hash(k) mod n` によるバケット分配は合同算術の最頻出応用。n の変更が全対応を崩す(再分配の雪崩)ことが、コンシステントハッシュ法の動機になる。
- **RSA-CRT と故障攻撃** (§3): 4倍の高速化と引き換えに、単一の計算誤りが秘密鍵を漏らす攻撃面が開く。検算(署名を公開鍵で検証してから出力)が標準対策である。
- **定数時間実装** (§4): square-and-multiply の分岐は秘密指数のビットを時間・電力に漏らす。「数学的に正しい」と「実装として安全」の間の距離が、サイドチャネルの章の主題そのものである。
- **鍵生成の確率設計** (§4): Miller–Rabin の反復回数は誤り確率の予算配分であり、「何回試行すれば足りるか」は確率・統計シリーズの言葉で決まる。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`software/networking/tls-internals.md`](../../../software/networking/tls-internals.md) | 鍵交換・署名の下部構造(冪乗・逆元・素数生成)、第3回への伏線 (§1, §4–5) |
| [`hardware/hardware-security/side-channels-tee-secure-boot.md`](../../../hardware/hardware-security/side-channels-tee-secure-boot.md) | square-and-multiply のタイミングリーク、RSA-CRT 故障攻撃 (§3–4) |
| [`software/networking/load-balancing-architecture.md`](../../../software/networking/load-balancing-architecture.md) | `hash mod n` の再分配問題とコンシステントハッシュ法の動機 (§2) |
| [`software/specifications/authentication-tokens-specification.md`](../../../software/specifications/authentication-tokens-specification.md) | 署名検証 (RS256/ES256) が依拠する冪乗と群の算術 (§4–5) |
| [`hardware/cpu-architecture/isa-riscv-intro.md`](../../../hardware/cpu-architecture/isa-riscv-intro.md) | 機械整数演算＝mod 2^XLEN の合同算術という意味論 (§2) |

## 参考文献 (References)

- Shoup, V. (2009). *A Computational Introduction to Number Theory and Algebra* (2nd ed.). Cambridge University Press. — 互除法・CRT・群の計算的な主参照
- Hardy, G. H., & Wright, E. M. (2008). *An Introduction to the Theory of Numbers* (6th ed.). Oxford University Press. — 数論本体の古典的主参照
- Knuth, D. E. (1997). *The Art of Computer Programming, Vol. 2: Seminumerical Algorithms* (3rd ed.). Addison-Wesley. — 互除法・冪乗・乱数の計算面の参照
- Rabin, M. O. (1980). Probabilistic Algorithm for Testing Primality. *Journal of Number Theory*, 12(1), 128–138. DOI: 10.1016/0022-314X(80)90084-0 — 確率的素数判定の原典
- Menezes, A. J., van Oorschot, P. C., & Vanstone, S. A. (1996). *Handbook of Applied Cryptography*. CRC Press. — モジュラ算術アルゴリズム群の実装面の標準参照
