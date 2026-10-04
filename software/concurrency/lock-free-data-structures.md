# Lock-free データ構造とメモリ回収 (Lock-free Data Structures, ABA & Memory Reclamation)

## 概要 (Overview)

[`memory-models-synchronization.md`](memory-models-synchronization.md) は言語メモリモデルと同期プリミティブを整理し、「lock-free / wait-free 構造への入口」で止めた。本ドキュメントはその本編——**ロックを使わず、アトミック操作 (CAS) だけで共有データ構造を正しく作る**技術を扱う。

lock-free の動機は「速いから」では**ない**。無競合ならロックと大差なく、高競合ではむしろ遅いことすらある。本当の価値は**進行保証 (progress guarantee)**——どのスレッドが OS に止められても・優先度逆転が起きても・（プロセス共有メモリなら）参加者が死んでも、**系全体としては誰かが必ず前進する**こと——にある。

そして lock-free 設計の最難関はアルゴリズム本体ではなく、**メモリ回収 (memory reclamation)**——「ノードを外した後、いつ `free` してよいか。まだ読んでいる誰かがいるかもしれないのに」——である。本ドキュメントは、(1) 古典アルゴリズム (Treiber stack / Michael–Scott queue) の実物、(2) ABA 問題、(3) 回収3方式 (hazard pointer / epoch / RCU)、(4) 性能の現実と実務指針を整理する。分析深度は Level 3。

## 方法論 (Methodology)

