# 有限体と符号の代数 (Finite Fields and the Algebra of Coding)

## 概要 (Overview)

数論・代数シリーズの第2回。前回、素数 p を法とする余りの世界 Z/pZ が四則の揃った**体 (field)** になることを見た。本回はこれを一般化し、**任意の素数冪 q = pⁿ に対して、ちょうど q 個の元を持つ体 GF(q) がただ一つ存在する**ことを構成的に示す。特に GF(2⁸)（1 バイトを1つの「数」として四則演算する体）が主役である。

なぜ工学がこの抽象代数を必要とするのか。答えは「**有限個の値の上で、厳密な線形代数がしたい**」からである。実数の上の計算は丸め誤差を伴うが、有限体の演算は厳密であり、しかも GF(2ⁿ) では加算が XOR（キャリーのない、ハードウェアで最速の演算）になる。CRC は多項式の割り算の余りであり、SerDes のスクランブラ (LFSR) は多項式環での「x を掛ける」操作であり、AES の S-box は GF(2⁸) の逆元であり、Reed–Solomon 符号は有限体上の多項式補間である。バイト列を騒がしく処理して見える機構の内側は、驚くほど整然とした一つの代数である。

## 前提知識 (Prerequisites)

- [`modular-arithmetic-groups.md`](modular-arithmetic-groups.md) — 合同算術、Z/pZ が体であること、群と巡回群
- [`../linear-algebra/vectors-matrices-linear-maps.md`](../linear-algebra/vectors-matrices-linear-maps.md) — §5 で有限体上の行列・連立方程式・補間を使う

## 本論 (Main Content)

### 1. 環と体 — 演算2つの世界 (rings and fields)

**環 (ring)** は加減乗が定義され分配律が成り立つ構造（整数 Z、多項式、n×n 行列）、**体 (field)** はさらに 0 以外の全元で割り算ができる構造である。前回の結果はこの言葉で「Z/nZ は常に環であり、n が素数のとき体になる」と言い直せる。

体であることの価値は、**線形代数一式がそのまま使える**ことにある。連立一次方程式の解法、行列の階数、基底、多項式補間は、係数が「割れる」ことだけに依存しており、実数でも複素数でも有限体でも同じ定理が成り立つ。有限体上の線形代数は、丸め誤差の存在しない厳密な線形代数である。この一点が、誤り訂正符号が実数でなく有限体の上に建つ理由である（[`../linear-algebra/matrix-factorizations-numerical.md`](../linear-algebra/matrix-factorizations-numerical.md) が扱う数値誤差の問題系が、最初から存在しない）。

### 2. 多項式環 — 整数ともう一つの「整除の世界」 (polynomial rings)

体 F 上の多項式全体 F[x] は、整数と驚くほど並行した構造を持つ。

```text
整数 Z              多項式環 F[x]
割り算と余り    ↔    次数についての割り算と余り
素数           ↔    既約多項式 (これ以上因数分解できない)
gcd と互除法    ↔    そのまま成立
mod n の世界    ↔    mod g(x) の世界 (次数 deg g 未満の余りたち)
```

もう一つの基本定理が**補間 (interpolation)** である。次数 k−1 以下の多項式は、相異なる k 点での値から一意に復元できる（Lagrange 補間）。「k 個の係数」と「k 点での値」が等価な表現である。**この冗長性のない等価が、次数を保ったまま評価点を増やすことで冗長性に変わる**（§5 の Reed–Solomon）。秘密分散 (Shamir) も同じ定理の応用で、閾値 k の秘密を「次数 k−1 の多項式上の点」として配る。

### 3. 有限体の構成 — GF(pⁿ) (construction of finite fields)

素数位数の体 GF(p) = Z/pZ は前回既に手にした。位数 pⁿ の体は、**整数から Z/pZ を作ったのと同じ手を多項式で繰り返す**ことで得られる。

