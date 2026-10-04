# 標準ライブラリのデータ構造内部 — dict・map・HashMap と文字列 (Standard Library Internals: Hash Tables & Strings in CPython, Go, Rust)

## 概要 (Overview)

プログラムが最も頻繁に触るデータ構造は、連想配列 (hash table) と文字列である。CPython の `dict`、Go の `map`、Rust の `HashMap`——API はほぼ同じに見えるこの三つが、内部では全く違う機械語とメモリレイアウトになる。本ドキュメントはその内部実装を比較する。[`language-implementations-cpython-go-rust.md`](language-implementations-cpython-go-rust.md) が言語処理系を「意味論の保証をいつ払うか」の軸で比較したのに対し、本書はその**各論としての最深部**——同じ抽象が言語の制約 (boxing・GC・所有権) を映してどう別物になるか——を扱う。

中心テーゼは二つ。第一に、**標準ライブラリのデータ構造は言語設計の縮図である**。CPython の dict が全要素をポインタで持つのは「すべてが PyObject」の帰結であり、Go の map の要素にアドレスを取れないのは GC と漸進的成長の帰結であり、Rust の HashMap のイテレータ無効化がコンパイルエラーなのは借用検査の帰結である——データ構造の「癖」に見えるものは、ほぼすべて言語の中核設計から導出できる。

第二に、**ハードウェアは言語の違いを押し流して同じ答えに収斂させる**。三言語は独立の歴史を持ちながら、ハッシュ表は連鎖法 (chaining) ではなく開番地法 (open addressing) に、そして 2020 年代には Rust (1.36, 2019) と Go (1.24, 2025) が揃って **SwissTable 系** (制御バイト列を SIMD で一括走査する設計) に到達した。キャッシュ階層 ([`../../hardware/cpu/cache-memory-hierarchy.md`](../../hardware/cpu/cache-memory-hierarchy.md)) と SIMD ([`../../hardware/cpu/simd-vector.md`](../../hardware/cpu/simd-vector.md)) という物理が、設計空間の勝者を決めた例である。

本書は (1) ハッシュ表の三実装 (レイアウト・探査・成長)、(2) 反復順序という意味論の分岐 (仕様への昇格 vs 意図的ランダム化)、(3) HashDoS とハッシュ関数の防御、(4) 文字列表現 (「i 番目の文字」への三つの答え)、を CPython/Go/Rust のソースと一次資料を基に整理する。分析深度は Level 3。

## 方法論 (Methodology)

- 三実装を同じ軸——メモリレイアウト (boxing の有無)・衝突戦略・反復順序の意味論・成長と無効化・ハッシュ関数の防御——で貫き、前掲書の「支払時期」テーゼの各論として読めるようにする。
- 実装の記述は各処理系のソース (CPython `Objects/dictobject.c`・`Objects/unicodeobject.c`、Go `runtime/map*.go`、Rust std/hashbrown) と公式設計文書 (PEP, Go blog, Abseil design notes) に基づく。バージョンで変わる事実 (Go 1.24 の map 刷新など) は版を明記する。
- ハッシュ表の一般理論 (計算量・ユニバーサルハッシュ) は前提とし、本書は**実装が理論から逸脱する場所**——キャッシュ・SIMD・GC との相互作用——に集中する。
- 文字列は「表現 (representation)」に集中し、正規表現・検索アルゴリズムは扱わない (将来の別文書候補)。

## 発見事項 (Findings)

