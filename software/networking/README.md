# Networking

ネットワーキングとプロトコル (networking & protocols) の分析。「誰も全体を管理していない系」
での分散制御 (輻輳制御) と、層をまたぐ設計判断 (HoL ブロッキング・硬直化・暗号化の範囲) を軸に、
リンク層 (LAN) からトランスポート、アプリケーションプロトコルまでを読み解く。
[`../distributed-systems/`](../distributed-systems/) の故障モデル (ロス・遅延・分断) の発生源で
あり、[`../../hardware/networking/`](../../hardware/networking/)（NIC・スイッチ ASIC）と
[`../../hardware/hw-sw-interface/`](../../hardware/hw-sw-interface/)（割り込み・DMA）の真上の層。

## 構成

- `ethernet-lan-switching.md` — LAN とスイッチング：Ethernet フレームと MAC アドレス
  (最小 64B の由来・ランダム化)、学習ブリッジとユニキャストの退化、ARP とアドレス衝突検出、
  VLAN (802.1Q) とネイティブ VLAN、STP/RSTP/MSTP (TTL なき層のループ対策と収束)、
  リンク集約と L3 化、VRRP によるゲートウェイ冗長、L2 の攻撃と防御 (ARP スプーフィング・
  VLAN ホッピング・DAI・802.1X・MACsec)。以下すべての文書が前提とする最下層
- `ip-addressing-icmp-dhcp-nat.md` — IP の基礎：IPv4 ヘッダと TTL、アドレッシングと CIDR
  (サブネット計算・特別用途アドレス・枯渇の経緯)、断片化と経路 MTU 探索 (PMTUD ブラックホール・
  RFC 8900)、ICMP と ICMPv6、DHCP のリースとリレー、NAT/NAPT (RFC 4787 の挙動要件・
  CGN・NAT 越え)、IPv6 (SLAAC・一時アドレス・移行技術・普及状況)、送信元偽装と BCP 38
- `email-delivery-authentication.md` — メールの配送と送信者認証：SMTP の蓄積転送と MX、
  エンベロープとヘッダの 3 種類の差出人、SPF・DKIM・DMARC (RFC 9989 とアライメント、転送と
  メーリングリストの衝突、ARC)、STARTTLS の格下げと MTA-STS/DANE、S/MIME・OpenPGP と EFAIL、
  大手事業者の送信者要件、SMTP smuggling と BEC
- `vpn-tunneling.md` — VPN とトンネリング：包む層による分類、IPsec (SPD/SA、ESP と AH、
  IKEv2、NAT トラバーサル)、SSH (TOFU、ポート転送、Terrapin、regreSSHion)、WireGuard
  (cryptokey routing、Noise IK、応答しない設計)、L2TP と PPTP、TLS の VPN、MTU と経路表、
  TunnelVision・TunnelCrack と VPN 装置の侵害、メッシュ型 VPN (調整サーバと NAT 越え)、
  耐量子鍵交換への移行 (SSH・IKEv2・WireGuard)、VPN の識別と検閲、一般向け VPN サービスの信頼モデル
- `proxies-anonymity-censorship.md` — プロキシ・匿名化・検閲：誰がどのフィールド (IP・SNI・DNS の名前・
  大きさとタイミング) を見て誰を信頼するかという共通の枠組み、フォワード・リバース・透過型のプロキシと
  CONNECT・SOCKS・PAC/WPAD、X-Forwarded-For・Forwarded・PROXY protocol の偽装、知識を分ける中継
  (iCloud Private Relay・OHTTP・ODoH・廃止された Chrome の IP Protection)、Tor (回路・ディレクトリ機関・ガード・
  オニオンサービス v3 と proof-of-work・規模・Arti)、相関攻撃・RAPTOR・Web サイトの指紋の評価論争・悪意ある中継、
  層ごとの検閲 (DNS の注入・RST・SNI・帯域制限・QUIC の Initial の復号・完全に暗号化された通信の推定・
  能動的な調査・ECH の遮断・国全体の遮断)、GFW・TSPU・イランの配置、OONI・Censored Planet・ICLab と計測の倫理、
  迂回の設計 (obfs4・meek・Snowflake・WebTunnel・Shadowsocks・模倣と乱数化・リフラクションネットワーキング・
  ドメインフロンティングの衰退)、DNS・WebRTC の漏れと TLS の指紋。散らばっていた検閲とプライバシの話題の入口
