# Distributed Systems

分散システム (distributed systems) の分析。複製 (replication)・分断 (network
partition)・並行性が絡む領域で、`hardware/cpu-architecture/cache-coherence.md` が
扱う「コヒーレンス対コンシステンシ」の議論を、ノード間スケールへ拡張して捉える。

## 構成

- `consistency-models.md` — 一貫性モデル (linearizability / sequential / causal /
  eventual)、CAP・PACELC、論理時計、quorum、CRDT

## 今後のテーマ候補

- 合意 (consensus) — Paxos, Raft, view-stamped replication
- 複製と分割 (replication & partitioning) — leader/follower, sharding
- 論理時計の深掘り (logical clocks) — Lamport, vector clock, HLC
- 障害モデル (failure models) — crash-stop, omission, Byzantine
