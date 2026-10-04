# バスと相互接続: PCIe・AMBA/AXI・CXL (Bus & Interconnect: PCIe, AMBA/AXI, CXL)

## 概要 (Overview)

`mmio-dma-interrupts.md` が扱った MMIO・DMA・割り込み (interrupt) は、「何が」CPU とデバイスの間を流れるかを定めた。本ドキュメントはその**物理経路**——MMIO のロード／ストア、DMA のデータ、MSI のメッセージが、実際にどの配線・プロトコルの上を流れて相手に届くか——を扱う。すなわち**バス (bus)** と**相互接続 (interconnect)** である。

主役は3つ。チップ間 (off-chip) を結ぶ **PCIe (PCI Express)**、SoC 内 (on-chip) を結ぶ **AMBA (Advanced Microcontroller Bus Architecture)** とその主役 **AXI (Advanced eXtensible Interface)**、そして PCIe の物理層の上にコヒーレンシ (coherence) を載せてメモリとアクセラレータを束ねる **CXL (Compute Express Link)**。

これらに共通する歴史的潮流が、**共有バス (shared bus) から点対点 (point-to-point) シリアルリンクへの移行**である。本ドキュメントはこの移行の理由を起点に、各相互接続のパケット・列挙 (enumeration)・チャネル・コヒーレンシを整理し、トポロジ (topology) と QoS／アービトレーション (arbitration)、帯域とレイテンシのトレードオフへと接続する。数値は世代・実装で変動するため、絶対値ではなく**桁感 (order of magnitude)** として扱う。

## 方法論 (Methodology)

- 相互接続を「物理層 (PHY)・リンク層・トランザクション層・プロトコル／意味論」の層に分解し、各層が何を保証するかを対応づける。
- スケール軸で3つを配置する: チップ間 = PCIe、チップ内 = AMBA/AXI、両者にまたがるコヒーレント拡張 = CXL。
- 設計判断（共有 vs 点対点、パラレル vs シリアル、メモリマップ vs パケット）について、何を得て何を失うかをトレードオフとして示す。
- `mmio-dma-interrupts.md` の MMIO／DMA／MSI を「バス上を流れる荷物」として再解釈し、本ドキュメントの「経路」と接続する。

## 発見事項 (Findings)

1. **歴史は「共有パラレルバス」から「点対点シリアルリンク」へ一方向に進んだ。** 共有バス（旧 PCI、フロントサイドバス）は配線を共有するため、スキュー (skew)・アービトレーション・容量負荷が周波数を頭打ちにした。PCIe は1対1の差動シリアルレーン (differential serial lane) を束ね、各リンクを独立にクロックすることでスケールさせた。同じ力学が SoC 内でも働き、共有バスは AXI のような点対点・スイッチ接続へ移った。

2. **現代の相互接続は「パケット交換ネットワーク」である。** PCIe の TLP (Transaction Layer Packet)、AXI の独立チャネル、NoC (Network-on-Chip) のフリット (flit)——いずれもアドレス＋データを**パケット**として運び、スイッチ／ルータで経路制御する。バスというより小さなネットワークと見るのが正確。

3. **「列挙と発見 (enumeration & discovery)」が動的な接続を可能にする。** PCIe ではコンフィグ空間 (configuration space) と BAR (Base Address Register) を介して、OS が起動時にデバイスを探索し、アドレス空間を割り当てる。`mmio-dma-interrupts.md` の MMIO アドレスは、この列挙の結果として決まる。

4. **オンチップとオフチップは設計圧力が異なる。** オフチップ (PCIe/CXL) はピン数とボード配線が制約で、シリアル化と高速 SerDes が要。オンチップ (AXI) はトランジスタが潤沢で配線が短いため、広いパラレルバスと多チャネル並行が許される。最適解が層で違う。

5. **CXL はバスをコヒーレンシ境界に変える。** 従来 PCIe デバイスは非コヒーレントで、DMA とキャッシュ整合は明示管理だった（`mmio-dma-interrupts.md` 参照）。CXL は PCIe 物理層の上に `cxl.cache`／`cxl.mem` を載せ、デバイスをキャッシュコヒーレンシ領域に引き入れる。これは `../cpu-architecture/cache-coherence.md` のコヒーレンシをチップ外へ拡張する動きである。

