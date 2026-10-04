# データベースのストレージエンジン (Database Storage Engines)

## 概要 (Overview)

データベース (database) は、論理的には「キーで引ける、書き換えられる、消えない表」という抽象を提供する。だが現実のハードウェアはこの抽象を直接は与えない。主記憶 (DRAM) は速いが**揮発 (volatile)** で容量に限りがあり、ディスク/SSD は不揮発で大容量だが、CPU から見ると桁で遅く、しかも**ブロック単位 (block-granular)** でしかアクセスできない。この「速いが消える層」と「消えないが遅くブロック単位の層」の間の溝を、専用のデータ構造とプロトコルで埋める層が**ストレージエンジン (storage engine)** である。InnoDB、RocksDB、WiredTiger、LMDB はいずれもこの層の実装である。

`../../hardware/hw-sw-interface/storage-nvm-interface.md` は、その下のハードウェア——NANDフラッシュの erase-before-write、書き込み増幅 (write amplification, WAF)、FTL、`fsync` が重い理由——を扱った。本ドキュメントはその**真上の層**である。ストレージエンジンが解く問題は、ハードウェアの非対称性に直接根ざしている。すなわち (1) シーケンシャルアクセスはランダムアクセスより桁で速い、(2) SSD では書き込みが摩耗と WAF を生み、消去はブロック単位、(3) `fsync` は高価で、しかも「書いた」と「残った」は別物。ストレージエンジンの設計選択は、ほぼすべてこの3つの不等式への応答として読める。

本ドキュメントは、ストレージエンジンの2大系譜——**B+tree（in-place update, 読み最適化）** と**LSM-tree（out-of-place/append, 書き込み最適化）**——を、read/write/space amplification の三つ巴トレードオフ (RUM conjecture) を中心に Level 3 で整理する。さらに、両者に共通する永続性の土台——**先行書き込みログ (write-ahead log, WAL)**、バッファプール (buffer pool)、チェックポイント (checkpoint)、ARIES——と、ページレイアウト・並行制御・セキュリティへ繋ぐ。並行制御 (concurrency control) とトランザクション分離 (isolation) の詳細は概観に留め、今後の transactions ドキュメントへ譲る。

## 方法論 (Methodology)

- ストレージエンジンを「**何をどう並べるか (data layout)**」「**どう探すか (index structure)**」「**どう壊れず更新するか (durability & recovery)**」の3問に分解し、それぞれを B+tree と LSM-tree で対比する。
- すべての設計判断を、`../../hardware/hw-sw-interface/storage-nvm-interface.md` の**ハードウェアの非対称性**（シーケンシャル≫ランダム、書き込みコスト、`fsync` の重さ）へ明示的に接続する。データ構造はこの非対称性を回避するための契約である、という視点を一貫させる。
- トレードオフの座標系として **RUM conjecture**（Read / Update / Memory(space) の3増幅のうち2つしか同時に最適化できない）を用い、各設計がこの三角形のどこに立つかを示す。
- 永続性は「**正しさ (correctness)** の問題」として、性能とは独立の軸で扱う。WAL プロトコル・ARIES・torn write 対策は、性能の話ではなく「電源が落ちても約束を破らない」ための話である。
- 数値（ページサイズ、fanout、増幅率）は実装・ワークロード依存で大きく変動する。絶対値ではなく**桁感 (order of magnitude)** と**相対的な序列**を理解の基準とする。

## 発見事項 (Findings)

1. **ストレージエンジンの本質は「ランダムを避ける契約」である。** ハードウェアはシーケンシャルが桁で速い。B+tree はデータを**整列 (sorted)** させてランダムを局所化し、LSM-tree は更新を**追記 (append)** に変換してランダム書き込みをシーケンシャルに化けさせる。アプローチは逆だが、目的はどちらも「ハードが嫌うランダムアクセスを構造で減らす」ことである。

2. **B+tree と LSM-tree の対立は RUM トレードオフの2つの極である。** B+tree は所定の位置を上書きする (in-place) ため読みが速い（=低 read amplification）が、ランダム書き込みとページ分割で書き込みコストが高い。LSM-tree は追記して後でまとめる (out-of-place) ため書き込みが速い（=低 write amplification の書き込み経路）が、読みが複数の層を探すため遅く、compaction が別の write/space amplification を生む。**両方を同時に最適化はできない**——これが RUM conjecture の含意である。

3. **ページは「ハードウェアのブロックに合わせた論理単位」である。** B+tree のノードは通常 4–16 KiB のページ (page) で、これは `../../hardware/hw-sw-interface/storage-nvm-interface.md` の NAND ページや OS のブロックサイズと整合させてある。I/O 回数は木の高さに比例し、高い fanout（1ページに多数のキー）が高さを抑え、I/O 回数を対数的に保つ。

