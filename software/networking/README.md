# Networking

ネットワーキングとプロトコル (networking & protocols) の分析。「誰も全体を管理していない系」
での分散制御 (輻輳制御) と、層をまたぐ設計判断 (HoL ブロッキング・硬直化・暗号化の範囲) を軸に、
トランスポートからアプリケーションプロトコルまでを読み解く。
[`../distributed-systems/`](../distributed-systems/) の故障モデル (ロス・遅延・分断) の発生源で
あり、[`../../hardware/hw-sw-interface/`](../../hardware/hw-sw-interface/)（NIC・割り込み・DMA）の真上の層。

## 構成

- `tcp-transport-congestion.md` — TCP と輻輳制御：信頼性の道具立て (ACK/SACK/RTO)、
  フロー制御 vs 輻輳制御、AIMD の公平性 (Chiu–Jain)、CUBIC、バッファブロートと AQM/ECN、
  BBR、TCP の構造的限界 (HoL・硬直化)
- `http-quic-evolution.md` — HTTP/1.1 → 2 → QUIC/HTTP3：多重化と HoL の局在化、
  HPACK/QPACK、QUIC の設計 (ストリーム、コネクション ID、トランスポート暗号化、0-RTT と
  リプレイ、アンプ制限)、硬直化への構造的応答

## 今後のテーマ候補

- TLS 1.3 深掘り (TLS internals) — ハンドシェイク、鍵スケジュール、PKI と証明書検証
- IP ルーティングと BGP (routing) — 経路制御、BGP の信頼モデル、RPKI
- DNS — 再帰解決、キャッシュ、DNSSEC/DoH/DoT
- カーネルネットワークスタック (kernel networking) — sk_buff, NAPI, XDP/eBPF