1. **boxing の有無がハッシュ表の物理を分ける。** CPython の dict はキーも値も `PyObject*` (8 バイトのポインタ) であり、比較のたびにポインタを追い、`__eq__` のディスパッチを挟む。Go/Rust はキーと値を**表の中にインライン格納**し、比較は機械語の比較命令に落ちる。同じ「ルックアップ」でもメモリアクセスの回数が構造的に違い、キャッシュミスの数が性能差の本体になる——[`language-implementations-cpython-go-rust.md`](language-implementations-cpython-go-rust.md) の「boxing の税」がデータ構造の中で複利になる場所である。
2. **三者は開番地法に収斂し、さらに SwissTable に収斂しつつある。** 連鎖法はノードごとの割り当てとポインタ追跡でキャッシュに敵対的であり、三実装とも採らない。その先で、ハッシュの下位で表を引き上位 7 bit を 1 バイトの「制御バイト」として別配列に持ち、**16 バイトの制御バイト群を SIMD 命令 1 発で照合する** SwissTable (Abseil, 2017 発表) に Rust (hashbrown を 1.36 で std 採用) と Go (1.24 で map を刷新) が到達した。ハッシュ表の設計を決めたのはアルゴリズムの漸近ではなく、キャッシュラインと SIMD というハードウェアである。
3. **CPython の compact dict は「密な記録 + 疎な索引」への分解であり、副産物が言語仕様になった。** 3.6 で dict は「挿入順に詰まったエントリ配列」と「そこへの添字だけを持つ疎なハッシュ索引」に分解され、メモリを 20-25% 節約した。副産物として反復が挿入順になり、3.7 でこれが**言語仕様に昇格**した——実装詳細が観測され、依存され、仕様になるという Hyrum の法則の教科書的事例である。
4. **Go は同じ現象を予見して反復順序を意図的にランダム化した。** Go 1.0 の時点で map の反復開始位置を毎回乱数で変え、「たまたまの順序」に依存するコードを**書いた瞬間に壊れる**ようにした。Python が事後に依存を追認して仕様化したのと正反対の、Hyrum の法則への先制攻撃である。Rust は per-instance の乱数シードにより順序が実行ごとに変わる (未規定のまま)。同じ「未規定の順序」への三つの態度——追認・妨害・放置——が言語の性格を映す。
5. **HashDoS がハッシュ関数を暗号学の領域に押し込んだ。** 攻撃者が衝突するキーを大量に送り込むとハッシュ表は線形リストに退化する (2011 年の hash flooding 攻撃、28C3)。対策として Python はシード付き SipHash (PEP 456)、Rust は既定ハッシュに SipHash-1-3、Go はプロセス乱数シード + ハードウェア AES 命令ベースのハッシュを採用した——**既定で DoS 耐性を持ち、速さが欲しい者だけが明示的に降りる** (Rust の FxHash 等) という安全側デフォルトに三者とも落ち着いた。
6. **「i 番目の文字」への答えが文字列表現を決める。** CPython (PEP 393) は文字列ごとに 1/2/4 バイト固定幅を選んで格納し、O(1) 添字を守る (メモリで買う)。Go は不変の UTF-8 バイト列とし、`s[i]` はバイトを返し、コードポイントが欲しければ `range` で走査する (プログラマに符号化を意識させる)。Rust は UTF-8 を型で強制した上で**整数添字そのものを提供しない** (「i 番目の文字」は Unicode では O(1) で答えられない問い、という立場)。妥協して守る・ずらして見せる・問いごと拒否する——三様の誠実さである。
7. **反復中の変更の扱いは支払時期テーゼの再演である。** dict はサイズ変更を実行時に検出して `RuntimeError` を投げ (実行時検査)、Go はスペックで挙動を定義し (削除済みは出ない・追加分は出るかもしれない——規約で払う)、Rust は反復中の変更を借用検査が**コンパイルエラー**にする。イテレータ無効化という同じ問題への三つの答えが、前掲書の軸にそのまま載る。

## アーキテクチャ (Architecture)

### CPython compact dict — 密な記録と疎な索引