4. **永続性は WAL プロトコルという「順序の契約」で成り立つ。** バッファプール上の dirty page をいつディスクへ落とすかは性能の自由だが、「**変更を記録したログを、対応するデータページより先に永続化する (write-ahead)**」という順序だけは絶対に守らねばならない。これを破ると、電源喪失後に「ログにないデータページの変更」が残り、回復不能になる。WAL は速度ではなく正しさの機構である。

5. **「書いた」と「残った」の差は、ハードウェアまで貫く。** WAL のコミットは `fsync`/`fdatasync` でデバイスの揮発キャッシュを NAND へ落として初めて確定する（`../../hardware/hw-sw-interface/storage-nvm-interface.md` の FLUSH/FUA）。さらに、ページ書き込みの最中に電源が落ちる **torn write (partial page write)** はハードのブロック粒度とページ粒度の不一致から生じ、double-write buffer や full-page write で別途守らねばならない。

6. **削除は「消去」ではない。** LSM-tree の削除は tombstone（墓標）を追記するだけで、旧値は未コンパクションの SSTable に残る。B+tree でも領域は再利用されるまで残骸を含む。さらに下層では FTL の out-of-place 更新で物理ページにも旧値が残る。論理削除・物理上書き・物理消去の三段の乖離が、GDPR の削除権とセキュリティ上の緊張を生む。

## アーキテクチャ (Architecture)

### ストレージエンジンの全体像

アプリの `INSERT`/`SELECT` が物理メディアに届くまで。各層が抽象を1段ずつ提供し、`../../hardware/hw-sw-interface/storage-nvm-interface.md` の I/O スタックの真上に載る。

```
   ┌────────────────────────────────────────────────┐
   │ Query Execution (planner, operators)            │ 論理: 行/タプル
   ├────────────────────────────────────────────────┤
   │ Access Methods   B+tree  /  LSM-tree            │ 論理: キー → 値
   │   ├ index (clustered / secondary)               │
   │   └ MVCC version visibility (概観)               │
   ├────────────────────────────────────────────────┤
   │ Buffer Pool (DRAM)  ─ dirty page / 置換ポリシ     │ ★ もう一段のキャッシュ階層
   │   page cache of on-disk pages                   │   (cache-memory-hierarchy.md)
   ├────────────────────────────────────────────────┤
   │ WAL (write-ahead log)  +  Checkpoint  +  ARIES   │ ★ 永続性と回復
   │   log records → fsync → redo/undo                │
   ├────────────────────── fsync / fdatasync ─────────┤  ★ ここで HW 境界へ
   │ File System / Block Layer  (page ↔ block)        │
   ├────────────────────────────────────────────────┤
   │ Device + FTL + NAND  (storage-nvm-interface.md)  │ 物理: erase-before-write
   └────────────────────────────────────────────────┘
            ▲ シーケンシャル ≫ ランダム / fsync は重い
```

### バッファプールはキャッシュ階層のもう一段

バッファプール (buffer pool) は、オンディスクのページを DRAM に保持するソフトウェア管理キャッシュである。CPU の L1/L2/L3 が DRAM を隠すのと**同形の構造**であり、`../../hardware/cpu-architecture/cache-memory-hierarchy.md` の階層に、DB が明示的に管理するもう一段を足したものと読める。

```
   CPU regs → L1 → L2 → L3 → DRAM → [Buffer Pool] → Disk/SSD
   ↑ ハードが自動管理 (cache-memory-hierarchy.md)  ↑ DBが明示管理
                                          ・ヒット=DRAMアクセス
                                          ・ミス=ブロックI/O (桁で遅い)
                                          ・dirty=未書き戻し=要WAL
```

- **ヒット/ミス**: 要求ページがプール内ならヒット（DRAM 速度）、無ければミスでブロック I/O（桁で遅い）。ハードのキャッシュミスと同じく、ミス率がそのまま性能を支配する。
- **置換ポリシ (replacement)**: 古典 LRU はスキャン1回で有用ページを追い出す (cache pollution) ため、多くの DB は LRU の改良（MySQL InnoDB の midpoint LRU、PostgreSQL の clock-sweep など）を使う。これは CPU キャッシュが純 LRU を近似で済ませるのと同じ動機。
- **dirty page**: 書き換えたがまだディスクへ書き戻していないページ。これが永続性問題の発生源であり、WAL とチェックポイントが管理する。

### B+tree と LSM-tree — 2大系譜

```
   ┌──────────────── B+tree ────────────────┐   ┌──────────────── LSM-tree ───────────────┐
   │ 所定位置を上書き (in-place update)        │   │ 追記して後でまとめる (out-of-place)       │
   │ データは常に整列済み (sorted on disk)     │   │ MemTable(整列, RAM) → 不変SSTable へ flush│
   │ 読み: 木を1パス下る (低 read amp)         │   │ 読み: 複数層を新しい順に探す (高 read amp) │
   │ 書き: ランダム書き+ページ分割 (高 write)   │   │ 書き: 追記=シーケンシャル (低 write 経路)  │
   │ → 読み最適化 (OLTP read-heavy)           │   │ compaction が write/space amp を生む      │
   │ 例: InnoDB, WiredTiger, LMDB             │   │ → 書き込み最適化。例: RocksDB, Cassandra  │
   └─────────────────────────────────────────┘   └──────────────────────────────────────────┘
```

