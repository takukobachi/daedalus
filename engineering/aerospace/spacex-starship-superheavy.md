# Starship / Super Heavy — 完全再使用アーキテクチャ (Starship / Super Heavy: Full Reusability)

## 概要 (Overview)

**Starship**（上段）と **Super Heavy**（下段ブースタ）から成る2段式ロケットは、SpaceX が「**両段・完全再使用 (fully reusable)**」を目標に開発する、史上最大・最強の打ち上げ機である。全高はおよそ **121〜124 m**、離陸推力は **約 74〜80 MN（1,800万 lbf 超）** に達し、Saturn V のほぼ2倍——人類が飛ばした最も強力なロケットである[^scientificamerican][^starship-official]。Super Heavy は **33基の Raptor**、Starship 上段は **6基の Raptor**（海面3＋真空3）を搭載し、推進剤は液体メタン／液体酸素（methalox）。機体は**ステンレス鋼**製で、LEO へ **100 t 超（再使用構成）** のペイロードを目標とする[^starship-official][^satnews]。

本ドキュメントは、Starship を「**Falcon 9 の部分再使用を、両段・全機へ徹底した到達点**」として捉える（前章 `spacex-reusability.md` の延長）。完全再使用がもたらす設計判断——ステンレス鋼、33基クラスタ、**射点アームでのブースタ空中キャッチ (Mechazilla)**、ホットステージング、耐熱タイル——と、それを「飛ばして壊して直す」反復で詰めてきた飛行試験の歴史を、Level 3 で分析する。

## 方法論 (Methodology)

- Starship を要求（完全再使用・低コスト・大ペイロード・火星）から逆算し、各設計判断が**どの要求に応える妥協か**を読み解く。
- Falcon 9 との差分（部分→完全再使用、過熱に耐える材料、キャッチ着陸）を対比軸に据える。
- 飛行試験 (IFT-1 〜 Flight 12) を**反復開発 (iterative, hardware-rich development)** のサイクルとして時系列で追い、各回の「狙いと学び」を整理する。
- 数値・経緯は SpaceX 公式（車両ページ・飛行試験ページ）と一次に近い報道・FAA 文書に基づき、典拠を脚注に付す。最新の飛行試験は流動的なため、確定値と暫定値を区別する。

## 発見事項 (Findings)

1. **完全再使用が全設計を駆動する。** 「両段とも無傷で戻し、最小整備で再飛行する」という要求が、材料・形状・着陸方式・エンジン数のすべてを規定する。Falcon 9 が残した「使い捨ての第2段」を無くすのが核心。
2. **ステンレス鋼という逆張り。** 航空宇宙の常識（軽量なアルミ／複合材）に反し、SpaceX は**ステンレス鋼**を選んだ——低温で強度が増し、高温（再突入）にも耐え、安価で溶接でき、量産に向くため。比強度より「広い温度域での総合性能とコスト」を取った判断。
3. **33基クラスタは「数による推力と冗長」。** Super Heavy は Raptor を33基束ねて巨大推力を得つつ、複数基喪失への耐性を確保する。製造可能な中型エンジンを大量に並べる思想（Falcon 9 の9基の極大化）。
4. **着陸は「脚」から「キャッチ」へ。** ブースタに着陸脚を付けず、**射点タワーのアーム (chopsticks) で空中キャッチ (Mechazilla)** する。脚の重量を排し、整備性と再飛行速度を上げる過激な最適化。実飛行で複数回成功している[^sfn-catch]。
5. **ホットステージング。** 上段エンジンを下段がまだ燃焼中に点火して分離する方式を採用し、段間の性能損失を抑える。
6. **再突入は耐熱タイルで凌ぐ。** 上段 Starship は軌道速度からの再突入時の空力加熱を、機体下面の**耐熱タイル (TPS)** で防護する。ここが上段再使用の最大の技術的難所。
7. **開発は「反復」そのもの。** IFT-1（2023年）の爆発から Flight 12（2026年）の V3 初飛行まで、SpaceX は失敗を織り込んだ高頻度試験で設計を詰めてきた。直近の Flight 12 ではブースタが不具合を起こし FAA 調査対象となるなど、開発は依然として途上にある[^sfn-faa][^mynews-faa]。

## アーキテクチャ (Architecture)

### スタック構成 (Stack Configuration)