- `tcp-transport-congestion.md` — TCP と輻輳制御：信頼性の道具立て (ACK/SACK/RTO)、
  フロー制御 vs 輻輳制御、接続管理 (SYN cookie・TIME_WAIT とポート再利用・RFC 5961・keepalive)、
  TCP Fast Open が広まらなかった理由、損失回復 (RTO の最小値・RACK-TLP・PRR・F-RTO・遅延 ACK と Nagle)、
  ウィンドウスケール・PAWS・バッファの自動調整、AIMD の公平性 (Chiu–Jain)、CUBIC、
  バッファブロートと AQM/ECN・AccECN、BBR (v1 の実装・公平性の実測・BBRv3 の状況)、
  データセンターの輻輳制御への橋渡し (DCTCP・Swift・HPCC・PLB)、MPTCP と結合輻輳制御、
  `ss -ti`・tracepoint による観測、TCP の脆弱性 (CVE-2016-5696・SACK Panic・TCP-AO)、
  TCP の構造的限界 (HoL・硬直化)
- `qos-traffic-management.md` — QoS とトラフィック制御：QoS は容量を作らず混雑時の配分を
  決めるだけ、IntServ/RSVP から DiffServ へ、DSCP と PHB (EF・AF・LE・NQB)、信頼境界と
  インターネットでの DSCP の書き換え、スケジューリング (厳密優先・WFQ・DRR)、ポリシングと
  シェーピング (3 色マーカ・TCP との相性)、階層型 QoS と Linux tc (HTB・fq_codel・CAKE)、
  L4S・5QI・PFC、優先度の窃取とネットワーク中立性
- `http-quic-evolution.md` — HTTP/1.1 → 2 → QUIC/HTTP3：多重化と HoL の局在化、
  HPACK/QPACK、意味論とワイヤ形式の分離 (RFC 9110–9114)、冪等性・キャッシュ・条件付きと範囲の
  リクエスト、仲介者と CONNECT・Extended CONNECT、Server Push の撤去と 103 Early Hints・
  RFC 9218 の優先度・WebSocket/SSE、リクエストスマグリング (H2.CL/H2.TE・0.CL)、
  QUIC の設計 (ストリーム、コネクション ID、トランスポート暗号化、0-RTT とリプレイ、アンプ制限)、
  損失検出と輻輳制御 (RFC 9002)、経路移動とステートレスリセット、QUIC v2・GREASE・スピンビット、
  DATAGRAM とマルチパス、UDP GSO/GRO と CPU のコスト、普及率と UDP 遮断時のフォールバック、
  MASQUE と iCloud Private Relay、硬直化への構造的応答
- `webrtc-realtime-media.md` — WebRTC とリアルタイムメディア伝送：遅延予算 (G.114) と「遅れたデータは
  失われたデータ」、ICE・STUN・TURN と trickle ICE・mDNS 候補、DTLS-SRTP と指紋、SCTP のデータチャネル、
  JSEP と Unified Plan、RTP/RTCP とジッタバッファ・NACK/RTX・FEC (FlexFEC・RED)・PLI/FIR、
  GCC・transport-wide CC・SCReAM・NADA と RMCAT の要件、メッシュ・MCU・SFU とサイマルキャスト・SVC、
  SFrame と MLS、WebTransport・MoQ・WHIP/WHEP、TURN の悪用・IP アドレスの漏洩・cryptex
- `kernel-network-stack.md` — カーネルネットワークスタック：NIC リングと NAPI (割り込みの
  経済)、sk_buff、GRO/GSO とパケット単価、RSS/RPS/RFS、qdisc/BQL、XDP・AF_XDP・DPDK の
  バイパス系譜。NAPI の予算 (`netdev_budget`)・スレッド化 NAPI・busy poll と IRQ suspension、
  page_pool とヘッダ分割、ソケットのメモリ計上 (`tcp_mem`)・listen のキューと SYN cookie・
  SO_REUSEPORT・epoll・io_uring、ゼロコピー (MSG_ZEROCOPY・devmem TCP・io_uring zcrx・kTLS)、
  FIB の LC-trie と経路キャッシュの削除・近隣表、EDT と TSQ、netns・veth、drop reason と
  落ちる場所の地図、netmap・mTCP・Snap とバイパスの再評価、SegmentSmack・FragmentSmack・
  SACK Panic。上 2 本のプロトコルを運ぶ機械側