## 技術詳細 (Technical Details)

### 1. RUM conjecture — 三つ巴のトレードオフ

ストレージエンジンの設計判断は、3種の**増幅 (amplification)** の綱引きに帰着する。

- **Read amplification（読み増幅）**: 論理的に1件読むのに、実際に読むデータ量。LSM-tree で複数層・複数 SSTable を探ると増える。
- **Write amplification（書き込み増幅）**: 論理的に1件書くのに、実際に書くデータ量。LSM の compaction や B+tree のページ全体書き戻しで増える。`../../hardware/hw-sw-interface/storage-nvm-interface.md` の FTL の WAF とは**別レイヤだが乗算的に重なる**——DB の write amp が SSD の WAF を増幅し、摩耗を加速する。
- **Space amplification（領域増幅）**: 論理データ量に対する実占有量。LSM の未コンパクションの重複・tombstone、B+tree の fill factor 余白で増える。

**RUM conjecture**（Athanassoulis ら, 2016）の主張は「Read・Update・Memory(space) の3つの増幅を**同時に最小化することはできず、最大2つしか良くできない**」というもの。

```
                Read amp
                  /\
                 /  \      B+tree: Read を取り、Update(write) を犠牲に寄る
                /    \     LSM    : Update(write) を取り、Read/Space を犠牲に寄る
               /______\    どの設計もこの三角形の内側のどこか1点を占める
          Update      Space        ── 三辺すべてを頂点に寄せることはできない
            amp        amp
```

この三角形が、以降の B+tree と LSM-tree の対比の座標系である。

### 2. B+tree — in-place・読み最適化

**B+tree** は、整列したキーを多分木 (high-fanout tree) に保持し、葉ノードだけが実データ（または行へのポインタ）を持つ。内部ノードはキーと子ポインタだけを持ち、葉は左右に連結 (linked) され範囲スキャン (range scan) を可能にする。

```
   内部ノード (キー + 子ポインタのみ。fanout を最大化)
   ┌───────────────────────────────────────┐
   │  [k1 | k2 | k3]   → 3キーで4子へ分岐     │   高さ h, fanout f
   └──┬─────┬─────┬─────┬──────────────────┘   ⇒ 約 f^h 件を h 回のI/Oで到達
      ▼     ▼     ▼     ▼
   ┌──────┐┌──────┐┌──────┐┌──────┐
   │葉:行 ││葉:行 ││葉:行 ││葉:行 │  葉のみ実データを保持
   └──┬───┘└──┬───┘└──┬───┘└──────┘
      └───────┴───────┴── 葉は連結 → 範囲スキャンが順次I/Oで済む
```

- **ノード = ページ**: 各ノードは1ページ（通常 4–16 KiB）で、`../../hardware/hw-sw-interface/storage-nvm-interface.md` の NAND ページ／OS ブロックサイズに整合させる。これにより1ノードの読み書きが1回のブロック I/O に収まる。
- **fanout と高さ**: 1ページに収まるキー数が fanout `f`。`N` 件のデータは高さ `h ≈ log_f(N)` の木になり、ルートから葉まで `h` 回の I/O で到達する。`f` が大きい（数百）ため、数十億件でも `h` は 3–4 程度。**読みのコストは木の高さ＝対数的**で、これが B+tree の読み速さの核。
- **ページ分割とマージ (split / merge)**: 葉が満杯のページに挿入が来ると、ページを2つに**分割 (split)** し、中央キーを親へ押し上げる。削除でページが過疎になると**マージ (merge)** または再分配する。分割は親の更新を連鎖させうる（最悪ルートまで）。これがランダム書き込みと write amp の源で、B+tree が書き込みに弱い理由。
- **fill factor**: ページを満杯まで詰めない余白率。余白があれば将来の挿入が分割を起こさず済む（write amp 低減）が、その分 space amp が増える——RUM の Space を犠牲に Update を稼ぐ調整ノブ。
- **クラスタ化 vs 非クラスタインデックス**: **クラスタ化 (clustered)** インデックスは葉に行本体そのものを置く（主キー順に物理配置。InnoDB の主キー）。**非クラスタ (secondary)** インデックスの葉は主キー値（または行ID）を持ち、本体へもう一段引く必要がある——これが secondary index の追加 I/O。範囲スキャンはクラスタ化キーで順次 I/O になり、シーケンシャルの利を最大化する。
- **B-link tree による並行性**: 素朴な B+tree は分割中に親子へ同時にラッチ (latch) が要り並行性が落ちる。**B-link tree**（Lehman & Yao, 1981）は各ノードに右隣への **link pointer** と high-key を足し、「分割が親に反映される前でも、right-link を辿れば正しい葉に着ける」ようにする。これにより**ラッチを1つずつ手放しながら下れる (latch crabbing / coupling)**——子のラッチを取ってから親を放す——並行制御が可能になり、現代の B+tree 実装の標準である。