```
 PyDictObject (3.6+)
 ┌────────────────────────────┐   ┌─────────────────────────────────────┐
 │ 索引 (疎, ハッシュで引く)      │   │ エントリ配列 (密, 挿入順)              │
 │ [-, 2, -, 0, -, -, 1, -]     │→ │ 0: (hash, key*, value*)  ← PyObject* │
 │  小さい表なら 1 バイト/枠      │   │ 1: (hash, key*, value*)              │
 │  (表の大きさで 1/2/4/8B)      │   │ 2: (hash, key*, value*)              │
 └────────────────────────────┘   └─────────────────────────────────────┘
   衝突時は perturb 探査: j = 5j + 1 + perturb; perturb >>= 5
   反復はエントリ配列を頭から舐めるだけ → 挿入順 (3.7 から言語仕様)
```

### SwissTable — 制御バイトを SIMD で束ねて引く (Rust hashbrown / Go 1.24)

```
 ハッシュ h を分割: H1 (上位 57bit) = グループ位置、H2 (下位 7bit) = 指紋
 制御バイト列 (16 個 = 1 グループ)          スロット列 (キー・値をインライン格納)
 [E5|80|3A|E5|FF|…|91]  ← 1 バイト/枠     [ (k,v) | (k,v) | … ]
   │ FF = 空, 80 = 墓標, それ以外 = H2
   └ SIMD (SSE2 等) で「H2 == E5 の枠」を 16 枠まとめて 1 命令で照合
     → 候補枠だけ実キーを比較。ほとんどの探査はキーに触れずに終わる
```

- 探査の大半が**制御バイト列 (キャッシュに乗る小さな配列) の中で完結**し、キー本体へのアクセスは「ほぼ確実に当たり」の枠だけ。負荷率 87.5% まで詰められる理由であり、開番地法の弱点 (クラスタリング) を SIMD の並列照合で相殺した設計である。

## 技術詳細 (Technical Details)

### 1. CPython dict — ポインタの表、しかし驚くほど最適化されている

- **すべてが間接参照**: エントリは `(hash, PyObject* key, PyObject* value)`。ハッシュ値をエントリにキャッシュするのは、比較・再ハッシュのたびに `__hash__` を呼び直さないためである (boxing 世界での防衛)。衝突時の等価判定はまずポインタ同一性 (`is`)、次に `__eq__` ディスパッチ——インターン済み文字列キー (変数名・属性名はほぼこれ) はポインタ比較だけで済み、**名前空間としての dict** が実用速度になる。
- **key-sharing dict (PEP 412, 3.3)**: 同じクラスのインスタンスは属性名の集合がほぼ同じ、という観測から、`__dict__` のキー部分をクラスで共有し、インスタンスは値の配列だけを持つ。[`language-implementations-cpython-go-rust.md`](language-implementations-cpython-go-rust.md) の hidden class と同じ「形の共有」の CPython 版であり、3.11+ の属性アクセス特殊化 (`LOAD_ATTR` の適応的最適化) の土台でもある。
- **成長戦略**: 負荷率 2/3 で拡張。削除は墓標 (dummy) を残し、拡張時に一掃する。反復中のサイズ変更は世代カウンタ (`ma_version` 系の検査) ではなく used 数の変化で検出され `RuntimeError` になる——検出は**サイズ変更のみ**で、値の上書きは合法という割り切りである。

### 2. Go map — GC と漸進成長が形を決める

- **旧設計 (〜1.23)**: 8 スロットのバケツ + tophash (ハッシュ上位 8bit の配列) + 溢れバケツの連結。SwissTable と同じ「1 バイトの指紋を先に見る」発想を 2010 年代初頭から持っていたが、照合はスカラループだった。
- **漸進的成長 (incremental evacuation)**: 拡張時に全要素を一括で移さず、以後の書き込みのたびに旧表から新表へ少しずつ「疎開」させる——GC を STW にしないのと同じ思想 ([`managed-runtimes-gc-jit.md`](managed-runtimes-gc-jit.md)) で、**レイテンシの尖りを操作全体に薄く塗る**。要素が移動しうるため `&m[k]` は言語仕様レベルで禁止 (map 要素はアドレス不可) であり、データ構造の実装戦略が言語の表面仕様に浮き上がった例である。
- **Swiss map (1.24, 2025)**: 制御ワード + グループ構造の SwissTable 系に全面刷新し、大きな map を固定サイズの表の集まりに分割して漸進成長を維持したまま、平均 10-60% の高速化 (Go blog、自社計測値) を得た。ハッシュ関数は AES 命令が使える環境では `aeshash` (ハードウェア混合)、プロセスごとの乱数シード付き。
- **要素インライン格納の条件**: キー/値が大きい (128 バイト超) 場合は間接化されるなど、GC のスキャン対象 (ポインタを含むか) で内部表現が分岐する——**GC のある言語では、データ構造のレイアウト設計は常に GC との共同設計**である。