- `packet-filtering-conntrack.md` — パケットフィルタと接続追跡：netfilter のフックと優先度、
  iptables から nftables へ (線形照合と全体更新 → 集合・verdict map・トランザクション、
  iptables-nft、ipset)、conntrack の状態機械 (TCP の窓の検査、UDP/ICMP の擬似状態、タイムアウト)、
  表の大きさと枯渇・ゾーン、helper の既定無効化の経緯、conntrack の上の NAT (ポート割り当て)、
  非対称経路と状態の同期 (conntrackd・pfsync)、pf・eBPF・flowtable との対比、
  NAT Slipstreaming と IPv6 の見落とし。上の文書のホストの中で、規則と状態を実装する部分
- `ddos-mitigation.md` — DDoS 対策のアーキテクチャ：容量の配置と早く安い廃棄、費用の非対称、
  BCP 38/84 と反射の踏み台の閉鎖 (monlist・memcached・RRL・QUIC の 3 倍の上限)、RTBH (RFC 5635・
  BLACKHOLE コミュニティ) が攻撃を完成させること、Flowspec (RFC 8955) と網全体を止めた障害、
  スクラビングへの BGP 迂回と GRE の戻り経路の MTU、エニーキャストの吸収と撤退、フロー計測による
  検知とカーペットボミング、L7 (Rapid Reset・CONTINUATION・MadeYouReset)、DOTS、合図の悪用と
  試験の範囲。上の部品を防御の層として組み直す
- `tls-internals.md` — TLS 1.3 (RFC 9846 による改訂を含む)：削除による安全化 (RSA 鍵交換・CBC の廃止)、1-RTT
  ハンドシェイク、HKDF 鍵スケジュール、HelloRetryRequest と鍵共有の予測・互換モード・GREASE、
  SNI・ALPN・ECH (RFC 9849)・証明書圧縮、PKI の運用 (47 日への短縮・ACME/ARI・OCSP の退場と
  CRLite・static-ct-api・Entrust などの不信任・サーバ認証専用化)、チケット鍵と前方秘匿性、
  0-RTT のリプレイ対策、AEAD の利用上限と KeyUpdate、形式検証と実装の欠陥 (Heartbleed・
  CVE-2024-12797)、mTLS と DTLS 1.3、TLS を終端する中間装置の実害、耐量子ハイブリッドの
  展開と MTC、故障の形
- `dns-resolution.md` — DNS：委任とキャッシュ (TTL 付き結果整合性)、解決の実経路
  (スタブ〜権威)、リゾルバの内部 (QNAME 最小化 RFC 9156・SRTT によるサーバ選択・
  積極的な NSEC キャッシュ・0x20)、Flag Day と 1232 バイト・TCP 必須 (RFC 9210)・断片化の
  回避 (RFC 9715)、権威側 (エニーキャスト・NOTIFY/IXFR・隠しプライマリ・複数署名者
  RFC 8901)、SVCB/HTTPS と ECH・apex の CNAME・ワイルドカード・CAA、ECS の精度と
  プライバシ、毒入れと DNSSEC (2018 年・2026 年のルート KSK の交代、アルゴリズムの交代、
  NSEC3)、DoT/DoH/DoQ・DDR・ODoH と集中化、`.internal`、障害の事例 (Dyn・Akamai・
  Slack・.nz・AWS 2025)、SAD DNS・NXNS・TsuNAME・KeyTrap・ダングリングレコード
- `multicast-service-discovery.md` — IP マルチキャストとローカルなサービス探索：状態を網に置く
  代償と、インターネットで広がらなかった理由 (Diot ら 2000)、MAC アドレスへの 32 対 1 の写像、
  IGMPv3/MLDv2 (RFC 9776/9777) と報告の抑制の廃止、スヌーピング (RFC 4541・未登録グループ・
  クエリアの不在)、PIM-SM の RP と最短経路木への移行・RPF・SSM・PIM-DM・AMT、BIER と EVPN/MVPN、
  IPTV・市場データ・無線 LAN (RFC 9119)、mDNS/DNS-SD (probe・既知応答の抑制・`.local` の衝突)、
  SSDP・LLMNR・NBT-NS、Discovery Proxy と SRP (Thread・Matter)、名前解決の毒入れ・SSDP の反射・
  CallStranger
