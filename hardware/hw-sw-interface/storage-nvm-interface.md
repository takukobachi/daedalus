# ストレージ・不揮発メモリインタフェース (Storage & Non-Volatile Memory Interface)

## 概要 (Overview)

ストレージ (storage) は、メモリ階層 (memory hierarchy) の最下層に位置する**不揮発 (non-volatile)** な層であり、電源を切ってもデータが残る。CPU から見ると主記憶 (DRAM) より桁で遅く、桁で大容量で、桁で安い。この「遅いが消えない」層を、いかに速い主記憶に橋渡しするかが I/O サブシステムの主題である。

本ドキュメントは、現代ストレージの中核である **NANDフラッシュ (NAND flash)** の物理的制約（消去前書き込み erase-before-write、摩耗 wear）と、それを隠蔽する **FTL (Flash Translation Layer)**、そして CPU・OS とストレージを繋ぐプロトコル（**SATA/AHCI**, **NVMe**, **eMMC/UFS**）とブロックデバイス抽象を整理する。さらに、DRAM とストレージの境界を侵食しつつある**永続メモリ (persistent memory)** にも触れる。なぜ SSD が「使ううちに遅くなる」のか、なぜ `fsync` が重いのか、なぜ「完全消去」が難しいのか、をハードウェアの観点から説明する。

`../cpu-architecture/cache-memory-hierarchy.md` が「主記憶までの速度差」を扱ったのに対し、本稿はその下、ストレージ層の内部とインタフェースを扱う。

## 方法論 (Methodology)

- 「物理メディア（NANDの制約）」「論理抽象（FTL とブロックデバイス）」「転送経路（プロトコルと DMA）」「永続性保証（FLUSH/FUA とバリア）」の4層に分解する。
- 各層について、何が起きるか・なぜその制約があるか・ソフト（ファイルシステム/DB/OS）がどう振る舞うべきか、を対応づける。
- 典型的な1回の I/O（NVMe 経由のブロック読み書き）を、コマンド発行から完了割り込みまで1本のシーケンスとして通す。
- 数値は世代・実装で大きく変動する。絶対値ではなく**桁感 (order of magnitude)** を理解の基準とし、世代依存を都度断る。

## 発見事項 (Findings)

1. **NANDフラッシュは「書き換え (overwrite)」ができない。** 書き込みはページ (page) 単位だが、消去 (erase) はより大きなブロック (block) 単位でしか行えず、書き込みの前に必ず消去が要る（**erase-before-write**）。この非対称性が FTL の存在理由そのものである。
2. **FTL がランダムな論理アドレスを物理配置から切り離す。** OS が見る論理ブロックアドレス (LBA) と、NAND 上の物理位置の対応を FTL が動的に張り替える。ウェアレベリング (wear leveling)・ガベージコレクション (garbage collection) はすべてこの間接化の上で動く。
3. **「使ううちに遅くなる」「書いた以上に摩耗する」は GC と書き込み増幅 (write amplification) の帰結。** 空きブロックが枯渇すると GC が有効データを退避してから消去するため、ホストの1書き込みが内部では複数書き込みになる。TRIM はこの圧力を下げる。
4. **プロトコルの世代交代は「並列性」が主因。** SATA/AHCI は単一キュー・回転ディスク前提だが、NVMe は多数の深いキューを PCIe 上に持ち、フラッシュ本来の並列性とマルチコアを引き出す。
5. **永続性 (durability) は「書いた」と「残った」の差で決まる。** 揮発キャッシュ (volatile cache) があるため、`FLUSH`/`FUA`/ライトバリア (write barrier) で順序と永続化を明示しないと、電源喪失でデータが消える・前後する。永続メモリ (PMEM) はこの境界をバイト単位へ押し下げる。

## アーキテクチャ (Architecture)

### I/O スタックの全体像

アプリの `write()` が物理 NAND に届くまでの経路。各層が抽象を1段ずつ提供する。

```
   ┌──────────────────────────────────────────────┐
   │ Application  ( write() / fsync() / mmap )      │  論理: ファイル
   ├──────────────────────────────────────────────┤
   │ VFS / File System (ext4, XFS, NTFS …)          │  論理: inode, ジャーナル
   │   page cache (DRAM) ─ 遅延書き戻し / read-ahead │
   ├──────────────────────────────────────────────┤
   │ Block Layer (bio, I/O scheduler, merge/queue)  │  論理: LBA + セクタ
   │   FLUSH / FUA / TRIM(discard) をここで表現      │
   ├──────────────────────────────────────────────┤
   │ Driver  (NVMe / AHCI / UFS HCI)                │  ★ MMIO・DMA・割り込み
   ├───────────────────── PCIe / SATA / M-PHY ──────┤  ★ bus-interconnect
   │ Device Controller  +  FTL                      │  論理LBA → 物理PPA
   │   wear leveling / GC / ECC / SLC cache         │
   ├──────────────────────────────────────────────┤
   │ NAND Flash array  (die → plane → block → page) │  物理: セル
   └──────────────────────────────────────────────┘
            ▲ 不揮発 (non-volatile) / バイト書換不可
```

