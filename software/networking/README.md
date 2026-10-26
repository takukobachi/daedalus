# Networking

ネットワーキングとプロトコル (networking & protocols) の分析。「誰も全体を管理していない系」
での分散制御 (輻輳制御) と、層をまたぐ設計判断 (HoL ブロッキング・硬直化・暗号化の範囲) を軸に、
リンク層 (LAN) からトランスポート、アプリケーションプロトコルまでを読み解く。
[`../distributed-systems/`](../distributed-systems/) の故障モデル (ロス・遅延・分断) の発生源で
あり、[`../../hardware/hw-sw-interface/`](../../hardware/hw-sw-interface/)（NIC・割り込み・DMA）の真上の層。

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
- `tcp-transport-congestion.md` — TCP と輻輳制御：信頼性の道具立て (ACK/SACK/RTO)、
  フロー制御 vs 輻輳制御、AIMD の公平性 (Chiu–Jain)、CUBIC、バッファブロートと AQM/ECN、
  BBR、TCP の構造的限界 (HoL・硬直化)
- `qos-traffic-management.md` — QoS とトラフィック制御：QoS は容量を作らず混雑時の配分を
  決めるだけ、IntServ/RSVP から DiffServ へ、DSCP と PHB (EF・AF・LE・NQB)、信頼境界と
  インターネットでの DSCP の書き換え、スケジューリング (厳密優先・WFQ・DRR)、ポリシングと
  シェーピング (3 色マーカ・TCP との相性)、階層型 QoS と Linux tc (HTB・fq_codel・CAKE)、
  L4S・5QI・PFC、優先度の窃取とネットワーク中立性
- `http-quic-evolution.md` — HTTP/1.1 → 2 → QUIC/HTTP3：多重化と HoL の局在化、
  HPACK/QPACK、QUIC の設計 (ストリーム、コネクション ID、トランスポート暗号化、0-RTT と
  リプレイ、アンプ制限)、硬直化への構造的応答
- `webrtc-realtime-media.md` — WebRTC とリアルタイムメディア伝送：遅延予算 (G.114) と「遅れたデータは
  失われたデータ」、ICE・STUN・TURN と trickle ICE・mDNS 候補、DTLS-SRTP と指紋、SCTP のデータチャネル、
  JSEP と Unified Plan、RTP/RTCP とジッタバッファ・NACK/RTX・FEC (FlexFEC・RED)・PLI/FIR、
  GCC・transport-wide CC・SCReAM・NADA と RMCAT の要件、メッシュ・MCU・SFU とサイマルキャスト・SVC、
  SFrame と MLS、WebTransport・MoQ・WHIP/WHEP、TURN の悪用・IP アドレスの漏洩・cryptex
- `kernel-network-stack.md` — カーネルネットワークスタック：NIC リングと NAPI (割り込みの
  経済)、sk_buff、GRO/GSO とパケット単価、RSS/RPS/RFS、qdisc/BQL、XDP・AF_XDP・DPDK の
  バイパス系譜。上 2 本のプロトコルを運ぶ機械側
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
- `tls-internals.md` — TLS 1.3：削除による安全化 (RSA 鍵交換・CBC の廃止)、1-RTT
  ハンドシェイク、HKDF 鍵スケジュール、PKI と Certificate Transparency、0-RTT と
  ダウングレード防御、耐量子ハイブリッドへの移行
- `dns-resolution.md` — DNS：委任とキャッシュ (TTL 付き結果整合性)、解決の実経路
  (スタブ〜権威)、毒入れと DNSSEC、DoT/DoH と集中化のトレードオフ、障害の構造
  (依存の環・ネガティブキャッシュ)
- `multicast-service-discovery.md` — IP マルチキャストとローカルなサービス探索：状態を網に置く
  代償と、インターネットで広がらなかった理由 (Diot ら 2000)、MAC アドレスへの 32 対 1 の写像、
  IGMPv3/MLDv2 (RFC 9776/9777) と報告の抑制の廃止、スヌーピング (RFC 4541・未登録グループ・
  クエリアの不在)、PIM-SM の RP と最短経路木への移行・RPF・SSM・PIM-DM・AMT、BIER と EVPN/MVPN、
  IPTV・市場データ・無線 LAN (RFC 9119)、mDNS/DNS-SD (probe・既知応答の抑制・`.local` の衝突)、
  SSDP・LLMNR・NBT-NS、Discovery Proxy と SRP (Thread・Matter)、名前解決の毒入れ・SSDP の反射・
  CallStranger
- `ip-routing-bgp.md` — IP ルーティングと BGP：最長一致とRIB/FIB、IGP (リンクステート)
  vs EGP (パスベクトル)、BGP のポリシー機構と Gao–Rexford 安定条件、経路ハイジャックと
  RPKI、エニーキャスト、クラウド接続 (専用線接続と経路制御) の読み方
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
- `virtual-networks-overlay.md` — 仮想ネットワークとオーバーレイ：マルチテナントの
  問題設定、VXLAN/Geneve カプセル化、マッピング分配の制御平面 (EVPN・中央ディレクトリ)、
  OVS と SmartNIC オフロード、セキュリティグループ = 分散ステートフル FW、NAT と
  インターネット境界。クラウド VPC の実装原理
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
→ トンネル (`vpn-tunneling.md`) の順に読む。通信事業者の基幹網に進むなら、経路制御の後に
`mpls-segment-routing.md` を読む。その後は関心に応じて、ホストの中
(`kernel-network-stack.md`、その上でパケットを通すか捨てるかを決める
`packet-filtering-conntrack.md`)、DDoS の防御の配置 (`ddos-mitigation.md`)、データセンターとクラウド
(ファブリックの経路制御の `dc-fabric-routing.md`、`virtual-networks-overlay.md`、
`load-balancing-architecture.md`、`cdn-edge-delivery.md`、`service-mesh.md`、`rpc-serialization.md`)
へ進む。1 対多の配信とリンクの上のサービス探索は `multicast-service-discovery.md` へ。運用の側から網の振る舞いを予測し観測するなら `network-verification-telemetry.md` へ。
防御の側から読み直すなら [`../security/`](../security/README.md) の境界防御・検知・
攻撃のライフサイクルの文書へ。