- `ip-routing-bgp.md` — IP ルーティングと BGP：最長一致とRIB/FIB、経路表の規模と 512k day、
  IGP (OSPF のエリアと LSA、IS-IS のレベル、SPF の遅延制御) vs EGP (パスベクトル)、BGP の
  決定プロセスの全段と MED・ルートリフレクタの振動、コミュニティ、Graceful Restart/LLGR、
  Route Flap Damping、Gao–Rexford 安定条件、相互接続の経済 (トランジット・ピアリング・IXP
  ルートサーバ・平坦化・オフネットキャッシュ・ネットワーク利用料)、計測 (RouteViews・RIS・
  IODA)、経路ハイジャックと経路リーク (ROV の普及、ASPA、RFC 9234、IRR の弱点、主要事件)、
  エニーキャスト、クラウド接続 (専用線接続と経路制御) の読み方
- `mpls-segment-routing.md` — MPLS と Segment Routing：ラベルスイッチングの価値 (表引きの
  速さではなく転送判断と IP 宛先の切り離し)、ラベルスタック・PHP・TTL の扱い、LDP と IGP の同期、
  RSVP-TE と状態の代償、FRR (RFC 4090・LFA・TI-LFA) と 50 ms の由来、BGP/MPLS L3VPN (RD と RT)・
  VPLS/EVPN・6PE/6VPE、SR-MPLS と SR ポリシー・PCE、SRv6 (SRH・ネットワークプログラミング・
  RFC 9800 の SID 圧縮)、閉じた信頼領域と SRv6 の境界フィルタ。通信事業者の基幹網の読み方
- `dc-fabric-routing.md` — データセンターファブリックの経路制御：BGP を IGP として使う設計
  (RFC 7938、AS 番号の割り当てと谷の遮断・経路探索、私用 AS と 4 オクテット、allowas-in と
  コンフェデレーション)、BGP unnumbered と RFC 8950、IS-IS/OSPF のフラッディングと動的フラッディング
  (RFC 9667)・RIFT、ECMP のハッシュと分極・resilient hashing・重み付け (WCMP・RFC 10005)、
  フローレット (CONGA・LetFlow)、BFD と局所修復・マイクロループ・ADD-PATH/PIC、退避
  (RFC 8326・Meta の 3 状態)、Google Firepath/Orion と Meta の BGP 設計、グレー障害と全体変更の
  影響範囲、セッション保護 (TCP-AO・GTSM) と CoPP。クラウドのアンダーレイの読み方
- `datacenter-congestion-control.md` — データセンターの輻輳制御とトランスポート：µs 級の RTT・浅いバッファ・
  incast と FCT の末尾、DCTCP (RFC 8257・Linux `tcp_dctcp.c` の α と PLB)、DCQCN と TIMELY、PFC の
  デッドロック・ストーム・ライブロックと IRN、Swift (ファブリックとホストの遅延の分割)、HPCC と HPCC++・CSIG、
  受信側主導 (pHost・NDP のトリミング・Homa と Linux への取り込みの状況・ExpressPass)、TCP の置き換え論争、
  散布・フローレット・PLB と順序の許容、SRD・Falcon・Azure の RDMA・Alibaba Solar、AI 学習の網と
  Meta の DCQCN の停止、UET (NSCC・RCCC・トリミング、1.0〜1.1)、信号の偽装・LoRDMA・NIC のサイドチャネル。
  信号と制御ループの置き場所で方式を読み分ける
- `virtual-networks-overlay.md` — 仮想ネットワークとオーバーレイ：マルチテナントの
  問題設定、VXLAN/Geneve カプセル化、マッピング分配の制御平面 (EVPN・中央ディレクトリ)、
  OVS と SmartNIC オフロード、セキュリティグループ = 分散ステートフル FW、NAT と
  インターネット境界。クラウド VPC の実装原理
- `nvme-over-fabrics.md` — NVMe over Fabrics とストレージネットワーク：SCSI・FC/FCP・iSCSI (RFC 7143)
  からの系譜と翻訳の除去、カプセルとキューペア (キュー 1 本 = コネクション 1 本)、Admin/I/O キュー・
  NQN・名前空間と予約、RDMA・FC-NVMe・NVMe/TCP (PDU・CRC32C ダイジェスト・R2T) の比較、
  Discovery と CDC (TP 8009/8010)・NBFT による起動、i10 と CPU 単価・DDP オフロード・incast・HoL、
  分離型ストレージ (ReFlex・AWS SRD/EBS・Alibaba Luna/Solar・Azure の RDMA)、Keep Alive・再接続・
  ANA・マルチパスのタイマと再接続の嵐・CQT、ホスト NQN の許可リスト・DH-HMAC-CHAP・TLS 1.3 と
  RDMA の暗号化の欠如。ブロックストレージを網の向こうへ伸ばす層