### NANDフラッシュの物理構造

NAND は階層構造を持ち、操作の粒度が**読み/書き/消去で異なる**のが最大の特徴。

```
   Device
    └─ Die (chip)            ← 並列動作の単位（複数 die で帯域を稼ぐ）
        └─ Plane
            └─ Block         ← 【消去 (erase) の単位】 数百〜数千ページ
                └─ Page      ← 【書き込み (program) / 読み出しの単位】 例: 4–16 KiB
                    └─ Cell  ← 1セル=1〜4bit (SLC/MLC/TLC/QLC)
```

- **読み出し (read)**: ページ単位。高速・低レイテンシ。ただし**リードディスターブ (read disturb)** — 同一ブロック内を繰り返し読むと隣接セルの電荷が乱れ、いずれ訂正不能になる。一定回数で予防的に再書き込み（リフレッシュ）する。
- **書き込み (program)**: ページ単位。**すでに書かれたページを上書きできない**。同じ場所を書き換えるには、まずブロック全体を消去する必要がある。
- **消去 (erase)**: ブロック単位（書き込みより桁で大きい）。時間も長い。
- **P/E サイクルと摩耗 (wear)**: 消去 (Program/Erase) のたびにセルの絶縁膜が劣化する。耐久は世代依存で、概ね SLC > MLC > TLC > QLC の順（目安として SLC は数万回、QLC は数百〜数千回オーダ）。寿命を使い切ったブロックは退役 (retire) させる。
- **セル多値化のトレードオフ**: 1セルに多ビット詰めるほど大容量・低コストだが、レイテンシ・耐久・信頼性が落ちる。多くの SSD は一部を擬似 SLC キャッシュ (SLC cache) として高速バッファに使い、空き時間に TLC/QLC へ移す。

### NVMe コマンド経路（キューと PCIe）

NVMe はホスト DRAM 上に **Submission Queue (SQ)** と **Completion Queue (CQ)** のリングを置き、コントローラと共有する。コマンド本体は DRAM にあり、デバイスはそれを DMA で読みに来る。

```
   Host DRAM                                  NVMe Controller
   ┌──────────────┐  ①コマンドをSQに書く        ┌──────────────┐
   │ Submission Q │ ─────────────────────────▶ │              │
   │   (ring)     │  ②Doorbell に MMIO write    │  処理 + DMA  │
   └──────────────┘  (尾尾ポインタ通知) ────────▶│  (データ転送)│
   ┌──────────────┐  ③完了をCQへ DMA write      │              │
   │ Completion Q │ ◀───────────────────────── │              │
   │   (ring)     │  ④MSI-X 割り込みで通知 ◀──── └──────────────┘
   └──────────────┘  ⑤ホストが CQ を処理し Doorbell 更新
```

SATA/AHCI が事実上1キュー・キュー深度 32（NCQ）なのに対し、NVMe は最大 64K キュー × 各 64K エントリを規格上許す。**キューをコアごとに割り当て**ることでロック競合を避け、フラッシュの内部並列（複数 die）を飽和させられる。

## 技術詳細 (Technical Details)

### 1. FTL とアドレス間接化 (logical-to-physical mapping)

FTL は LBA → 物理ページアドレス (PPA) の写像表を持ち、これを RAM（コントローラ内 DRAM や SRAM）に保持する。上書き要求が来ると、**元の場所を消さずに別の空きページへ書き、写像を新しい位置に張り替える**（out-of-place update / コピーオンライトに近い）。元ページは「無効 (invalid)」と印を付けるだけ。

```
# 例示 (illustrative)。実FTLのデータ構造ではなく概念を示す疑似コード。
def write(lba, data):
    ppa_new = allocate_free_page()      # 空きページを確保（消去済み）
    program(ppa_new, data)              # 物理書き込み（上書きではない）
    old = L2P[lba]
    L2P[lba] = ppa_new                  # 写像を張り替え
    if old is not None:
        mark_invalid(old)               # 旧ページは無効化（即消去はしない）
    if free_pages() < THRESHOLD:
        garbage_collect()               # 空き枯渇でGC起動
```