- 正しさの基準を**線形化可能性**（各操作がある一点で原子的に起きたように見える。[`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) の単一オブジェクト版）に置き、各アルゴリズムの「線形化点 (linearization point)」を特定しながら読む。
- コードは C++ 風の擬似コードで示し、`memory_order` を明示する（[`memory-models-synchronization.md`](memory-models-synchronization.md) の意味論の適用例として）。
- ABA と回収を「同じ根 (ポインタの再利用) から生える2つの症状」として統一的に扱う。
- 一次資料（Treiber、Michael & Scott、Michael、McKenney、Herlihy）に基づく。

## 発見事項 (Findings)

1. **lock-free の価値は速度ではなく進行保証と障害耐性である。** ロック保持者がプリエンプトされると全員が待つ（コンボイ）、優先度逆転が起きる、保持したまま死ぬと永久に壊れる——lock-free はこれらを構造的に排除する。スループットはワークロード依存で、**「lock-free だから速い」は誤解**である。
2. **CAS ループの骨格は「読む→計算する→交換を試みる→負けたらやり直す」。** 交換の成功点が線形化点になる。負けたスレッドの再試行は他者の成功を意味するので、系全体は前進している（= lock-free。ただし個々のスレッドは飢えうる＝wait-free ではない）。
3. **ABA は「値の一致は状態の不変を意味しない」ことの帰結。** CAS はビット列しか比較しない。ポインタ A が free され、同じ番地に別ノードが割り当てられて再び A になると、CAS は「変化なし」と誤認して**構造を破壊する**。GC のある言語では（その参照が生きている限り再利用が起きないので）この形の ABA は自然に消える——**GC は ABA 対策を言語ランタイムに外注する行為**とも言える。
4. **最難関は削除、正確には「解放のタイミング」。** リストからノードを外しても、直前にポインタを読んだ読者がまだ中を見ているかもしれない。即 `free` すれば use-after-free。解決はすべて「**読者がいなくなったと証明できるまで解放を遅延する**」の変種であり、証明の取り方でコストが変わる。
5. **回収3方式は「読者のコストをどこまで下げるか」の階段である。** hazard pointer は読者がポインタ単位で在圏を宣言（読み側にストア＋バリア、回収は正確）、epoch 系は世代単位で一括（読み側は世代印だけ、停滞読者がいると回収が無限に遅れる）、RCU は読者ほぼゼロコスト（カーネルはプリエンプション禁止区間で代用）で**読み圧倒的多数**の構造に最適。
6. **実務の第一選択は「自分で書かない」。** 正しい lock-free コードはメモリモデル・ABA・回収・プラットフォーム差の全てを同時に満たす必要があり、専門家の実装 (Linux カーネルの RCU、`liburcu`、Rust の `crossbeam`、C++ の `folly`) か、そもそも**ロック＋分割 (sharding)** で足りないかをまず問うべきである。

## アーキテクチャ (Architecture)

### 進行保証の再掲と本書の位置

```
 blocking (mutex)  ─▶ obstruction-free ─▶ lock-free ─▶ wait-free
  誰かが止まると        単独実行なら         誰かは必ず      全員が有界
  全員止まりうる        必ず進む             進む           ステップで進む
                                      ◀── 本書の主対象 ──▶
```

### Treiber stack — 最小の lock-free 構造

```
   head ──▶ [C] ──▶ [B] ──▶ [A] ──▶ null

 push(X):  X.next = head を読んだ値;  CAS(head, 読んだ値, X)   失敗なら再読・再試行
 pop():    old = head;  CAS(head, old, old->next)             成功した CAS が線形化点
```

### Michael–Scott queue — 2ポインタと「助け合い」

```
   head ──▶ [dummy] ──▶ [A] ──▶ [B] ◀── tail
   dequeue は head 側、enqueue は tail 側で CAS。
   enqueue は2手 (①末尾ノードの next を CAS → ②tail を前進) に分かれるため、
   ①と②の間に割り込んだ他スレッドは「遅れた tail を自分で前進させて」から進む
   —— helping: 他者の未完了操作を完成させる、lock-free 設計の定石。
```

tail が一時的に遅れても不変条件（tail から辿れば必ず末尾に着く）は保たれる。**「完全に最新」ではなく「回復可能な程度に正しい」状態を許し、観測者が修復する**——この緩めた不変条件＋helping が、2ワード以上を単一 CAS で更新できない制約への標準的な答えである。

## 技術詳細 (Technical Details)

### 1. Treiber stack の実装と memory_order

```cpp
// 例示コード (illustrative)。回収問題を除いた骨格。
struct Node { T value; Node* next; };
std::atomic<Node*> head{nullptr};

void push(T v) {
    Node* n = new Node{v, nullptr};
    n->next = head.load(std::memory_order_relaxed);
    while (!head.compare_exchange_weak(
               n->next, n,
               std::memory_order_release,   // 成功: n の中身の公開 (release)
               std::memory_order_relaxed))  // 失敗: n->next が最新値に更新されて再試行
        ;
}

std::optional<T> pop() {
    Node* old = head.load(std::memory_order_acquire);
    while (old && !head.compare_exchange_weak(
               old, old->next, std::memory_order_acquire))
        ;
    if (!old) return std::nullopt;
    T v = old->value;
    // delete old; ← ここが問題の全て（§3 メモリ回収）
    return v;
}
```

- `compare_exchange_weak` は失敗時に期待値を**現在値へ書き戻す**ので、ループが自然に再読になる。
- push 成功の release / pop の acquire ペアが「ノードの中身は繋いだ後に見える」を保証する（[`memory-models-synchronization.md`](memory-models-synchronization.md) の release/acquire の教科書的適用）。

### 2. ABA 問題 — 具体的な破壊シナリオ

```
 スタック: head → A → B → C
 T1: pop 開始。old=A, old->next=B を読んだ直後にプリエンプト
 T2: pop A、pop B、A を free、新ノードを push —— アロケータが同じ番地を再利用し、
     スタックは head → A' → C  (A' は番地が A と同じ別ノード)
 T1: 再開。CAS(head, A, B) —— 番地は一致するので成功
 結果: head → B —— とっくに外れて free 済みの B を指す。構造破壊・use-after-free