## アーキテクチャ (Architecture)

### スケールで見た3つの相互接続

```
   ┌──────────────────────── SoC / Package ────────────────────────┐
   │                                                                │
   │   ┌─────┐   ┌─────┐         AMBA AXI (on-chip interconnect)    │
   │   │CPU0 │   │CPU1 │              点対点 + スイッチ              │
   │   └──┬──┘   └──┬──┘                                            │
   │      │  AXI    │  AXI                                          │
   │   ┌──▼─────────▼──────────────────────────────┐               │
   │   │        Interconnect / NoC (crossbar)        │              │
   │   └──┬───────────┬───────────────┬─────────────┘               │
   │   ┌──▼──┐    ┌───▼───┐       ┌───▼────┐                        │
   │   │ DRAM│    │  SRAM │       │ PCIe RC│ ← Root Complex         │
   │   │ Ctrl│    │       │       │ /CXL   │                        │
   │   └─────┘    └───────┘       └───┬────┘                        │
   └─────────────────────────────────┼─────────────────────────────┘
                                      │  PCIe/CXL link (off-chip,
                                      │  差動シリアルレーン x N)
                  ┌───────────────────┼───────────────────┐
                  │                   │                   │
             ┌────▼────┐         ┌────▼────┐         ┌─────▼─────┐
             │  Switch │         │  GPU    │         │ CXL Mem   │
             │ (PCIe)  │         │ (EP)    │         │ Expander  │
             └──┬───┬──┘         └─────────┘         └───────────┘
             ┌──▼┐ ┌▼──┐
             │NIC│ │NVMe│      EP = Endpoint, RC = Root Complex
             └───┘ └────┘
```

要点: チップ内は AXI を束ねた相互接続／NoC、チップの境界に **Root Complex (RC)** があり、そこから PCIe／CXL の差動シリアルリンクが外部の **Endpoint (EP)** やスイッチへ伸びる。論理的には RC を根とする**ツリー (tree)** である。

### PCIe の階層 (Protocol Layers)

```
   ┌─────────────────────────────────────────────┐
   │  Software / Device Driver                     │ ← MMIO/DMA を発行
   ├─────────────────────────────────────────────┤
   │  Transaction Layer   → TLP (header+data)      │ アドレス/データの意味
   │     (MRd/MWr, CfgRd/CfgWr, MSI 書き込み)       │
   ├─────────────────────────────────────────────┤
   │  Data Link Layer     → DLLP, ACK/NAK, 再送     │ 信頼性・フロー制御
   │     (シーケンス番号 + LCRC)                     │
   ├─────────────────────────────────────────────┤
   │  Physical Layer      → 128b/130b 符号, レーン   │ 差動シリアル, SerDes
   │     (Gen3+ のエンコード, リンク幅 x1..x16)       │
   └─────────────────────────────────────────────┘
```

PCIe は3層構造で、上から TLP（意味）→ DLLP と再送（信頼性）→ 物理レーン（電気）と降りる。`mmio-dma-interrupts.md` の MMIO 書き込みは Memory Write TLP に、DMA は Endpoint 起点の Memory Read/Write TLP に、MSI/MSI-X は特定アドレスへの Memory Write TLP に対応する。

### AMBA: AXI / AHB / APB の役割分担

```
   高性能・高帯域 ──────────────────────────▶ 低速・低消費電力・単純
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │   AXI    │      │   AHB    │      │   APB    │
   │ 5チャネル │ ───▶ │ 単一パイプ│ ───▶ │ 単純周辺 │
   │独立・並行 │      │ライン     │      │レジスタ   │
   │burst/OST │      │burst対応  │      │1転送/1サイクル系│
   └──────────┘      └──────────┘      └──────────┘
    CPU/DMA/GPU       中速ペリフェラル    UART/GPIO/タイマ
```

同じ SoC 内でも、帯域が要る経路は AXI、軽い制御レジスタは APB、と適材適所で使い分ける。ブリッジ (bridge) で階層をまたぐ。

## 技術詳細 (Technical Details)

