# Distributed Systems

分散システム (distributed systems) の分析。複製 (replication)・分断 (network
partition)・並行性が絡む領域で、`hardware/cpu/cache-coherence.md` が
扱う「コヒーレンス対コンシステンシ」の議論を、ノード間スケールへ拡張して捉える。

## 構成

- `consistency-models.md` — 一貫性モデル (linearizability / sequential / causal /
  eventual)、CAP・PACELC、論理時計、quorum、CRDT
- `consensus.md` — 分散合意 (consensus)：FLP 不可能性、Paxos/Raft/VSR、複製状態機械、
  クォーラム交差、リーダ交代と再構成、ReadIndex/リースリード、実システム (etcd/ZooKeeper/Spanner)
- `replication-partitioning.md` — 複製と分割 (replication & partitioning)：単一リーダ/マルチリーダ/
  リーダレス、Dynamo 型クォーラム (sloppy quorum, hinted handoff, read repair, Merkle tree anti-entropy)、
  複製遅延の異常、範囲/ハッシュ分割、consistent hashing と仮想ノード、リバランス、ホットキー、ルーティング
- `failure-models.md` — 障害モデル (failure models)：crash-stop/crash-recovery/omission/timing/Byzantine、
  同期・非同期・部分同期、FLP、故障検出器 (Chandra–Toueg, φ accrual)、gray failure、fail-slow、
  メタステーブル障害、BFT (LSP, PBFT) とデータセンターが crash 故障を仮定する理由

## 今後のテーマ候補

- 論理時計の深掘り (logical clocks) — Lamport, vector clock, HLC