```

対策は3系統ある。

- **タグ付きポインタ (versioned pointer)**: ポインタに更新カウンタを併置し、ダブル幅 CAS（x86 の `cmpxchg16b`）で「番地＋世代」を比較する。世代が回り切る理論的隙は残るが実用上十分なことが多い。
- **LL/SC**: ARM 等の load-linked/store-conditional は「番地への介入があったか」自体を検出するので原理的に ABA がない（[`../../hardware/cpu/memory-ordering-atomics.md`](../../hardware/cpu/memory-ordering-atomics.md)）。ただし偽失敗があり、C++ からは抽象の下に隠れる。
- **回収の遅延（次節）**: 「読者がいる間は再利用させない」なら、そもそも同一番地の再登場が起きない。**ABA 対策と回収問題は同じ解で同時に解ける**——これが実務で回収機構が中心課題になる理由。

### 3. メモリ回収 — 3方式の設計空間

| 方式 | 読者のコスト | 回収の正確さ | 弱点 | 代表実装 |
|---|---|---|---|---|
| **Hazard Pointer** | 参照ごとにポインタ公表＋バリア | 正確（公表中のノードだけ保留） | 読み側が重い。取得は検証ループ | `folly::hazptr`, C++26 提案 |
| **Epoch (EBR)** | 区間の出入りで世代印 | 世代単位で一括 | **停滞読者が全回収を止める** | Rust `crossbeam-epoch` |
| **RCU** | ほぼゼロ（カーネル: プリエンプション禁止区間） | grace period 単位 | 書き手が高コスト。使い方に規律 | Linux カーネル, `liburcu` |

- **Hazard pointer (Michael 2004)**: 読者は「いま見ているポインタ」をスレッド固有スロットに公表する。回収側は削除済みノードを溜め、**どのスロットにも載っていない**ものだけ free する。公表→再検証（読んだ後にまだ繋がっているか確認）の順序が要で、ここにバリアが要る。
- **Epoch-based reclamation**: 大域世代カウンタと「各スレッドがどの世代で読み区間に入ったか」を記録し、全読者が世代 e を離れたら e 以前の削除物を一括解放。読者は速いが、**1本の長い読み区間**（またはブロックした読者）が全体の回収を止め、メモリが際限なく溜まりうる。
- **RCU (Read-Copy-Update)**: 「読者を数えない。**読者がいなくなったはずの時点** (grace period) を待つ」。カーネル実装では読者区間 = プリエンプション禁止区間とし、**全 CPU が一度スケジュール点を通過**したら読者は残っていない、とみなす。読者は文字通りゼロコスト（`rcu_read_lock` は多くの構成でコンパイラバリアのみ）。書き手は「コピーして差し替え、`synchronize_rcu()` を待ってから旧版を解放」する。Linux カーネルのルーティング表・dentry など**読み 99.9% の構造**を支える基盤機構であり、`liburcu` がユーザ空間版を提供する。

3方式は「**読者に払わせるか、メモリ滞留で払うか、書き手に払わせるか**」の選択である。読み書き比・レイテンシ要求・メモリ上限が決め手になる。

### 4. wait-free への一歩

- lock-free では個々のスレッドは飢えうる。**wait-free**（全スレッドが有界ステップで完了）には、遅いスレッドの操作を他者が代行する **helping** の一般化が要る。Herlihy (1991) は、合意数 (consensus number) ∞ のプリミティブ（CAS/LL-SC）があれば**任意のデータ構造の wait-free 実装が存在する**ことを示した（universal construction）——ただし汎用構成は遅く、実用の wait-free は fetch_add ベースのキューなど特定構造に限られる。
- 「fetch_add は必ず成功する」——CAS ループ（失敗しうる）を fetch_add（無条件に前進する）へ置き換えられる構造は、それだけで実質 wait-free 化と高スループット化を同時に得る。実務の高速 MPMC キューがインデックスを fetch_add で刻む理由。

### 5. 性能の現実 — 競合はコヒーレンスで決まる

- 全スレッドが同じ `head` を CAS する構造は、その**キャッシュラインの所有権が Modified 状態でコア間を往復**する（[`../../hardware/cpu/cache-coherence.md`](../../hardware/cpu/cache-coherence.md)）。CAS 自体は数十サイクルでも、ラインの奪い合いで実効コストは桁で膨らみ、**コア数を増やすほど遅くなる**ことすらある。
- 緩和策: **backoff**（負けたら少し待つ）、**elimination**（push と pop を出会わせて相殺——スタックでは中心へ触れずに済む）、**combining**（1スレッドが皆の操作をまとめて実行）、そして最も効くのは**単一ポイントを作らない設計**（per-core シャーディング、SPSC キューの束、フラットコンバイニング）。
- 測定なしに lock-free を選ばない。「mutex で十分か」→「sharded mutex で十分か」→「既製 lock-free ライブラリ」→「自作」の順で、**自作は最後の手段**である。

## セキュリティ考察 (Security Considerations)

- **回収バグは即 use-after-free 脆弱性**: lock-free の解放誤りは、ヒープ再利用を介した型混同・任意書き込みに直結する、攻撃者にとって上質な原料である。しかも競合窓が狭く**テストをすり抜けて本番で顕在化**する。カーネルの RCU 誤用 (grace period を待たない解放) は実際に権限昇格級の CVE を生んできたクラスであり、[`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) の Dirty COW と並ぶ「カーネル内競合＝昇格」の系譜にある。
- **ABA は整合性の静かな破壊**: クラッシュせず構造だけ壊れる場合、破損データがそのまま下流（認可判定・課金）に流れる。fail-stop でない失敗は検出も遅れる。
- **検証はモデル検査で**: この領域の正しさは通常テストでは確立できない。C11/C++11 モデル上の網羅検査 (CDSChecker, GenMC)、Rust の並行 permutation テスト (loom)、カーネルの KCSAN のような**データ競合検出器**を CI に組み込むのが現実的な最低線である。「動いた」ではなく「モデル上で反例がない」を基準にする（[`memory-models-synchronization.md`](memory-models-synchronization.md) の結論と同じ規律）。
- **DoS としての回収停滞**: epoch 系で読者を意図的に停滞させられる構成（読者区間がユーザ入力で伸びる等）は、メモリ枯渇 DoS の攻撃面になる。読者区間の長さを入力から独立させる設計が要る。