```
        ┌─────────────┐  ← Starship（上段 / 宇宙船）
        │  ペイロード   │     Raptor ×6（海面3 + 真空3）
        │  ベイ        │     耐熱タイル(TPS)で再突入を防護
        │             │     ペイロード >100 t → LEO（再使用構成）
        ├─────────────┤  ← ホットステージング段間部
        │             │
        │             │  ← Super Heavy（下段 / ブースタ）
        │  推進剤      │     Raptor ×33（methalox）
        │  タンク      │     グリッドフィン、Mechazilla で空中キャッチ
        │             │
        └──┬┬┬┬┬┬┬┬┬──┘
           33基の Raptor → 離陸推力 ~74–80 MN（Saturn V の約2倍）
   全高 ~121–124 m / 直径 9 m / ステンレス鋼製 / 両段完全再使用
```

| 諸元 | 値 | 典拠 |
|---|---|---|
| 全高（スタック） | ~121–124 m | [^scientificamerican][^satnews] |
| 直径 | 9 m | [^starship-official] |
| 離陸推力 | ~74–80 MN（>18M lbf、Saturn V の約2倍） | [^scientificamerican] |
| Super Heavy エンジン | Raptor ×33 | [^starship-official] |
| Starship エンジン | Raptor ×6（海面3＋真空3） | [^starship-official] |
| 推進剤 | 液体メタン / 液体酸素（methalox） | [^starship-official] |
| 構造材 | ステンレス鋼（30X 系） | [^starship-official] |
| ペイロード（LEO, 再使用） | 100 t 超（V3 目標、使い捨て構成で更に増） | [^satnews] |
| 再使用 | 両段・完全再使用を目標 | [^starship-official] |

### Falcon 9 からの飛躍 (Leap from Falcon 9)

| 観点 | Falcon 9 | Starship / Super Heavy |
|---|---|---|
| 再使用 | **部分**（第1段＋フェアリング、第2段は使い捨て） | **完全**（両段とも回収・再使用を目標） |
| 構造材 | アルミリチウム合金 | **ステンレス鋼** |
| 第1段エンジン | Merlin ×9（ケロシン, ガスジェネレータ） | Raptor ×33（メタン, FFSC） |
| 着陸（下段） | 着陸脚で陸／ドローン船 | **タワーのアームで空中キャッチ (Mechazilla)** |
| 上段の帰還 | （使い捨て） | **耐熱タイルで再突入し帰還** |
| 段分離 | 通常分離 | **ホットステージング** |

### Mechazilla — 空中キャッチ (Tower Catch)