```text
GF(pⁿ) = GF(p)[x] を n 次の既約多項式 g(x) で割った余りの世界
       = 次数 n 未満の多項式 (係数は GF(p)) に、mod g(x) の乗算を入れたもの
```

既約性が §1 の「素数を法に取る」に対応し、零因子を消して割り算を可能にする。基本定理: **有限体の位数は必ず素数冪であり、各位数 q = pⁿ に対し GF(q) は（同型を除いて）一意に存在する**。さらに乗法群 GF(q)^× は必ず巡回群である (Lidl & Niederreiter 1997, Ch. 2)。これにより、前回の生成元・離散対数の議論が有限体一般に持ち上がる。

**GF(2⁸) を具体的に**: 元は 8 ビット＝バイトそのもの（ビット i が xⁱ の係数）。加算は係数ごとの mod 2 加算＝**XOR**（キャリーが無い！）、乗算は多項式の掛け算をして 8 次の既約多項式で余りを取る。「バイト同士を足しても桁上がりで隣に波及しない」算術は、ハードウェアでは XOR ゲートの並列、ソフトウェアではテーブル参照や SIMD のキャリーレス乗算命令で実装される。

### 4. GF(2) の計算工学 — CRC・LFSR・AES (engineering over GF(2))

- **CRC は多項式の余りである**: ビット列をGF(2) 上の多項式と見なし、生成多項式 g(x) で割った余りがチェックサムになる (Peterson & Brown 1961)。バーストエラー検出能力は g(x) の代数的性質（次数、原始性、(x+1) 因子の有無）から**証明として**導かれる。つまり「経験的に良いハッシュ」ではなく、検出保証つきの設計である。ハードウェア実装はシフトレジスタと XOR のみで、1 クロック 1 ビット（並列化で 1 クロック複数バイト）で流れる。
- **LFSR は mod g(x) の「x 倍」である**: 線形帰還シフトレジスタの 1 ステップは、多項式環 GF(2)[x]/(g) で x を掛ける操作に等しい。g が**原始多項式**（根が乗法群の生成元）なら状態は 2ⁿ−1 周期で全非零状態を巡る。これが最長周期系列 (m-系列) であり、SerDes のスクランブラ・PRBS テストパターン・スペクトラム拡散符号の数学的実体である。「ほぼ乱数に見えるが完全に決定的で、同じ回路で解除できる」性質は、巡回群の言葉でそのまま説明される。
- **AES は GF(2⁸) の上に書かれている**: S-box の中核は GF(2⁸) での逆元（非線形性の供給源）、MixColumns は GF(2⁸) 係数の 4×4 行列乗算（拡散の供給源）である (Daemen & Rijmen 2002)。「テーブルの塊」に見える暗号が、実は 1 つの体の算術で閉じている。代数的構造は解析（安全性の議論）と実装（ビットスライス、AES-NI 命令）の両方の言語になっている。

### 5. Reed–Solomon 符号 — 補間の冗長化 (Reed–Solomon codes)

§2 の補間定理を冗長化の道具に変える。k 個のデータシンボル（GF(2⁸) の元＝バイト）を次数 k−1 の多項式 f の係数と見なし、**n > k 個の点で評価した値**を符号語として送る (Reed & Solomon 1960)。

```text
符号化:  (c₀,…,c_{k−1}) → f(α₁), f(α₂), …, f(α_n)   [n−k 個の冗長]
復元:    どの k 個の値からでも f を補間で一意復元
        ⇒ n−k 個までの消失 (erasure) に耐える — この冗長量での理論最良 (MDS)
```

位置不明の誤り (error) に対しても (n−k)/2 個まで訂正でき、復号は有限体上の連立方程式（シンドローム・Berlekamp–Massey 法）に帰着する。§1 で述べた「厳密な線形代数」がここで全面稼働する。消失訂正の顔がオブジェクトストレージのイレイジャーコーディング RS(k, m)、誤り訂正の顔が CD/DVD・QR コード・DVB であり、同じ1つの符号の両面である。NAND の主流 ECC である BCH・LDPC も GF(2ⁿ) 上の多項式・線形代数の同族で、媒体の誤り特性（ランダムビット誤り）に合わせて族の中の別の点を選んだものと読める。