### 1. PCIe: レーン・世代・帯域 (Lanes, Generations, Bandwidth)

- **レーンとリンク幅 (lane & link width)**: 1レーン = 送受1対の差動ペア (TX/RX)。リンクはレーンを束ねて `x1, x4, x8, x16` を構成し、データはレーン間にストライプ (stripe) される。GPU は `x16`、NVMe SSD は `x4` が典型。
- **世代と帯域 (generation)**: 世代ごとに転送レートがおよそ倍化してきた（桁感; 実装・世代依存）。

  | 世代 | レーンあたり転送レート | エンコード | x16 片方向帯域(概算) |
  |---|---|---|---|
  | Gen3 | ~8 GT/s | 128b/130b | ~16 GB/s |
  | Gen4 | ~16 GT/s | 128b/130b | ~32 GB/s |
  | Gen5 | ~32 GT/s | 128b/130b | ~64 GB/s |
  | Gen6 | ~64 GT/s | PAM4 + FEC | ~128 GB/s |

  GT/s（giga-transfers/s）は生のシンボルレートで、実効バイト帯域はエンコード効率（Gen3〜5 は約 128/130 ≈ 98.5%）を掛けて求める。Gen6 は信号方式が NRZ から **PAM4**（1シンボル2ビット）へ変わり、誤り訂正 (FEC) を伴う。上表は片方向 (per-direction) の概算で、PCIe は全二重 (full-duplex)。

- **TLP (Transaction Layer Packet)**: PCIe トランザクションの基本単位。ヘッダ（型・アドレス・要求元 ID・長さ）＋任意のデータ＋エンドツーエンド保護。主な型は Memory Read/Write、Configuration Read/Write、Completion。**MMIO はメモリ空間 TLP、列挙はコンフィグ空間 TLP** として流れる。
- **列挙とコンフィグ空間 (enumeration & configuration space)**: 起動時、ファームウェア／OS が `bus:device.function` を深さ優先に走査し、各機能のコンフィグ空間（先頭 256B、PCIe 拡張で 4KB）を読んでデバイスを発見する。Vendor/Device ID で識別し、**BAR (Base Address Register)** に物理アドレス範囲を割り当てる。
- **BAR**: デバイスが「自分のレジスタ／メモリにこれだけの空間が要る」とサイズを申告し、OS が空きアドレスを割り付ける。`mmio-dma-interrupts.md` でドライバが触る MMIO ベースアドレス `base` は、この BAR の割り当て結果である。
- **MSI / MSI-X**: 物理割り込み線の代わりに、あらかじめ設定したアドレスへ Memory Write TLP を送って割り込みを通知する（`mmio-dma-interrupts.md` の MSI と同一機構を、ここでは経路側から見ている）。MSI-X はベクタごとに宛先アドレス・データ・マスクを持つテーブルを備え、多数ベクタと柔軟な CPU 振り分けを可能にする。

```
// PCIe コンフィグ空間ヘッダの先頭（例示 illustrative; レイアウトは規格準拠の概略）
offset 0x00:  Vendor ID (16b) | Device ID (16b)
offset 0x04:  Command (16b)   | Status (16b)
offset 0x08:  Revision(8b) | Class Code (24b)
offset 0x10:  BAR0  ← MMIO/IO ベースアドレス（サイズ申告→OSが割当）
offset 0x14:  BAR1
   ...        BAR2..BAR5
offset 0x34:  Capabilities Pointer → MSI/MSI-X, PCIe, PM 等の能力リスト
```

```c
// 列挙の骨子（例示コード illustrative。リポジトリ内の実コードではない）
for (bus = 0; bus < 256; bus++)
  for (dev = 0; dev < 32; dev++)
    for (fn = 0; fn < 8; fn++) {
        u32 id = cfg_read32(bus, dev, fn, 0x00);     // Vendor/Device ID
        if ((id & 0xFFFF) == 0xFFFF) continue;       // 0xFFFF = 不在
        size = bar_probe(bus, dev, fn, 0x10);        // BAR にサイズ問い合わせ
        assign_mmio_window(bus, dev, fn, 0x10, size);// 空きアドレスを割当
        if (is_bridge(bus,dev,fn)) recurse(secondary_bus); // ブリッジ下を再帰
    }
```