### 3. Rust HashMap — 所有権の上の SwissTable

- **hashbrown (1.36 で std 採用)**: Abseil SwissTable の Rust 移植で、上の図の通り H2 指紋の SIMD 照合が核。キーと値はインライン格納され、`Drop` は表の破棄時に各要素へ決定的に走る。負荷率 7/8。
- **既定ハッシュは SipHash-1-3 + インスタンスごとの乱数シード** (`RandomState`)。暗号学的な頑健性と引き換えに整数キーでは目に見えて遅く、内部ツール等では `FxHashMap` (rustc 自身が使用)・`ahash` への差し替えが定番——**安全な既定 + 明示的なオプトアウト**の設計であり、ベンチマークで「Rust の HashMap が遅い」と言うときはほぼ常にこのハッシュ関数の話である。
- **無効化はコンパイルエラー**: 反復中の挿入・削除は `&mut` の重複としてコンパイル時に拒否される。エントリ API (`entry(k).or_insert(v)`) は「検索 1 回で挿入まで済ませる」ためのインタフェースで、二重検索を型で避ける——実行時検査 (Python)・仕様で定義 (Go) との対比がここでも成立する。

### 4. 文字列表現 — 「i 番目の文字」への三つの答え

- **CPython (PEP 393, 3.3)**: 文字列ごとに、含まれる最大コードポイントに応じて 1 (Latin-1)・2 (UCS-2)・4 (UCS-4) バイト固定幅の表現を選ぶ。ASCII だけの文字列は最小サイズで持ち、どの文字列も `s[i]` は O(1)——「Python の str はコードポイントの列」という抽象を、メモリの可変幅で買っている。代償は、非 ASCII が 1 文字混じると文字列全体が広い表現に膨らむこと、そして C 拡張へ渡す際の変換である。
- **Go**: string は**不変の UTF-8 バイト列** (ポインタ + 長さの 2 ワードヘッダ) であり、`s[i]` はバイト、`range s` はコードポイント (rune) を返す——**同じ値が見る演算子で違う顔をする**ことで符号化の現実を隠さない。`[]byte(s)` / `string(b)` は原則コピー (不変性の防衛) で、コンパイラが安全な場合だけ省く。この「文字列 = ただのバイト列 + 規約」は、I/O とネットワークの言語である Go の割り切りである。
- **Rust**: `String`/`&str` は**妥当な UTF-8 であることを型が保証する**バイト列。`s[i]` は存在せず (整数添字は実装されない)、バイト範囲のスライスは可能だが文字境界を跨ぐと panic する。「i 番目の文字」は Unicode では定義の怪しい問い (コードポイントか、書記素クラスタか) であり、Rust は**嘘の O(1) を提供するより問いを拒否する**——書記素が必要なら unicode-segmentation クレートへ、という明示的な階層化である。
- 三者に共通する正直な限界: どの言語も「文字」の直感 (書記素クラスタ、例えば絵文字の合成列) は標準の添字・長さでは扱えない。`len("👨‍👩‍👧")` が三者三様の値を返すことは、表現の違いがユーザ可視になる最小の実験である。

### 5. 観測の入口 (Observability)