### 3. LSM-tree — out-of-place・書き込み最適化

**LSM-tree (Log-Structured Merge-tree)**（O'Neil ら, 1996）は、ランダム書き込みを**追記とバッチ統合**に変換する。

```
   write ──▶ ┌──────────────┐  full ┌──────────────┐
             │ MemTable     │ ────▶ │ 不変 SSTable  │ ──┐ flush (シーケンシャル書き)
             │ (RAM, 整列    │       │ (sorted, 不変)│   │
             │  skip list/   │       └──────────────┘   ▼
             │  赤黒木)       │              Level 0 [SST][SST][SST]   (重なり許容)
             └──────┬───────┘                 │ compaction
                    │ 同時に WAL へ追記(永続)    ▼
              (クラッシュ時 WAL から MemTable 再生)  Level 1 [────SST────][────SST────]
                                                   │ compaction (size-tiered / leveled)
                                                   ▼
                                            Level 2 [──────────SST──────────] ...
                                            下層ほど大きく・古く・キー範囲が整然
```

- **MemTable**: RAM 上の整列構造（通常 **skip list** か赤黒木 red-black tree）。書き込みはまずここへ入り、同時に WAL へ追記される（永続性のため）。MemTable が一杯になると不変化し、新しい MemTable に切り替える。
- **flush と SSTable**: 満杯の MemTable は **SSTable (Sorted String Table)** としてディスクへ**シーケンシャルに**書き出される。SSTable は一度書いたら**不変 (immutable)**——上書きしない。書き込み経路がすべて追記＝シーケンシャルなので、ランダム書き込みが消え、HDD でも SSD でも速く、SSD の WAF も抑えられる。これが LSM の書き込み速度の核。
- **読み経路と read amplification**: あるキーを読むには、新しい順に MemTable → Level 0 → Level 1 → … と探し、最初に見つかった値（または tombstone）を採る。最悪、全層を触る——これが **read amplification**。各 SSTable には **Bloom filter**（後述）を付け、「このキーは確実に居ない」を高速判定して I/O を省く。
- **compaction（コンパクション）**: 不変な SSTable は溜まる一方なので、定期的に複数 SSTable をマージソートして新しい SSTable に書き直し、重複キーと tombstone を解消する。2大戦略がある。
  - **size-tiered**: 同サイズの SSTable が一定数たまったらまとめて1つに。**書き込み増幅は小さい**が、同じキーの版が複数層に散らばり **read/space amplification が大きい**（Cassandra の既定）。
  - **leveled**: 各レベルをサイズ上限つきの「キー範囲が重ならない整然な層」に保ち、上層が溢れたら下層へマージ。**read/space amplification は小さい**（各レベル内でキー範囲が一意）が、マージ頻度が高く **write amplification が大きい**（RocksDB の既定）。
  - これは RUM 三角形そのもの——size-tiered は Write 頂点寄り、leveled は Read/Space 頂点寄り。
- **compaction が招く読み遅延スパイク**: compaction は大量の SSTable を読み書きする重い背景処理であり、前景の読み I/O 帯域とバッファを奪う。結果、**テールレイテンシ (tail latency) のスパイク**が起きる。これは `../../hardware/hw-sw-interface/storage-nvm-interface.md` の **GC が招くテールレイテンシ**と同型の現象が、DB レイヤで再現したものである（LSM の compaction と FTL の GC は「追記して後で回収する」同じ構造の二重化）。

### 4. Bloom filter による読み回避

LSM の read amplification を抑える鍵。**Bloom filter** は「集合に**確実に居ない**こと」を、偽陽性 (false positive) ありで判定する確率的データ構造。

```
   k 個のハッシュ → mビットのビット配列の k 箇所を立てる
   問い合わせ: k 箇所が全部1 → 「居るかもしれない」(偽陽性あり)
              1箇所でも0    → 「確実に居ない」(偽陰性なし) ⟹ この SSTable を読まずに済む
```

各 SSTable にキー集合の Bloom filter を付けると、「このキーは確実にこの SSTable に居ない」場合に**ディスク I/O を完全に省ける**。偽陰性が無い（居るものを居ないとは言わない）ため正しさを壊さず、偽陽性は「無駄に1回読む」だけのコストで済む。LSM の point lookup を実用速度に保つ要。

### 5. 永続性と障害回復 — WAL・バッファプール・チェックポイント・ARIES

ストレージエンジンの正しさの核。すべては「電源が任意の瞬間に落ちても、コミット済みは残り、未コミットは残さない」を保証するためにある。