## 工学への接続 (Engineering Connections)

- **XOR の代数的地位** (§3): パリティ・RAID の XOR 復元・ワンタイムパッドは、すべて「GF(2) の加算は自分自身が逆演算」という1つの事実の応用である。
- **検出保証の設計** (§4): CRC の生成多項式選択は、想定する誤りモデル（バースト長・ビット誤り率）に対する検出保証を代数から逆算する作業であり、Ethernet/PCIe/NVMe の各規格が固有の g(x) を規定する理由である。
- **DC バランスとクロック回復** (§4): スクランブラは信号のスペクトルを白色化し、遷移密度を保証する。高速シリアルリンクの物理要件を、原始多項式の最長周期性が支えている。
- **ストレージ耐久性の通貨** (§5): RS(k, m) の「任意の k 断片から復元」という MDS 性が、複製よりも容量効率の良い耐久性を可能にする。修復には k 断片の読み出しが要る、という修復増幅もまた補間の式形から直接読み取れる。
- **SIMD・専用命令との親和** (§3–4): キャリーレス乗算 (PCLMULQDQ)・AES-NI・GF-NI と、テーブル参照のベクトル化。有限体演算は現代 ISA が専用命令を持つ程度に頻出の内側ループである。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`software/object-storage/object-storage-architecture.md`](../../../software/object-storage/object-storage-architecture.md) | RS(k,m) イレイジャーコーディングの数学的本体＝GF(2⁸) 上の補間と MDS 性 (§2, §5) |
| [`hardware/storage/nand-flash-ftl.md`](../../../hardware/storage/nand-flash-ftl.md) | BCH/LDPC による ECC——有限体上の符号族を媒体誤り特性へ適合させたもの (§5) |
| [`hardware/signal-integrity/serdes-link-architecture.md`](../../../hardware/signal-integrity/serdes-link-architecture.md) | スクランブラ・PRBS＝原始多項式 LFSR の最長周期系列、リンク FEC (§4–5) |
| [`hardware/wireless/radio-fundamentals-modulation.md`](../../../hardware/wireless/radio-fundamentals-modulation.md) | チャネル符号化・拡散符号の代数的基礎 (§4–5) |
| [`software/networking/kernel-network-stack.md`](../../../software/networking/kernel-network-stack.md) | チェックサム・CRC オフロードの計算の実体 (§4) |
| [`foundations/mathematics/probability-statistics/reliability-theory.md`](../probability-statistics/reliability-theory.md) | 「n−k 故障まで耐える」符号が、故障確率モデルと合成されて耐久性になる (§5) |

## 参考文献 (References)

- Lidl, R., & Niederreiter, H. (1997). *Finite Fields* (2nd ed.). Cambridge University Press. DOI: 10.1017/CBO9780511525926 — 有限体の構成・一意性・乗法群の巡回性の主参照
- Reed, I. S., & Solomon, G. (1960). Polynomial Codes over Certain Finite Fields. *Journal of the Society for Industrial and Applied Mathematics*, 8(2), 300–304. DOI: 10.1137/0108018 — Reed–Solomon 符号の原典
- Peterson, W. W., & Brown, D. T. (1961). Cyclic Codes for Error Detection. *Proceedings of the IRE*, 49(1), 228–235. DOI: 10.1109/JRPROC.1961.287814 — CRC の理論的基礎の原典
- MacWilliams, F. J., & Sloane, N. J. A. (1977). *The Theory of Error-Correcting Codes*. North-Holland. — 符号理論の包括的古典
- Daemen, J., & Rijmen, V. (2002). *The Design of Rijndael: AES — The Advanced Encryption Standard*. Springer. DOI: 10.1007/978-3-662-04722-4 — AES の代数的設計の一次資料