- **写像粒度**: ページ単位マッピング（柔軟だが表が巨大）と、ブロック単位、両者の混成 (hybrid) がある。表が大きいほど DRAM を食う（DRAM-less SSD は HMB でホスト DRAM を借りる）。
- **電源喪失と写像**: L2P 表は揮発 RAM 上にあるため、電源喪失で失うと全データを失いかねない。表のスナップショットとログを NAND に保存し、起動時に復元する。

### 2. ガベージコレクションと書き込み増幅 (Write Amplification)

無効ページが散在するブロックを再利用するには、**残った有効ページを別ブロックへ退避してから、ブロックを消去する**。この退避コピーがホストの書き込み以外の追加書き込みを生む。

```
   GC 前: Block A                    GC 後
   ┌─┬─┬─┬─┬─┬─┬─┬─┐               有効4ページ → Block B へコピー
   │V│I│V│I│I│V│I│V│   (V=有効, I=無効)   その後 Block A を erase（再利用可能に）
   └─┴─┴─┴─┴─┴─┴─┴─┘
```

- **書き込み増幅率 (Write Amplification Factor, WAF)** = NAND への実書き込み量 / ホストが要求した書き込み量。理想は 1.0 だが、ランダム書き込み主体・空き容量が少ないほど悪化し、数倍に達することがある。
- **WAF の悪化は摩耗を早め、性能を落とす。** 同じ寿命 P/E 予算をGCコピーが食い潰し、書き込みレイテンシにGCが割り込む（テールレイテンシ増大）。
- **オーバープロビジョニング (over-provisioning)**: ユーザに見せない予備容量を持つと、空きブロックに余裕ができ WAF が下がる。エンタープライズ SSD が公称容量を抑える理由。

### 3. ウェアレベリング (Wear Leveling)

特定ブロックばかり消去すると、そこだけ先に寿命が尽きる。FTL は P/E 回数を均すよう書き込み先を分散する。

- **ダイナミック (dynamic)**: 頻繁に書き換わる「ホット」データを、消去回数の少ないブロックへ回す。
- **スタティック (static)**: ほとんど書き換わらない「コールド」データが居座るブロックも、時々強制的に移動させ、消去回数を全体で均す。

### 4. TRIM / discard

ファイルシステムがファイルを削除しても、ブロック層が何も言わなければ SSD は「そのLBAはもう不要」と知れない。**TRIM (ATA) / Deallocate (NVMe) / discard (Linux block layer)** は「このLBA範囲は無効」とデバイスへ通知するコマンド。

- 効果: 無効と分かったページを GC が即「無効」と扱え、有効ページの退避コピーが減る → **WAF 低下・性能維持・摩耗低減**。
- TRIM がないと、デバイスは削除済みデータを「有効」と誤認し、無駄に退避コピーし続ける。

### 5. 永続性: キャッシュ・FLUSH・FUA・バリア (Durability)

デバイスには揮発ライトキャッシュ (volatile write cache) があり、「書き込み完了」を返した時点ではまだ NAND に届いていないことがある。永続化を保証するには明示が要る。

- **FLUSH (cache flush)**: それまでに受理した書き込みを不揮発メディアへ確実に落とす。`fsync()`/ジャーナルのコミットがこれを発行する。
- **FUA (Force Unit Access)**: その書き込み「だけ」をキャッシュを経由せず（または完了前に）メディアへ確実に書く指定。
- **ライトバリア (write barrier) / 順序**: ファイルシステムのジャーナリングは「ログを永続化してからデータ本体を反映」という順序を要求する。順序が崩れると電源喪失時に不整合になるため、FLUSH/FUA で順序点を作る。
- **電源喪失保護 (Power Loss Protection, PLP)**: エンタープライズ SSD はコンデンサ (capacitor) を積み、停電時に残った揮発キャッシュを NAND へ書き切る。これにより FLUSH 相当の保証を犠牲なく得られ、`fsync` を高速化できる。コンシューマ SSD には通常無い。

```
# 例示 (illustrative)。fsync が裏で要求する保証の概念。
write(fd, data)        # page cache に載るだけ。まだ揮発
fsync(fd)              # ① データをデバイスへ送出 (DMA)
                       # ② FLUSH を発行し、デバイス揮発キャッシュを NAND へ
                       # ③ 完了でようやく「永続化された」と言える
```

### 6. プロトコルとフォームファクタの整理