**WAL プロトコル（先行書き込み規則）**:

```
   鉄則: データページを書き換える前に、その変更を記述したログを永続化せよ。
   ─────────────────────────────────────────────────────────────
   ① 変更を WAL レコードとして生成 (redo情報 + undo情報)
   ② ログをディスクへ fsync (← ここでデバイス揮発キャッシュも NAND へ)
   ③ ②の完了後にのみ、トランザクションを「コミット済み」と返答してよい
   ④ dirty page の実書き戻しは後でよい (チェックポイントが回収)
```

- **なぜ先行か**: ログを先に永続化しておけば、電源喪失後に「ログにあってデータページに未反映」の変更は **redo（前ロール）** で再適用でき、「ログになくデータページに反映済み」の未コミット変更は **undo（後ロール）** で巻き戻せる。逆順（データ先・ログ後）だと、ログに無い変更がページに残り、回復の根拠が消える。
- **fsync とハードウェアの接続**: WAL のコミットが速度を支配するのは、`fsync`/`fdatasync` が `../../hardware/hw-sw-interface/storage-nvm-interface.md` の FLUSH を発行し、デバイスの揮発キャッシュを NAND へ落とすまで待つから。`fdatasync` はメタデータ更新を省きデータだけ落とす軽量版。エンタープライズ SSD の PLP（電源喪失保護, コンデンサ）があれば、FLUSH を実質ノーコストにでき WAL コミットが速くなる——DB の `fsync` の重さは、そのままハードの永続化機構に律速される。
- **WAL を逐次にまとめる (group commit)**: 個々のコミットで毎回 `fsync` すると遅いので、短時間の複数コミットを1回の `fsync` に束ねる **group commit** が標準。WAL が**追記=シーケンシャル**書き込みである点も、ランダム I/O を避けハードに優しい。

**チェックポイント (checkpoint)**: WAL は無限には伸ばせない（回復時の redo 範囲も伸びる）。定期的に「この時点までの dirty page はすべてディスクへ書き戻した」と記録し、それ以前のログを回復に不要として切り詰める。

**ARIES**（Mohan ら, 1992）は、WAL ベース回復の事実上の標準アルゴリズム。3つの原則と3フェーズで成る。

```
   原則:
    (1) WAL          : ログを先に永続化 (上記の鉄則)
    (2) Repeating History during Redo:
                      回復時、コミット未コミットを問わずまず全変更を redo で
                      「クラッシュ直前の状態」へ完全再現してから、未コミットを undo する
    (3) Logging during Undo: undo 操作自体も CLR (Compensation Log Record) として
                      ログに残す ⟹ 回復中に再クラッシュしても二重 undo しない (冪等)

   3フェーズ:
    Analysis ─▶ 最後の checkpoint から WAL を走査し、dirty page と active txn を特定
    Redo     ─▶ 必要な最古 LSN から全ログを再適用し、クラッシュ直前を再現
    Undo     ─▶ 未コミット txn を逆順に巻き戻し、各 undo を CLR で記録
```

- **LSN (Log Sequence Number)**: 各ログレコードと各ページに付く単調増加番号。ページの LSN とログの LSN を比べ、「このページにこのログは既に反映済みか」を判定する——redo の冪等性の鍵。
- **fuzzy checkpoint**: 全 dirty page を書き戻すまでシステムを止める素朴なチェックポイントは高価。**fuzzy checkpoint** は、進行中のトランザクションを止めず、チェックポイント開始時点の dirty page リストと active txn を記録するだけにして、実際の書き戻しは背景で進める。停止時間をほぼゼロにする実用的工夫。
- ここは並行制御・分離レベルと密接に絡むが、本ドキュメントは回復の枠組みまで。ロックと MVCC の詳細は後述の概観と、今後の transactions ドキュメントへ。

### 6. torn write と partial page write

WAL があっても守れない別の障害。DB のページ（例 16 KiB）は、ハード/FS のブロック（例 4 KiB）**複数個**にまたがって書かれる。書き込みの途中で電源が落ちると、ページの一部だけ新しく一部が古い**ちぎれた書き込み (torn write / partial page write)** が生じる。

```
   16KiB DB ページ = 4KiB ブロック ×4 を順に書く最中に電源喪失:
   ┌──────┬──────┬──────┬──────┐
   │ new  │ new  │ OLD  │ OLD  │  ← 中身が新旧混在で内部矛盾 (チェックサム不一致)
   └──────┴──────┴──────┴──────┘
   WAL の redo は「正しいページに差分を当てる」前提なので、ページ自体が壊れていると救えない
```

対策は次のとおりである。