| 疑問 | CPython | Go | Rust |
|---|---|---|---|
| 表・文字列の実メモリ | `sys.getsizeof`, tracemalloc | pprof (heap), `unsafe.Sizeof` | `std::mem::size_of`, heaptrack |
| ハッシュの衝突・退化 | `PYTHONHASHSEED` 固定で再現実験 | ランタイムが隠蔽 (計測は bench で) | ハッシュ関数差し替えで比較 (criterion) |
| レイアウトの確認 | `dis` + `object.__basicsize__` | `GOEXPERIMENT` 世代の差は go version で | `cargo asm`, miri (UB 検査) |
| 文字列表現の実態 | `sys.getsizeof` の段差 (1/2/4B kind) | `len(s)` vs `utf8.RuneCountInString` | `s.len()` (バイト) vs `chars().count()` |

## セキュリティ考察 (Security Considerations)

- **HashDoS は「アルゴリズムの平均計算量」を信頼境界の問題に変えた。** 攻撃者が入力キーを選べる場所 (HTTP ヘッダ・JSON のキー・フォームフィールド) では、ハッシュ表の O(1) は**敵対的入力の下では成立しない**。2011 年の hash flooding (28C3) は主要 Web 言語を一斉に直撃し、シード付き・短入力向け PRF である SipHash (Aumasson & Bernstein, 2012) が事実上の業界標準になった。教訓は一般化できる: **平均計算量の主張は、入力分布を攻撃者が握った瞬間に無効になる**。
- **ランダム化はセキュリティ機構であると同時に再現性の敵である。** ハッシュシードのランダム化はテストの flakiness や「本番でだけ順序が違う」を生む。CPython は `PYTHONHASHSEED=0` で無効化できる (デバッグ用・本番では危険)、Go/Rust は無効化の公式手段を提供しない——再現性と防御のトレードオフで三者の位置が微妙に違う。
- **表そのものの実装は unsafe の塊である。** CPython の dict は C で書かれ、hashbrown は Rust の unsafe を多用する (生ポインタ・未初期化スロット)。「安全な言語のハッシュ表」の安全性は、少数の検証された unsafe 実装に全ユーザが相乗りする構造で成立している——[`language-implementations-cpython-go-rust.md`](language-implementations-cpython-go-rust.md) の「unsafe が正しく包まれていること」の代表例であり、miri や fuzzing がその検証装置である。
- **Go の map は並行書き込みを検出して即 fatal になる** (回復不能・recover 不可)。これは診断機構であって同期機構ではなく、すり抜けた競合は前掲書の通りメモリ安全性を破りうる。共有するなら `sync.Map` かロックを明示する——「検出器で直す」哲学の、標準ライブラリ内での現れである。

## 結論 (Conclusions)

- ハッシュ表と文字列の実装は言語設計の縮図である。dict のポインタ表は「すべてがオブジェクト」の、map のアドレス禁止と漸進疎開は「GC と低レイテンシ」の、HashMap のコンパイル時無効化検査は「所有権」の、それぞれ論理的帰結であり、データ構造の癖は言語の中核から導出できる。
- 同時に、ハードウェアは言語の違いを押し流す。三者が独立に開番地法へ、そして Rust と Go が揃って SwissTable へ収斂したのは、キャッシュラインと SIMD が設計空間の勝者を決めたからである——アルゴリズムの教科書の漸近記法の外側に、実装性能の本体がある。
- 反復順序 (追認して仕様化 / ランダム化で妨害 / 未規定のまま) とイテレータ無効化 (実行時検査 / 仕様で定義 / コンパイルエラー) は、前掲書の「支払時期」テーゼの各論として綺麗に整列する。標準ライブラリは言語哲学が最も具体的な機械語になる場所である。
- HashDoS の教訓——平均計算量は敵対的入力の下で崩れる——は、標準ライブラリの既定を「安全側 + 明示的オプトアウト」に統一させた。既定の安全と最高性能の分離は、[`managed-runtimes-gc-jit.md`](managed-runtimes-gc-jit.md) の GC 選択と同じく、「唯一の正解」ではなく位置取りの問題である。