```
        射点タワー (launch/catch tower)
        ┃   ╱━━ chopsticks（2本のアーム）━━╲
        ┃  ║                              ║
        ┃  ║   ← 降下する Super Heavy を    ║
        ┃  ║      アームで挟んで受け止める   ║
        ┃  ║   （着陸脚を持たない＝軽量化）  ║
        ┃  ╲______________________________╱
        ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

- Super Heavy は**着陸脚を持たない**。代わりに、降下してくるブースタを**射点タワーの2本のアーム（通称 chopsticks）で空中で挟んで捕捉**する。
- 利点: (1) 着陸脚の重量を排して性能向上、(2) 機体を直接タワーへ戻すことで**整備・再スタックを高速化**（飛行機的な運用に近づく）。
- 難所: ブースタを **数十 cm の精度でタワー位置へ誘導**する制御が要る。Falcon 9 のホバスラング（`spacex-reusability.md`）を、固定構造物への精密ランデブーへ先鋭化したもの。実飛行で複数回キャッチに成功している[^sfn-catch]。

## 技術詳細 (Technical Details)

### なぜステンレス鋼か (Why Stainless Steel)

航空宇宙では軽量なアルミ合金・複合材が常識だが、Starship は**ステンレス鋼 (30X 系)** を選んだ。その根拠を示す。

| 要求 | ステンレス鋼の利点 |
|---|---|
| 極低温強度 | 低温で**強度・靭性が増す**（液体メタン/酸素タンクに好適） |
| 再突入の耐熱 | **融点が高く高温に強い**（アルミは ~150°C で軟化、鋼は遥かに上） |
| コスト | アルミ/複合材より**桁違いに安い**——完全再使用＋量産の経済に効く |
| 製造性 | **溶接で大型構造を作れる**。複合材のオートクレーブ不要、屋外で量産可能 |
| 代償 | 比強度はアルミに劣り**重い**——が、上記の総合利得で相殺と判断 |

これは「比強度で材料を選ぶ」航空機の発想（`aircraft-engineering-fundamentals.md` の材料節）への意図的な逆張りで、**広い温度域での性能 × コスト × 製造性**を総合最適化した結果である。

### ホットステージング (Hot-Staging)

段分離時、**上段 Starship のエンジンを、下段 Super Heavy がまだ推力を出している間に点火**してから分離する方式。通常分離で生じる「分離後にエンジンが立ち上がるまでの推力空白（重力損失）」を避け、性能を稼ぐ。段間部に高温排気を逃がす構造（vented interstage）を設ける必要があり、Flight 2 から導入された。

### 耐熱防護システム (TPS) — 上段再使用の難所

上段 Starship は、軌道速度（~7.5 km/s 級）からの再突入で激しい**空力加熱**を受ける。これを機体下面に貼った**セラミック耐熱タイル (heat shield tiles)** で防ぐ。

- 完全再使用の鍵は「タイルが再突入に耐え、かつ最小整備で再飛行できる」こと——Space Shuttle が苦しんだ点であり、SpaceX の最大の技術課題。
- 直近の試験では、タイルの挙動を確認するため**ヒートシールドを撮像するカメラ**を搭載するなど、データ取得を重視している[^starship-official]。

### 33基クラスタの制御 (33-Engine Cluster)

Super Heavy は中型エンジン Raptor を**33基**束ねて巨大推力を得る。設計思想は Falcon 9 の9基クラスタの極大化である。

- **数による推力**: 製造可能なサイズのエンジンを大量に並べ、単一巨大エンジンの開発リスクを避ける。
- **冗長**: 複数基の喪失に耐える（ただし基数が増えるほど統合・配管・始動シーケンスの複雑さが増す）。
- **難所**: 33基の同時始動・推力ベクタ制御・燃焼安定の協調。直近の Flight 12 ではブースタのブーストバック燃焼で計画した全基を点火できず、燃焼が早期終了して FAA の「事故 (mishap)」認定に至った[^sfn-faa][^mynews-faa]——多発化の統合難度が依然として開発の最前線にあることを示す。

## 信頼性・開発考察 (Reliability & Development)

> 本分野の「セキュリティ考察」を、開発途上の機体については**信頼性・開発プロセス**として読み替える。

### 反復開発の歴史 (Iterative Flight-Test History)

SpaceX の開発は「**ハードウェア潤沢・高頻度試験 (hardware-rich, fly-fail-fix)**」を特徴とする。地上で完璧を期すより、飛ばして実データで学ぶ。

| 試験 | 時期 | 狙いと結果（要約） |
|---|---|---|
| IFT-1 | 2023-04 | 初の統合飛行。上昇中に姿勢を失い飛行終了（複数エンジン不調・段分離前に破壊） |
| IFT-2 | 2023-11 | ホットステージング導入。段分離成功も両段とも喪失 |
| IFT-3〜4 | 2024 | 上段の再突入・着水へ前進、データ取得を拡大 |
| **Flight 5** | 2024-10 | **史上初の Super Heavy 空中キャッチ成功 (Mechazilla)** |
| Flight 7 | 2025-01 | ブースタはキャッチ成功、上段は喪失[^sfn-catch] |
| 〜IFT-11 | 2025-10 | V2 機体での飛行を重ね、キャッチを反復実証（3回成功） |
| **Flight 12** | 2026-05 | **初の Block 3 / V3・Raptor 3 飛行**。Starlink 模擬22基を放出。ブースタはブーストバック不調で FAA 事故認定→打ち上げ一時停止[^mynews-flight12][^sfn-faa] |

- **成功と失敗が混在するのが正常**: 各回の「喪失」は設計反復の入力であり、Mechazilla キャッチ成功（Flight 5）や V3 初飛行（Flight 12）といったマイルストンと並走する。
- **依然として開発途上**: Flight 12 の FAA 事故認定・打ち上げ一時停止が示すように、完全再使用（特に上段の再突入・両段の確実な回収）は未完であり、信頼性は飛行データの蓄積で詰めている段階[^sfn-faa]。

### Falcon 9 との信頼性プロセスの対比

| 観点 | Falcon 9（成熟） | Starship（開発途上） |
|---|---|---|
| 着陸成功率 | Block 5 で 99% 級[^reuse-doc] | キャッチは複数回成功も、両段の確実回収は未確立 |
| 主たる学習源 | 大量の実運用飛行 | 高頻度の試験飛行（fly-fail-fix） |
| 規制 | 定常運用 | FAA の事故調査・飛行許可がマイルストンを律速[^sfn-faa] |

## 結論 (Conclusions)

Starship / Super Heavy は、Falcon 9 が証明した再使用の論理を**両段・全機・最小整備**へ徹底した到達点である。完全再使用という一つの要求が、**ステンレス鋼**（広温度域×コスト×製造性）、**33基 Raptor クラスタ**（数による推力と冗長）、**Mechazilla 空中キャッチ**（脚を排した整備性）、**ホットステージング**（段間損失の回避）、**耐熱タイル**（上段再突入）という、相互に連関した設計判断を一斉に駆動している。

同時に、Flight 12 の FAA 事故認定が示すように、完全再使用——とりわけ**上段の再突入と両段の確実な回収**——は依然として未完の工学であり、SpaceX は「飛ばして壊して直す」反復でそれを詰めている。

### 3部作を貫くもの (The Thread)

本ドキュメント群は、航空機の基礎工学（4つの力・重量との戦い）から出発し、SpaceX を3つの断面で見てきた。

1. **エンジン (`spacex-rocket-engines.md`)** — Isp と推力重量比への執着。FFSC で効率の上限を、メタンで再使用と火星を狙う。
2. **再使用性 (`spacex-reusability.md`)** — 帰還を制御工学に翻訳し、頻度の経済で「捨てるより安く」を成立させる。
3. **Starship（本書）** — その論理を完全再使用へ徹底し、材料・形状・着陸方式まで作り替える。

貫くのは、航空工学と同じ骨格——**推進・構造・制御を、重量とコストと引き換えに、システムとして最適化する**——を、ロケットの過酷な熱・圧力・速度域で、かつ **「反復で素早く詰める」開発文化**とともに先鋭化する姿勢である。航空機が一世紀かけて型式証明という枠組みで信頼性を作り込んだのに対し、SpaceX は高頻度の飛行データでそれを加速しようとしている——その帰結は、これからの飛行試験が語る。

---

## 参考文献 (References)

> SpaceX 公式（車両・飛行試験ページ）と FAA 文書を一次資料とし、専門報道で補う。最新の飛行試験は流動的なため取得日を明記する。

**一次資料 (Primary — 公式・規制)**

[^starship-official]: [SpaceX — Starship（公式車両ページ）](https://www.spacex.com/vehicles/starship) ／ [Starship V3 アップデート](https://www.spacex.com/updates/starship-v3) — 両段完全再使用、Super Heavy ×33 / Starship ×6 Raptor、methalox、ステンレス鋼、直径9 m、ヒートシールド撮像カメラ等。Accessed: 2026-06-30.

[^sfn-faa]: [FAA requires SpaceX-led mishap investigation before resumption of Starship launches — Spaceflight Now (2026-05-27)](https://spaceflightnow.com/2026/05/27/faa-requires-spacex-led-mishap-investigation-before-resumption-of-starship-launches/) — Flight 12 のブースタ不調を FAA が「事故」認定、調査完了まで打ち上げ停止。Accessed: 2026-06-30.

[^sfn-catch]: [SpaceX successfully catches Super Heavy booster, loses Starship upper stage during Flight 7 — Spaceflight Now (2025-01-17)](https://spaceflightnow.com/2025/01/17/spacex-successfully-catches-super-heavy-booster-loses-starship-upper-stage-during-flight-7/) — Mechazilla による Super Heavy 空中キャッチの実証、上段喪失。Accessed: 2026-06-30.

**専門報道 (Specialist reporting)**

[^scientificamerican]: [Watch SpaceX launch Starship V3 — the tallest and most powerful rocket yet — Scientific American](https://www.scientificamerican.com/article/watch-spacex-launch-starship-v3-the-tallest-and-most-powerful-rocket-yet/) — V3 が史上最大・最強、Saturn V の約2倍の推力。Accessed: 2026-06-30.

[^satnews]: [SpaceX Debuts Starship V3: Redefining Heavy-Lift Launch Capability — SatNews (2026-05-14)](https://satnews.com/2026/05/14/spacex-debuts-starship-v3-redefining-heavy-lift-launch-capability/) — 全高 ~124 m、LEO 100 t 超（再使用）、ステンレス鋼。Accessed: 2026-06-30.

[^mynews-flight12]: [SpaceX launches 12th Starship test flight; booster fails — Spectrum News 13 (2026-05-21)](https://mynews13.com/fl/orlando/space/2026/05/21/spacex-starship-12-test) — Flight 12（初の V3・Raptor 3）、Starlink 模擬22基放出、ブースタ不調。Accessed: 2026-06-30.

[^mynews-faa]: [FAA grounds SpaceX's Starship after booster malfunction — Spectrum News 13 (2026-05-27)](https://mynews13.com/fl/orlando/space/2026/05/27/faa-grounds-spacexs-starship-after-booster-malfunction) — ブースタ不調による FAA の打ち上げ一時停止。Accessed: 2026-06-30.

**本リポジトリ内参照**

[^reuse-doc]: 本リポジトリ `spacex-reusability.md`（Block 5 着陸成功率 99% 級、再使用の経済性）。