- `load-balancing-architecture.md` — ロードバランシング：DNS/ECMP/L4/L7 の積層、
  コンシステントハッシュと Maglev/Ananta 系譜、L7 終端と HTTP/2 の罠、ヘルスチェックと
  カスケード障害、クラウド LB (NLB/ALB 型) と弾力性の制御ループ
- `cdn-edge-delivery.md` — CDN・エッジ配信：光速という遅延の床、リクエストルーティング
  (DNS vs エニーキャスト)、キャッシュ階層とオリジンシールド、無効化 vs バージョン付き URL、
  エッジでのプロトコル終端、エッジ計算 (isolate 型)。CloudFront 型クラウド CDN の読み方
- `rpc-serialization.md` — RPC とシリアライズ：タイムアウトの「不明」と冪等キー・期限の伝播、
  形式の設計空間 (JSON/Protobuf/Avro/ゼロコピー)、Protobuf のワイヤ形式とフィールド番号の
  互換性規則、Avro のスキーマ解決とスキーマレジストリ、gRPC の HTTP/2 上の設計
  (トレーラの状態コードと gRPC-Web、長寿命接続と L7 LB)、デシリアライズ攻撃
- `service-mesh.md` — サービスメッシュ：ライブラリからサイドカー (Envoy/xDS) へ、
  mTLS 自動化とワークロードアイデンティティ (SPIFFE)、L7 トラフィック制御とリトライ予算、
  サイドカー税と eBPF 系メッシュ (Cilium)。L3/4 (セキュリティグループ) から L7 への
  ポリシーの移動
- `network-verification-telemetry.md` — ネットワーク検証とテレメトリ：意図・設定・制御プレーン・
  データプレーン・実際のパケットの食い違い、データプレーン検証 (HSA・Anteater・VeriFlow・NetPlumber・
  Delta-net・APKeep と増分化)、設定と制御プレーンの検証 (Batfish・Minesweeper とモデルの忠実度、
  AWS Tiros/Zelkova・Azure SecGuru/RCDC・Google の Zero Touch/CAPA)、SNMP の限界 (カウンタの桁あふれ・
  マイクロバースト) と gNMI/OpenConfig (dial-in/dial-out)、NetFlow v9/IPFIX・sFlow と標本化の精度、
  帯域内テレメトリ (INT・IOAM・DEX・ポストカード方式) と MTU の代償、能動測定 (Pingmesh・STAMP・
  Paris traceroute)、グレー障害と局所化 (007・NetBouncer)、変更前の検証と事後報告 (Facebook 2021・
  Google 2019・Azure 2023)、テレメトリの偽装・漏洩と gNMI の保護。網を予測し観測する運用の側

読む順の目安: 下の層から積み上げるなら、LAN (`ethernet-lan-switching.md`) → IP
(`ip-addressing-icmp-dhcp-nat.md`) → 経路制御 (`ip-routing-bgp.md`) → TCP
(`tcp-transport-congestion.md`) → 混雑したリンクでの配分 (`qos-traffic-management.md`)
→ 名前解決 (`dns-resolution.md`) → 暗号化 (`tls-internals.md`)
→ アプリケーションのプロトコル (`http-quic-evolution.md`、`email-delivery-authentication.md`、
対話的な音声・映像の `webrtc-realtime-media.md`)
→ トンネル (`vpn-tunneling.md`) の順に読む。その先で、中継・匿名化・検閲を経路の上の見え方として読み直すなら
`proxies-anonymity-censorship.md` へ進む。通信事業者の基幹網に進むなら、経路制御の後に
`mpls-segment-routing.md` を読む。その後は関心に応じて、ホストの中
(`kernel-network-stack.md`、その上でパケットを通すか捨てるかを決める
`packet-filtering-conntrack.md`)、DDoS の防御の配置 (`ddos-mitigation.md`)、データセンターとクラウド
(ファブリックの経路制御の `dc-fabric-routing.md`、その上の輻輳制御とトランスポートの
`datacenter-congestion-control.md`、`virtual-networks-overlay.md`、
`load-balancing-architecture.md`、`cdn-edge-delivery.md`、`service-mesh.md`、`rpc-serialization.md`、
ブロックストレージを網で運ぶ `nvme-over-fabrics.md`) へ進む。1 対多の配信とリンクの上のサービス探索は `multicast-service-discovery.md` へ。運用の側から網の振る舞いを予測し観測するなら `network-verification-telemetry.md` へ。
防御の側から読み直すなら [`../security/`](../security/README.md) の境界防御・検知・
攻撃のライフサイクルの文書へ。