### 2. AMBA/AXI: チャネルとハンドシェイク (Channels & Handshake)

AXI の特徴は、読み・書きを**5つの独立チャネル**に分け、各チャネルが `VALID`/`READY` の対で握手 (handshake) することにある。送り手が `VALID`、受け手が `READY` を立て、両方が高い同一クロックで1転送が成立する。

| チャネル | 略 | 向き | 内容 |
|---|---|---|---|
| Write Address | AW | master→slave | 書き込み先アドレス・バースト属性 |
| Write Data | W | master→slave | 書き込みデータ（複数ビート） |
| Write Response | B | slave→master | 書き込み完了応答 |
| Read Address | AR | master→slave | 読み出しアドレス・バースト属性 |
| Read Data | R | slave→master | 読み出しデータ＋応答 |

- **アドレスとデータの分離**: アドレスチャネルとデータチャネルが独立なので、後続要求のアドレスを先に投入でき、パイプライン化される。読みと書きも別チャネルゆえ同時進行できる。
- **バースト転送 (burst)**: 1つのアドレス発行で連続する複数ビート (beat) を転送する。種別は `FIXED`（同一アドレス, FIFO 向け）・`INCR`（増加, メモリ向け）・`WRAP`（キャッシュラインフィル向け、境界で巻き戻る）。`../cpu-architecture/cache-memory-hierarchy.md` のキャッシュラインフィルは `WRAP` バーストに対応する。
- **アウトスタンディング転送 (outstanding transactions)**: 応答を待たずに複数の要求を投入できる。各要求に **ID** を付け、応答を ID で対応づける。これがレイテンシ隠蔽の鍵で、メモリ遅延の間も次々と要求を出せる。
- **ordering（順序保証）**: 同一 ID の転送は発行順に完了するが、**異なる ID の転送は順不同で完了してよい**。これにより低速スレーブが高速スレーブをブロックしない (out-of-order completion)。順序が必要なら同一 ID を使うか、バリアで律する。MMIO レジスタの順序依存（`mmio-dma-interrupts.md` の「開始ビットは最後」）は、この順序規則の上で守る必要がある。

```
// AXI 書き込みバーストの握手（例示 illustrative; 4ビート INCR）
   AW: ──[ADDR=A, LEN=4, BURST=INCR, ID=3]──▶        (VALID&READY で成立)
   W : ──[D0]──[D1]──[D2]──[D3,LAST]──▶              (各ビート毎に握手)
   B : ◀──────────────[RESP=OKAY, ID=3]──            (完了応答, ID で対応)
```

### 3. CXL: PCIe 物理層上のコヒーレント拡張 (Coherent Extension)

CXL は PCIe の物理層・電気仕様を共有しつつ、上に3つのサブプロトコルを多重化する (CXL は PCIe コネクタ／PHY 上で動作し、リンク確立時に折衝する; 詳細は世代・実装依存)。

- **`cxl.io`**: 実質 PCIe 互換の I/O。列挙・コンフィグ・割り込みなど、従来 PCIe と同じ役割（後方互換の足場）。
- **`cxl.cache`**: デバイスがホストメモリをコヒーレントにキャッシュできる。アクセラレータがホスト DRAM を、明示 flush/invalidate なしに一貫した状態で読み書きできる。
- **`cxl.mem`**: ホストがデバイス側メモリを、自分のメモリ空間の一部としてコヒーレントにアクセスする。**メモリ拡張 (memory expansion)** の中核——CXL メモリエクスパンダ (memory expander) を挿せば、DRAM 容量・帯域をスロット経由で増設できる。

これらの組み合わせで、CXL は3タイプのデバイスを定義する（桁感の整理; 規格上の Type 1/2/3 に対応）。

| タイプ | 使うプロトコル | 典型例 |
|---|---|---|
| Type 1 | `cxl.io` + `cxl.cache` | コヒーレントキャッシュを持つアクセラレータ（NIC 等） |
| Type 2 | `cxl.io` + `cxl.cache` + `cxl.mem` | 自前メモリを持つアクセラレータ（GPU/FPGA） |
| Type 3 | `cxl.io` + `cxl.mem` | メモリエクスパンダ（容量・帯域拡張） |