- **double-write buffer**（InnoDB）: dirty page を本来の位置へ書く前に、まず連続領域の「二重書きバッファ」へ書いて `fsync` し、その後に本来位置へ書く。本来位置が torn になっても、回復時に二重書きバッファの無傷なコピーから復元できる。
- **full-page write**（PostgreSQL）: チェックポイント後に各ページを最初に変更するとき、差分でなく**ページ全体を WAL に書く**。torn なページは WAL 中の完全コピーで丸ごと置換できる。代償は WAL 量の増加（write amp）。
- これらは `../../hardware/hw-sw-interface/storage-nvm-interface.md` の「ページ粒度とブロック粒度の不一致」が DB レイヤに顔を出したもの。アトミックな大ページ書き込みをハードが保証しない以上、ソフトで原子性を作り直すしかない。

### 7. ページレイアウト — slotted page・行指向 vs 列指向・圧縮

ページ内部のバイト配置も、アクセスパターンに直結する設計判断。

**slotted page**: 可変長レコードを1ページに詰める標準レイアウト。

```
   ┌────────────────────────────────────────────────┐
   │ Header │ slot[0] slot[1] slot[2] ... →           │  slot 配列はオフセットの索引
   │        │ (各 slot = レコードへのオフセット+長さ)   │  前から伸びる
   │········│··············空き············ │·········· │
   │              ← レコード実体は後ろから詰める ······· │  後ろから伸びる
   │ ... rec2 │ rec1 │ rec0                            │
   └────────────────────────────────────────────────┘
   ・slot 配列とレコード実体が両端から中央へ伸び、間が空き領域
   ・レコード移動(更新で伸長)しても slot 経由なので外部ポインタは無効化されない
   ・削除は slot を無効化するだけ ⟹ 旧データはコンパクションまで残る (セキュリティ考察)
```

- **行指向 (row-oriented) vs 列指向 (column-oriented)**: 行指向は1行の全列を隣接配置し、**OLTP**（1行を読み書きするトランザクション）に最適。列指向は同一列の値を連続配置し、**OLAP**（少数列を全行集計する分析）に最適——必要な列だけ読み I/O を減らし、同種値が並ぶため圧縮も効く。同じデータでもワークロードでレイアウトを変えるべき、という RUM の Read 側の最適化。
- **圧縮 (compression)**: **dictionary encoding**（頻出値を小さなコードに置換）と **run-length encoding (RLE)**（連続する同値を「値×回数」に畳む）が代表。列指向で特に効く（同種・低カーディナリティの値が並ぶ）。圧縮は space amp を下げ、I/O 量も下げる（読むバイトが減る）が、CPU の伸長コストと引き換え——これも Read/Space/CPU のトレードオフ。

### 8. 並行制御への入口 — ラッチ・ロック・MVCC（概観）

複数トランザクションが同じページ・同じ行に触るときの調停。**本ドキュメントは入口の概観に留め、詳細は今後の transactions ドキュメントへ譲る。**

- **ラッチ (latch) とロック (lock) は別物**である。
  - **ラッチ**は、データ構造（B+tree のノード、バッファプールのフレーム）を**物理的な破壊から守る**短命な相互排他。保持はマイクロ秒オーダ、デッドロック検出はせず（取得順序の規律で回避）、回復の対象外。これは `../concurrency/memory-models-and-synchronization.md` の**ユーザ空間ロック（mutex/spinlock）そのもの**——CPU のアトミック命令とメモリバリアの上に立つ。B-link tree の latch crabbing はこのラッチの取り回しの話。
  - **ロック**は、トランザクションの**論理的な分離 (isolation)** を守る長命な論理ロック。トランザクション境界まで保持し、デッドロック検出を行い、分離レベル (isolation level) を実現する。回復の対象。
  - 「latch は構造を、lock は論理を守る」「latch は短く規律で、lock は長くデッドロック検出で」が区別の核。
- **MVCC（多版型同時実行制御, Multi-Version Concurrency Control）** は、書き込みが読み手をブロックしないよう、**同一行の複数版 (version)** を保持する。版をストレージにどう置くかで2系統に分かれる。
  - **追記型 (append-only / in-place new version)**: PostgreSQL は新版を**テーブル内に新しいタプルとして追記**し、旧版に「いつまで有効か (xmin/xmax)」を刻む。読み手は自分のスナップショットで見える版を選ぶ。死んだ旧版は **VACUUM** が後で回収する——LSM の compaction や FTL の GC と同じ「追記して後で掃除」構造。
  - **undo log 型**: InnoDB は最新版を所定位置に置き、**旧版を undo log に退避**する。読み手は undo log を遡って自分の時点の版を再構成する。undo log は回復（未コミットの巻き戻し）とも共用される。
  - どちらも「読みと書きの衝突を、版を増やすことで回避する」点で共通し、その版をどこに積むか（テーブル本体か別ログか）がストレージ設計の分岐。詳細な可視性規則・分離レベル・SI/SSI は transactions ドキュメントへ。

## セキュリティ考察 (Security Considerations)