| 規格 | 物理/トランスポート | キュー/並列性 | 主な用途 | レイテンシ感（世代依存） |
|---|---|---|---|---|
| SATA + AHCI | SATA | 1キュー, NCQ 深度32 | 旧来のHDD/SATA SSD | ~数十〜百 µs（SSD時） |
| NVMe | PCIe (`bus-interconnect`) | 多数キュー×深いQD | 高性能 SSD | ~10 µs オーダ |
| eMMC | 組込み（半二重バス） | 浅い | スマホ/組込み（旧） | 比較的高い |
| UFS | M-PHY（全二重, シリアル） | コマンドキュー(深度) | スマホ/組込み（現） | eMMC より低レイテンシ |

- **SATA/AHCI**: 回転ディスク時代の遺産。プロトコルのオーバヘッドと単一キューがフラッシュには足枷。
- **NVMe**: PCIe 直結。コマンドが薄く、割り込みは MSI-X、完了通知は DRAM 上の CQ への DMA write。詳細は後述のシーケンス参照。
- **eMMC → UFS**: モバイル/組込み。eMMC は半二重で1コマンドずつに近いが、UFS は全二重シリアル (M-PHY) とコマンドキューイングで NVMe 的な並列性に寄る。

### 7. 性能特性の桁感 (order of magnitude)

世代・モデルで大きく変わる前提での目安。

| 指標 | HDD | SATA SSD | NVMe SSD | 永続メモリ (PMEM) |
|---|---|---|---|---|
| ランダム読みレイテンシ | ~ミリ秒 (10^-3 s) | ~100 µs | ~10 µs | ~数百 ns |
| ランダム IOPS | ~100 | ~10^4–10^5 | ~10^5–10^6 | （バイトアクセス） |
| 帯域 (シーケンシャル) | ~100–200 MB/s | ~500 MB/s | ~数 GB/s（PCIe世代依存） | ~10 GB/s 級 |

- HDD は**シーク**が支配的でランダムが極端に遅い。SSD/フラッシュはシークが無く、ランダムとシーケンシャルの差が小さい。
- それでも SSD でもシーケンシャル＞ランダム、リード＞ライトの傾向は残る（GC とプログラム時間のため）。
- NVMe の帯域は PCIe のレーン数・世代に律速される（`bus-interconnect.md`）。

### 8. 不揮発メモリ / 永続メモリ (NVDIMM, PMEM, CXL)

ストレージとDRAMの中間を埋める層。**バイト単位アクセス可能で不揮発**という、ブロックデバイスと根本的に異なる性質を持つ。

- **NVDIMM-N**: DRAM + フラッシュ + バックアップ電源（スーパーキャパシタ等）。通常は DRAM として動き、停電時に内容をフラッシュへ退避。速度は DRAM、容量は DRAM 並み。
- **永続メモリ (persistent memory, PMEM)**: メモリバスに挿し、CPU のロード/ストアで直接アクセスできる不揮発メモリ。ブロック層 (FTL/LBA) を経ずに、`mmap` した領域へ直接書く **DAX (Direct Access)** が可能。`../cpu-architecture/virtual-memory-mmu.md` の `mmap` がそのまま永続データに繋がる。
- **永続化の難所**: CPU のストアはキャッシュに留まる。永続を保証するには、キャッシュラインを書き戻す命令（`CLWB`/`CLFLUSHOPT`）＋順序付け (`SFENCE`) が要る。ブロックの FLUSH に相当する操作が、キャッシュライン粒度に降りてくる。
- **CXL メモリ (CXL.mem)**: PCIe 物理層上の CXL で、メモリ（揮発/不揮発）をデバイスとして接続し、コヒーレントに共有・拡張する。メモリ階層とストレージ階層の境界をさらに溶かす方向。

### 典型シーケンス: NVMe ブロック読み出し

`mmio-dma-interrupts.md` の「MMIO → DMA → 割り込み」の具体例として読める。

```
1. (Block層) ファイルシステムが LBA・長さ・転送先 DRAM を決定し bio を発行。
2. (Driver)  ドライバが NVMe Read コマンド（PRP/SGL でバッファ記述）を
             Submission Queue (DRAM上) に書き込む。
3. (MMIO)    SQ Doorbell レジスタへ MMIO write し、新コマンドの存在を通知。
             ← 書き込み順序が重要 → バリア（mmio-dma-interrupts.md 参照）
4. (DMA)     コントローラが SQ をDMAで読み、NAND から DRAM へデータをDMA転送。
             内部では FTL が L2P を引き、複数 die から並列に読む。
5. (DMA)     完了エントリを Completion Queue (DRAM上) へ DMA write。
6. (IRQ)     MSI-X 割り込みでホストへ完了通知。
7. (ISR)     ハンドラが CQ を読み、ステータス確認、待機プロセスを起床、
             CQ Doorbell を更新して消費位置を返す。
```