## 結論 (Conclusions)

- lock-free の本質は進行保証である。CAS ループ＋線形化点＋helping という骨格で「誰が止まっても系は進む」を作り、その正しさは線形化可能性で判定する。
- ABA と回収は同じ根（番地の再利用）を持つ双子の問題で、hazard pointer / epoch / RCU は「読者・メモリ・書き手のどこにコストを置くか」の3解である。読み圧倒的多数なら RCU 系が強い。
- 性能はアルゴリズムの段数ではなくキャッシュコヒーレンス上の競合で決まる。単一ホットポイントを作らない設計が、どの同期方式よりも効く。
- 実務の順序は「ロック＋分割 → 既製ライブラリ → 自作」。自作するなら、メモリモデルの明示・モデル検査・回収の証明までがワンセットである。

### 次の接続先 (Related Topics)

- [`memory-models-synchronization.md`](memory-models-synchronization.md) — 本書の前提。happens-before・memory_order・進行保証の定義。
- [`../../hardware/cpu/memory-ordering-atomics.md`](../../hardware/cpu/memory-ordering-atomics.md) — CAS/LL-SC の命令実体と ABA 耐性の差。
- [`../../hardware/cpu/cache-coherence.md`](../../hardware/cpu/cache-coherence.md) — CAS 競合の実コストを決めるライン所有権の物理。
- [`../distributed-systems/consistency-models.md`](../distributed-systems/consistency-models.md) — 正しさ基準（線形化可能性）の分散版。
- [`../operating-systems/virtual-memory-paging.md`](../operating-systems/virtual-memory-paging.md) — カーネル内競合が権限昇格になる構造 (Dirty COW) との並び。

## 参考文献 (References)

**一次資料 (Primary — 原論文・公式ドキュメント)**

- Treiber, R. K. (1986). *Systems Programming: Coping with Parallelism* (IBM Research Report RJ 5118). — Treiber stack の原典。
- Michael, M. M., & Scott, M. L. (1996). Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms. *PODC '96*. DOI: 10.1145/248052.248106 — M&S queue の原典。
- Michael, M. M. (2004). Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects. *IEEE TPDS*, 15(6), 491–504. DOI: 10.1109/TPDS.2004.8
- Herlihy, M. (1991). Wait-Free Synchronization. *ACM TOPLAS*, 13(1), 124–149. DOI: 10.1145/114005.102808 — 合意数と universal construction。
- Herlihy, M., & Wing, J. (1990). Linearizability: A Correctness Condition for Concurrent Objects. *ACM TOPLAS*, 12(3), 463–492. DOI: 10.1145/78969.78972 — 正しさ基準の原典。
- Linux Kernel Documentation, *RCU concepts*. — [kernel.org RCU](https://www.kernel.org/doc/html/latest/RCU/index.html) — Accessed: 2026-07-02。

**二次資料 (Secondary — 解説・教科書・実装)**

- McKenney, P. E. (2007). What is RCU, Fundamentally? *LWN.net*. — [lwn.net/Articles/262464](https://lwn.net/Articles/262464/) — Accessed: 2026-07-02。RCU 設計者による3部作の解説。
- Herlihy, M., & Shavit, N. (2012). *The Art of Multiprocessor Programming* (Revised ed.). Morgan Kaufmann. — lock-free 構造・elimination・combining の標準教科書。
- crossbeam (Rust) — epoch-based reclamation の実装。 — [docs.rs/crossbeam](https://docs.rs/crossbeam/latest/crossbeam/) — Accessed: 2026-07-02。
