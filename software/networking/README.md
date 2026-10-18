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
- `tcp-transport-congestion.md` — TCP と輻輳制御：信頼性の道具立て (ACK/SACK/RTO)、
  フロー制御 vs 輻輳制御、AIMD の公平性 (Chiu–Jain)、CUBIC、バッファブロートと AQM/ECN、
  BBR、TCP の構造的限界 (HoL・硬直化)
- `http-quic-evolution.md` — HTTP/1.1 → 2 → QUIC/HTTP3：多重化と HoL の局在化、
  HPACK/QPACK、QUIC の設計 (ストリーム、コネクション ID、トランスポート暗号化、0-RTT と
  リプレイ、アンプ制限)、硬直化への構造的応答
- `kernel-network-stack.md` — カーネルネットワークスタック：NIC リングと NAPI (割り込みの
  経済)、sk_buff、GRO/GSO とパケット単価、RSS/RPS/RFS、qdisc/BQL、XDP・AF_XDP・DPDK の
  バイパス系譜。上 2 本のプロトコルを運ぶ機械側
- `tls-internals.md` — TLS 1.3：削除による安全化 (RSA 鍵交換・CBC の廃止)、1-RTT
  ハンドシェイク、HKDF 鍵スケジュール、PKI と Certificate Transparency、0-RTT と
  ダウングレード防御、耐量子ハイブリッドへの移行
- `dns-resolution.md` — DNS：委任とキャッシュ (TTL 付き結果整合性)、解決の実経路
  (スタブ〜権威)、毒入れと DNSSEC、DoT/DoH と集中化のトレードオフ、障害の構造
  (依存の環・ネガティブキャッシュ)
- `ip-routing-bgp.md` — IP ルーティングと BGP：最長一致とRIB/FIB、IGP (リンクステート)
  vs EGP (パスベクトル)、BGP のポリシー機構と Gao–Rexford 安定条件、経路ハイジャックと
  RPKI、エニーキャスト、クラウド接続 (専用線接続と経路制御) の読み方
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

## 今後のテーマ候補

- 帯域内テレメトリとネットワーク検証 (network verification) — INT、経路の形式検証。
  運用の観測・保証側の深掘り