ストレージエンジンは**永続データの番人**であり、その永続性・残留性・隔離の各性質が、そのままセキュリティ上の論点になる。

- **保存時暗号化 (encryption at rest / TDE)**: **Transparent Data Encryption (TDE)** は、データファイル・WAL・一時ファイルをディスクへ書く直前に暗号化 (AES) する。鍵管理が核心——鍵は外部 KMS/HSM で保持し、DB プロセスには使用時のみ渡すのが堅い。データファイルだけ暗号化して **WAL やバックアップを平文で残す**のは典型的な漏洩経路で、暗号化の境界は**ログ・一時表・スワップまで**含めねば意味が薄い。ハード層の自己暗号化ドライブ (SED/TDE, `../../hardware/hw-sw-interface/storage-nvm-interface.md`) と多層化すると、片方の鍵漏洩で全滅しない。

- **WAL／バックアップからの情報漏洩**: WAL は**変更前後の値 (before/after image)** を含むため、本体テーブルに無い「過去の値・削除済みの値」まで保持しうる。バックアップ・レプリカ・WAL アーカイブは本体と同じ機密度で保護・暗号化・アクセス制御せねばならない。`../distributed-systems/consistency-models.md` のレプリケーションで WAL/ログを他ノードへ転送する経路（log shipping）も、平文なら盗聴で全変更履歴が漏れる——durability のための複製が攻撃面を広げる。

- **削除データのディスク残留 (data remanence)**: 論理削除は物理消去ではない、という乖離が三段で重なる。
  - **LSM の tombstone**: 削除は墓標を追記するだけ。**旧値は未コンパクションの SSTable に残り続け**、compaction が当該キーの全版を処理して初めて消える。
  - **B+tree の slotted page**: 削除は slot を無効化するだけで、レコードバイトはページ再編まで残骸として残る。
  - **FTL の out-of-place 更新**: さらに下層で、DB が「上書き」しても旧物理ページに値が残る（`../../hardware/hw-sw-interface/storage-nvm-interface.md` の data remanence）。
  - 帰結: **GDPR の削除権（消去権）との緊張**。「削除した」とユーザに返答した後も、旧値が複数レイヤに残る。確実な消去には、暗号化を前提とした **crypto shredding**（その行/テナントの鍵を破棄して復号不能化）や、強制 compaction・サニタイズが要る。論理削除の応答と物理消去の完了を混同してはならない。

- **`fsync` を欺くストレージ層によるデータ損失**: DB は `fsync` の成功を「永続化された」と信じるが、その下の層（嘘をつくディスクの揮発キャッシュ、`fsync` を握りつぶす一部仮想化/ネットワークストレージ、誤設定の `nobarrier`）が約束を破ると、コミット済みデータが電源喪失で消える。これは可用性・整合性の侵害であり、`../../hardware/hw-sw-interface/storage-nvm-interface.md` の FLUSH/FUA/バリアが正しく貫通しているかの検証（power-cut テスト、`diskchecker` 類）が要る。DB の正しさは、最下層まで `fsync` が誠実であることに依存する。

- **マルチテナントでの side channel（I/O・バッファプール）**: 共有 DB インスタンスで複数テナントがバッファプールや I/O 帯域を共有すると、片方のアクセスパターンが他方に観測されうる。あるテナントのクエリ後にページがバッファプールに載る→別テナントの同ページアクセスが速くなる、という**キャッシュタイミングチャネル**は、`../../hardware/cpu-architecture/cache-memory-hierarchy.md` のキャッシュ側チャネルと同型で、DB のバッファプール粒度で再現する。compaction/GC の I/O スパイクのタイミングからも負荷が漏れうる。緩和はテナントごとのバッファプール/インスタンス分離、I/O のレート制御、機密判定の定数時間化に帰着する。

## 結論 (Conclusions)

- ストレージエンジンは、揮発で速い DRAM と、不揮発で遅くブロック単位の SSD/HDD の間の溝を埋める層であり、その設計はほぼすべて `../../hardware/hw-sw-interface/storage-nvm-interface.md` のハードウェアの非対称性（シーケンシャル≫ランダム、書き込みコスト、`fsync` の重さ）への応答として読める。
- 2大系譜は RUM conjecture の2つの極である。**B+tree** は整列と in-place 更新で読みを最適化し（書き込みとページ分割を犠牲に）、**LSM-tree** は追記と compaction で書き込みを最適化する（読みと領域を犠牲に、Bloom filter で読みを救う）。三辺すべては同時に立てられない。
- 永続性は性能とは独立の正しさの軸であり、**WAL プロトコル（先行書き込み規則）** が土台、**ARIES**（repeating history redo + CLR つき undo + fuzzy checkpoint）がその標準的実現である。WAL のコミットは `fsync` を通じてハードの FLUSH に律速され、torn write は double-write/full-page write でハード粒度の不一致を補う。
- バッファプールは `../../hardware/cpu-architecture/cache-memory-hierarchy.md` のキャッシュ階層に DB が明示管理するもう一段を足したものであり、ヒット率・置換ポリシ・dirty page 管理という同形の問題を持つ。LSM の compaction は FTL の GC と、MVCC の旧版回収（VACUUM）も含め「追記して後で掃除する」同じ構造の各レイヤでの反復である。
- 並行制御では **latch（構造を守る短命なユーザ空間ロック, `../concurrency/memory-models-and-synchronization.md`）** と **lock（論理分離を守る長命な論理ロック）** を峻別し、MVCC は版をテーブル追記か undo log に置く。詳細は今後の transactions ドキュメントへ。
- セキュリティ上、TDE は WAL・バックアップ・一時表まで境界を広げねば意味が薄く、削除は tombstone・slotted page・FTL の三段で物理に残留して GDPR と緊張し、`fsync` を欺く下層はコミットを失わせ、マルチテナントのバッファプールは side channel になる。永続性・残留性・隔離が、そのまま防御線である。