回転ディスクの I/O スケジューラ（シーク最小化のための並べ替え）が、NVMe ではほぼ不要になる点に注意（`none`/`mq-deadline` 等の軽いスケジューラが選ばれる）。

## セキュリティ考察 (Security Considerations)

- **データ残留と完全消去の困難 (data remanence)**: FTL の out-of-place 更新ゆえ、ホストが「上書き」しても旧データは別の物理ページに残る。OS レベルの上書き (overwrite) では消えない領域（予備容量・退役ブロック）が生じる。HDD 時代の「同じセクタを上書きすれば消える」前提が成り立たない。
- **サニタイズ (sanitize)**: 確実な消去にはデバイス自身のコマンドが要る。**ATA Secure Erase / NVMe Sanitize (Block Erase)** は物理ブロックを消去し、**Crypto Erase** は内部の暗号鍵を破棄してデータを瞬時に復号不能化する。後者は SED 前提だが最速・最確実。
- **自己暗号化ドライブ (SED, Self-Encrypting Drive)**: コントローラが全データを透過的に暗号化 (AES) して保存する。利点は (1) Crypto Erase による瞬時無効化、(2) 鍵を解錠しない限りデータ無効。**TCG Opal** が標準。注意点として、認証・鍵管理の実装欠陥（鍵がパスワードに依存しない、ファームの解錠バイパス等）が過去に報告されており、SED 任せにせずソフト暗号 (dm-crypt/BitLocker) との多層化が堅い。
- **ファームウェアと信頼境界**: ストレージコントローラは強力な DMA エンジンを持つ。悪意ある／脆弱なファームは DMA でホストメモリを侵せる。`mmio-dma-interrupts.md` の DMA 攻撃と同根で、IOMMU 隔離とファーム署名検証が前提。
- **デバイス入力の検証**: 完了エントリやログ長など、デバイスから来る値は信頼できない入力として扱う（長さ取り違えはオーバーフローに直結）。

## 結論 (Conclusions)

- ストレージはメモリ階層の不揮発な最下層であり、NANDフラッシュの「ページ書き/ブロック消去/上書き不可」という非対称な物理制約が、上位のすべての複雑さの源である。
- FTL がこの制約を隠蔽し、ウェアレベリング・GC・TRIM で寿命と性能を保つ。「使ううちに遅くなる」「書いた以上に摩耗する」は書き込み増幅 (WAF) の帰結として理解できる。
- プロトコルの進化（SATA/AHCI → NVMe、eMMC → UFS）の本質は**並列性とキュー**であり、フラッシュ本来の性能とマルチコアを引き出すための更新である。
- 永続性は「書いた」と「残った」の差で決まり、FLUSH/FUA/バリアと PLP がそれを律する。`fsync` が重いのも、停電でデータが消える/前後するのも、この層の理解で説明できる。
- 永続メモリ (PMEM/CXL) はブロック抽象を飛び越え、永続化の問題をキャッシュライン粒度へ押し下げる。メモリ階層とストレージ階層の境界は今後さらに溶ける。
- FTL の間接化ゆえに「完全消去」は上書きでは達成できず、サニタイズ／Crypto Erase が要る。これがストレージ特有のセキュリティ上の要点である。

### 次の接続先 (Related Topics)

- `mmio-dma-interrupts.md` — NVMe の Doorbell(MMIO)・データ転送(DMA)・MSI-X 完了通知(割り込み)の機構。
- `bus-interconnect.md` — NVMe が載る PCIe と、CXL の物理層。
- `../cpu-architecture/cache-memory-hierarchy.md` — ストレージ階層の最下層としての位置づけ（速度・容量・コストの桁感）。
- `../cpu-architecture/virtual-memory-mmu.md` — `mmap`/page cache と、PMEM の DAX 直接アクセス。

## 参考文献 (References)

**一次資料 (Primary — 仕様・論文)**

- NVM Express, Inc. *NVM Express Base Specification*. — [nvmexpress.org/specifications](https://nvmexpress.org/specifications/) — Accessed: 2026-07-02。送信/完了キュー・コマンドセットの規範。
- ONFI Workgroup. *Open NAND Flash Interface Specification*. — [onfi.org](https://onfi.org/) — Accessed: 2026-07-02。NAND デバイスインタフェースの規範。
- Agrawal, N. et al. (2008). Design Tradeoffs for SSD Performance. *USENIX Annual Technical Conference*. — [PDF](https://www.usenix.org/legacy/event/usenix08/tech/full_papers/agrawal/agrawal.pdf) — Accessed: 2026-07-02。FTL・書き込み増幅・並列性のトレードオフを扱う代表的論文。