### 次の接続先 (Related Topics)

- [`language-implementations-cpython-go-rust.md`](language-implementations-cpython-go-rust.md) — 本書の親。支払時期テーゼと boxing の税の一般論。
- [`async-executors-asyncio-tokio.md`](async-executors-asyncio-tokio.md) — 姉妹編の各論 (その 2)。実行器の内部と協調スケジューリングの統治。
- [`../../hardware/cpu/cache-memory-hierarchy.md`](../../hardware/cpu/cache-memory-hierarchy.md) — 開番地法と SwissTable を勝たせた物理。
- [`../../hardware/cpu/simd-vector.md`](../../hardware/cpu/simd-vector.md) — 制御バイト一括照合を可能にする命令群。
- [`managed-runtimes-gc-jit.md`](managed-runtimes-gc-jit.md) — 漸進疎開と同型の「停止を薄く塗る」設計の一般論。
- [`../concurrency/memory-models-synchronization.md`](../concurrency/memory-models-synchronization.md) — map の並行書き込み検出の下にある一般問題。

## 参考文献 (References)

**一次資料 (Primary — 論文・仕様・設計文書)**

- Aumasson, J.-P., & Bernstein, D. J. (2012). SipHash: A Fast Short-Input PRF. *INDOCRYPT 2012*, LNCS 7668. DOI: 10.1007/978-3-642-34931-7_28 — HashDoS 対策の標準となった PRF の原典。
- [PEP 393 — Flexible String Representation](https://peps.python.org/pep-0393/)、[PEP 412 — Key-Sharing Dictionary](https://peps.python.org/pep-0412/)、[PEP 456 — Secure and interchangeable hash algorithm](https://peps.python.org/pep-0456/) — Accessed: 2026-08-15。
- CPython ソース — [`Objects/dictobject.c`](https://github.com/python/cpython/blob/main/Objects/dictobject.c) (compact dict の設計コメントを含む)、[`Objects/unicodeobject.c`](https://github.com/python/cpython/blob/main/Objects/unicodeobject.c) — Accessed: 2026-08-15。
- [The Go Programming Language Specification — For statements / Index expressions](https://go.dev/ref/spec) — map 反復・アドレス不可の規定。Accessed: 2026-08-15。
- Klink, A., & Wälde, J. (2011). [Efficient Denial of Service Attacks on Web Application Platforms (28C3)](https://events.ccc.de/congress/2011/Fahrplan/events/4680.en.html) — hash flooding 攻撃の公表。Accessed: 2026-08-15。

**二次資料 (Secondary — 実装解説)**

- Kulukundis, M. (2017). [Designing a Fast, Efficient, Cache-friendly Hash Table, Step by Step (CppCon 2017)](https://www.youtube.com/watch?v=ncHmEUmJZf4) および [Abseil: Swiss Tables Design Notes](https://abseil.io/about/design/swisstables) — SwissTable の設計原典解説。Accessed: 2026-08-15。
- Go Blog (2025). [Faster Go maps with Swiss Tables](https://go.dev/blog/swisstable) — Go 1.24 の map 刷新の一次解説 (高速化は自社計測値)。Accessed: 2026-08-15。
- [hashbrown](https://github.com/rust-lang/hashbrown) — Rust std の HashMap 実装。README に SwissTable 移植の設計判断。Accessed: 2026-08-15。
- Hettinger, R. (2012). [More compact dictionaries with faster iteration (python-dev)](https://mail.python.org/pipermail/python-dev/2012-December/123028.html) — compact dict の提案原文。Accessed: 2026-08-15。
- Pike, R. (2013). [Strings, bytes, runes and characters in Go](https://go.dev/blog/strings) — Go 文字列設計の一次解説。Accessed: 2026-08-15。