**コヒーレンシとの接続**: 従来 PCIe DMA は非コヒーレントで、`mmio-dma-interrupts.md` の通り CPU 側が flush/invalidate を明示管理した。CXL はこれをハードウェアのコヒーレンシプロトコルに肩代わりさせ、ホストとデバイスが MESI 系（`../cpu-architecture/cache-memory-hierarchy.md` 参照）の一貫したビューを共有する。ホストがコヒーレンシの調停点 (home agent) となり、デバイスのキャッシュ状態を追跡する。これは**チップ内コヒーレンシをパッケージ外へ延長する**動きであり、`../cpu-architecture/cache-coherence.md` が扱う領域がボードレベルへ広がることを意味する。

### 4. トポロジ: ツリー / メッシュ / NoC (Topology)

- **ツリー (tree)**: PCIe の論理構造。Root Complex を根に、スイッチを内部ノード、Endpoint を葉とする階層。発見と経路が単純だが、根に近いリンクが集約点 (aggregation point) でボトルネックになりうる。
- **メッシュ / リング (mesh / ring)**: 多コア CPU のオンチップ相互接続。コアと LLC スライス、メモリコントローラを格子状／環状に結ぶ。スケーラブルだがホップ数 (hop count) でレイテンシが伸びる。
- **NoC (Network-on-Chip)**: SoC 内をルータとリンクの網で構成し、パケット（フリット flit に分割）を経路制御する。多数の IP ブロックを擁する大規模 SoC で、点対点配線の組合せ爆発を避ける。

```
   ツリー (PCIe)              メッシュ (multicore NoC)
       RC                     C─R─C─R─C─R
      / \                     │ │ │ │ │ │
    SW   EP                   R─R─R─R─R─R     R=router
   /  \                       │ │ │ │ │ │     C=core/IP
  EP   EP                     C─R─C─R─C─R
```

### 5. QoS とアービトレーション (QoS & Arbitration)

複数のマスタ（CPU・GPU・DMA・ディスプレイ）が同じ相互接続を共有すると、帯域の奪い合いが起きる。**アービトレーション (arbitration)** がアクセス権を調停する。

- **方式**: ラウンドロビン (round-robin, 公平)、固定優先度 (fixed priority, 遅延に厳しいマスタ優先)、重み付き／クレジットベース (credit-based, 帯域配分)。
- **QoS (Quality of Service)**: AXI には QoS 信号があり、遅延に敏感なマスタ（例: ディスプレイのフレームバッファ読み出し、アンダーフローすると画面が乱れる）を優先できる。リアルタイム保証と平均帯域のバランスを取る。
- **フロー制御 (flow control)**: PCIe はクレジットベースのフロー制御を持ち、受け手のバッファ空きをクレジットで通知して送り過ぎを防ぐ。これが「アウトスタンディング数」の上限を実質規定する。

### 6. 帯域とレイテンシのトレードオフ (Bandwidth vs Latency)

- **シリアル化のコスト**: 点対点シリアルは配線を減らし周波数を上げられるが、パラレル→シリアル→パラレル変換 (SerDes) とパケット化のオーバヘッドでレイテンシが増える。短距離・低遅延が要る経路（CPU↔L1 など）はパラレルのまま、長距離・高帯域が要る経路（チップ間）はシリアル、と棲み分ける。
- **バースト長と効率**: ヘッダは固定コストなので、バーストを長くするほどペイロード効率（実効帯域）が上がる。一方で長いバーストは他マスタを待たせ、レイテンシ公平性を損なう。QoS と相反する。
- **アウトスタンディングと帯域**: レイテンシ × 帯域 = 必要な「飛行中 (in-flight)」のデータ量（帯域遅延積）。これを満たすだけのアウトスタンディング転送数・クレジットがないと、リンクが太くてもレイテンシで律速され帯域を使い切れない。

## セキュリティ考察 (Security Considerations)