### 次の接続先 (Related Topics)

- `../../hardware/hw-sw-interface/storage-nvm-interface.md` — 本ドキュメントの真下の層。NANDの erase-before-write、FTL、WAF、`fsync`/FLUSH/FUA、data remanence。DB の write amp はここの WAF と乗算的に重なり、compaction は FTL の GC と同型。
- `../../hardware/cpu-architecture/cache-memory-hierarchy.md` — バッファプールはこのキャッシュ階層のもう一段。ヒット/ミス・置換ポリシ・側チャネルが DB 粒度で再現する。
- `../concurrency/memory-models-and-synchronization.md` — latch はここのユーザ空間ロック（mutex/spinlock）そのもの。アトミック命令とメモリバリアの上に B-link tree の latch crabbing が立つ。
- `../distributed-systems/consistency-models.md` — 単一ノードの durability（WAL コミット）が、分散レプリケーション（log shipping, クォーラム）でどう拡張されるか。本ノードの永続性は分散一貫性の構成要素になる。

## 参考文献 (References)

- Bayer, R., & McCreight, E. (1972). Organization and Maintenance of Large Ordered Indices. *Acta Informatica*, 1(3), 173–189. DOI:10.1007/BF00288683
- Comer, D. (1979). The Ubiquitous B-Tree. *ACM Computing Surveys*, 11(2), 121–137. DOI:10.1145/356770.356776
- Lehman, P. L., & Yao, S. B. (1981). Efficient Locking for Concurrent Operations on B-Trees. *ACM TODS*, 6(4), 650–670. DOI:10.1145/319628.319663
- O'Neil, P., Cheng, E., Gawlick, D., & O'Neil, E. (1996). The Log-Structured Merge-Tree (LSM-Tree). *Acta Informatica*, 33(4), 351–385. DOI:10.1007/s002360050048
- Mohan, C., Haderle, D., Lindsay, B., Pirahesh, H., & Schwarz, P. (1992). ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging. *ACM TODS*, 17(1), 94–162. DOI:10.1145/128765.128770
- Athanassoulis, M., Kester, M. S., Maas, L. M., Stoica, R., Idreos, S., Ailamaki, A., & Callaghan, M. (2016). Designing Access Methods: The RUM Conjecture. *Proc. EDBT 2016*, 461–466. DOI:10.5441/002/edbt.2016.42
- Bloom, B. H. (1970). Space/Time Trade-offs in Hash Coding with Allowable Errors. *Communications of the ACM*, 13(7), 422–426. DOI:10.1145/362686.362692
- Stonebraker, M., Abadi, D. J., Batkin, A., et al. (2005). C-Store: A Column-oriented DBMS. *Proc. VLDB 2005*, 553–564.
- Abadi, D. J., Madden, S. R., & Hachem, N. (2008). Column-Stores vs. Row-Stores: How Different Are They Really? *Proc. SIGMOD 2008*, 967–980. DOI:10.1145/1376616.1376712
- Dong, S., Callaghan, M., Galanis, L., Borthakur, D., Savor, T., & Strum, M. (2017). Optimizing Space Amplification in RocksDB. *Proc. CIDR 2017*.
- Hellerstein, J. M., Stonebraker, M., & Hamilton, J. (2007). Architecture of a Database System. *Foundations and Trends in Databases*, 1(2), 141–259. DOI:10.1561/1900000002
- Graefe, G. (2011). Modern B-Tree Techniques. *Foundations and Trends in Databases*, 3(4), 203–402. DOI:10.1561/1900000028
- [RocksDB Wiki — Compaction & Leveled vs Universal](https://github.com/facebook/rocksdb/wiki) — Accessed: 2026-06-29
- [PostgreSQL Documentation — Reliability and the Write-Ahead Log](https://www.postgresql.org/docs/current/wal-intro.html) — Accessed: 2026-06-29