- **DMA 攻撃と IOMMU**: バスマスタリング (bus mastering) するデバイスは、`mmio-dma-interrupts.md` で述べた通り任意の物理メモリへ DMA しうる。PCIe/Thunderbolt の外部接続では特に危険で、**IOMMU** がデバイスごとにアドレス変換・保護（バス上の DMA の宛先制限）を課すのが防御の要。本ドキュメントの視点では、IOMMU は「バス上を流れる DMA TLP の宛先を検査する関所」である。
- **PCIe の信頼境界 (trust boundary)**: 列挙で発見したデバイスは無条件に信頼できない。悪意あるデバイスは偽の BAR サイズ・不正な TLP・過剰な MSI を送りうる。**ACS (Access Control Services)** はスイッチで P2P（ピア間）トランザクションを制御し、デバイス同士が IOMMU を迂回して直接通信するのを防ぐ。BAR・コンフィグ由来の値（長さ・アドレス）は信頼できない入力として検証する。
- **CXL のメモリ共有の境界**: CXL でデバイスがホストメモリをコヒーレントに触れる、あるいはホストがデバイスメモリを使うということは、**コヒーレンシ境界がそのまま信頼境界になる**ことを意味する。共有メモリの完全性・機密性を守るため、リンク暗号化・完全性保護 (IDE: Integrity and Data Encryption に相当する機構) と、デバイス／メモリ範囲ごとのアクセス制御が要る。複数ホストでメモリをプールする構成では、ホスト間の隔離 (isolation) が崩れると情報漏洩・改ざんに直結する。
- **物理層・リンク層の攻撃**: インターポーザ等でリンクに割り込む物理攻撃に対し、リンク暗号化が機密性・完全性を担保する。再送・LCRC は故障耐性のためであり、敵対的改ざんには暗号学的保護が別途必要。

## 結論 (Conclusions)

- バスと相互接続は、MMIO・DMA・割り込みが流れる**物理経路**であり、その歴史は共有パラレルバスから点対点シリアルリンク（＝小さなパケット交換ネットワーク）への一方向の移行で説明できる。
- PCIe はチップ間をツリー状に結び、TLP・コンフィグ空間・BAR・MSI/MSI-X で「発見と通信」を成立させる。`mmio-dma-interrupts.md` の MMIO アドレスは、この列挙が割り当てた BAR の結果である。
- AMBA/AXI はチップ内を5チャネルの独立握手・バースト・アウトスタンディング転送で結び、ID ベースの順序規則でレイテンシを隠蔽する。
- CXL は PCIe 物理層の上にコヒーレンシを載せ、`../cpu-architecture/cache-coherence.md` のコヒーレンシ領域をパッケージ外（メモリ拡張・アクセラレータ）へ拡張する。これにより DMA のキャッシュ整合の明示管理が不要になる代わりに、コヒーレンシ境界が新たな信頼境界となる。
- 設計はつねに帯域とレイテンシ、効率と公平性のトレードオフであり、トポロジ・QoS・アービトレーション・クレジットがその調停手段である。

### 次の接続先 (Related Topics)

- `mmio-dma-interrupts.md` — 本ドキュメントのバス上を実際に流れる MMIO・DMA・MSI の中身。
- `../cpu-architecture/cache-coherence.md` — CXL が `cxl.cache`/`cxl.mem` で拡張する、コヒーレンシ領域そのもの。
- `../cpu-architecture/cache-memory-hierarchy.md` — バースト／ラインフィルとコヒーレンシ状態 (MESI) の背景。

## 参考文献 (References)

**一次資料 (Primary — 仕様)**

- PCI-SIG. *PCI Express Base Specification*. — [pcisig.com/specifications](https://pcisig.com/specifications) — Accessed: 2026-07-02。レーン・世代・トランザクション/データリンク/物理層の規範。
- Arm. *AMBA AXI Protocol Specification* (IHI 0022). — [developer.arm.com](https://developer.arm.com/documentation/ihi0022/latest) — Accessed: 2026-07-02。AXI チャネルとハンドシェイクの規範。
- CXL Consortium. *Compute Express Link Specification*. — [computeexpresslink.org](https://computeexpresslink.org/cxl-specification/) — Accessed: 2026-07-02。CXL.io/.cache/.mem の規範。
