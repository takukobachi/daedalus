# 光と半導体 (Optoelectronics)

## 概要 (Overview)

量子力学・固体物理シリーズの第5回(最終回)。バンドギャップ Eg は、これまで電気伝導の制御パラメータとして働いてきた。最終回はその**もう1つの顔**——Eg は光との交換レートである——を主題にする。光子エネルギー `E = hf` が Eg を超えれば半導体は光を吸収して電子・正孔対を作り(受光)、電子と正孔が再結合すれば Eg ぶんのエネルギーを光子として放てる(発光)。吸収・自然放出・**誘導放出**という3つの素過程の使い分けが、イメージセンサ・LED・半導体レーザーという3系統のデバイスを生む。

工学的な緊張関係は明快である: 受光は「光子を漏らさず数える」戦い(量子効率・暗電流・ショットノイズ)、発光は「注入した電子を漏らさず光子にする」戦い(直接遷移材料・ヘテロ構造による閉じ込め)、そして通信は「光を情報の運び手として銅の限界を超える」戦い(ファイバの低損失窓・レーザーの単色性・シリコンフォトニクス)。電磁気学シリーズ最終回(伝送線路)が示した銅の損失限界に、本回の光インターコネクトが答を与える——2本のシリーズの最終回同士が噛み合って、Daedalus の「信号の物理」の全体像が閉じる。

## 前提知識 (Prerequisites)

- [`band-theory-semiconductors.md`](band-theory-semiconductors.md) — バンドギャップ、直接/間接遷移(第1回で予告)
- [`pn-junction-diode.md`](pn-junction-diode.md) — 空乏層、フォトダイオードと LED の基本動作 (§5)
- [`../electromagnetism/maxwell-equations-em-waves.md`](../electromagnetism/maxwell-equations-em-waves.md) — 電磁波としての光、スペクトル
- [`../electromagnetism/transmission-lines-signal-integrity.md`](../electromagnetism/transmission-lines-signal-integrity.md) — 銅の損失限界(光への移行動機)

## 本論 (Main Content)

### 1. 光と電子の3つの素過程 (absorption, spontaneous and stimulated emission)

半導体中の電子と光子の相互作用は3過程に整理される (Saleh & Teich 2019, Ch. 14)。

```text
吸収 (absorption): 光子 (hf ≥ Eg) が価電子帯の電子を伝導帯へ蹴り上げる
  → 電子・正孔対の生成 (受光デバイスの原理)
自然放出 (spontaneous emission): 電子と正孔が自発的に再結合し、
  hf ≈ Eg の光子をランダムな位相・方向に放つ (LED の原理)
誘導放出 (stimulated emission): 既存の光子が再結合を誘発し、
  同位相・同方向・同波長のコピー光子を放つ (レーザーの原理)
```

発光には決定的な材料条件がある——**直接遷移型バンドギャップ**。運動量保存のため、間接遷移型の Si では再結合に格子振動の助けが要り、発光効率が桁で落ちる(第1回で予告した「Si は光らない」の理由)。ゆえに発光デバイスは GaAs・InP・GaN 系の化合物半導体が担い、受光は(吸収は間接遷移でも起きるので)Si で広く賄える——**「受光は Si、発光は化合物」という産業の分業構造**は、バンド構造の運動量空間の形という、最も基礎的な物理に由来する。

### 2. 受光 — 光子を数える (photodetection: counting photons)

フォトダイオード(第2回 §5)を「測定器」として見ると、性能は3つの量で決まる (Sze & Ng 2007, Ch. 13)。

```text
量子効率 η: 入射光子のうち電子・正孔対になり収集される割合
暗電流:    光ゼロでも流れる熱起源の電流 (Eg と温度に指数依存 — 第1回 §3)
ショットノイズ: 光電流の揺らぎ。光子計数はポアソン過程 (確率・統計第2回 §3)
  であり、N 個の光子に対し S/N ∝ √N — 光の量子性が課す統計限界
```

イメージセンサの画素はこの3量の最適化問題である: 画素を大きくすれば集光量 N が増え √N 則で S/N が改善する(スマートフォンの「大型センサ」競争の物理的根拠)。暗電流は露光時間に比例して蓄積し、温度で指数的に増える(天体撮影でセンサを冷やす理由)。ピン止めフォトダイオード+転送ゲートという現代 CMOS センサの画素構造は、暗電流源となる界面を空乏層から遠ざけ、完全電荷転送で読出ノイズを抑える工夫であり、微弱光では**1光子単位の計数**(SPAD: なだれ降伏——第2回 §4——を利用)まで到達している。低照度画質の限界がデバイスでなく**光子統計そのもの**に迫っている、というのが現代センサの到達点である。

### 3. 発光 — LED とヘテロ構造 (light emission: LEDs and heterostructures)

LED は順バイアスの pn 接合で注入された少数キャリアの再結合発光(第2回 §5)だが、実用効率への道はもう1つの発明——**ヘテロ構造 (heterostructure)**——が拓いた。Eg の異なる半導体を積層すると、バンドの段差でキャリアを狭い層(量子井戸)に**閉じ込め**られる。

```text
閉じ込め → 電子・正孔の密度が局所的に上がる → 再結合(発光)が加速
量子井戸の厚さ (~nm) → 量子化 (第1回 §1) により発光波長も微調整できる
```

「発光する場所・キャリア・光を同じ数 nm に閉じ込める」というこの設計原理は、バンドギャップ・エンジニアリングの精華である。波長は材料系で決まり(AlGaInP: 赤〜黄、InGaN: 青〜緑)、長年欠けていた高効率の青色 LED は GaN 系の結晶成長と p 型ドーピングの困難を突破して実現された(2014年ノーベル物理学賞)——青が揃って白色照明(青 LED + 蛍光体)が成立し、照明の電力効率を一変させた。ディスプレイでは、OLED(有機半導体の再結合発光)と、無機 LED を画素サイズに刻む microLED が自発光画素の2路線をなす——「バックライトの選択透過(液晶)」から「画素自体の発光」への移行は、コントラストと効率の物理的優位に基づく。

### 4. 半導体レーザー — 誘導放出を優勢にする (semiconductor lasers)

誘導放出 (§1) を主役にするには2つの条件を同時に満たす必要がある (Coldren, Corzine & Mašanović 2012)。

```text
反転分布 (population inversion): 伝導帯に電子が「詰まって」いる状態を
  強い注入で維持 — 吸収より誘導放出が勝つ条件
光共振器 (optical cavity): 鏡 (劈開面や回折格子) で光を往復させ、
  誘導放出の連鎖 = 光増幅を1つのモードに集中させる
⟹ 利得が損失を超えた瞬間 (しきい値電流) にレーザー発振
```

出てくる光は LED と対照的である——単一波長・位相の揃った(コヒーレント)・細く絞れるビーム。この「規律の取れた光」が、光ファイバ通信(色分散を抑え、波長多重を可能にする)、LiDAR(ビームの指向性とパルス制御)、光ピックアップ、原子時計・原子物理の操作光源を成立させる。通信用には、回折格子で単一波長を選ぶ DFB レーザー、面から垂直に発振し2次元アレイ化できる VCSEL(データセンターの短距離リンクと顔認証の投光に偏在)という2つの形態が主力である。

### 5. 光通信とシリコンフォトニクス (optical links and silicon photonics)

光を通信に使う根拠は石英ファイバの驚異的な低損失にある。

```text
石英ファイバ損失: ~0.2 dB/km @1550 nm (100 km で 1/100 に減る程度)
銅の高速リンク: ~数十 cm–数 m で等化が必要 (電磁気学第6回 §4)
⟹ 距離×帯域で銅を桁違いに凌駕。加えて波長多重 (WDM) で1本に多重化
```

リンクの構成は「レーザー (§4) → 変調 → ファイバ → フォトダイオード (§2)」であり、本回の全部品がここに並ぶ。現代の焦点は**シリコンフォトニクス**——導波路・変調器・受光器(Ge)を CMOS プロセスで集積する技術である。皮肉な構図がその本質を突いている: Si は 1550 nm に対して**透明**(Eg = 1.12 eV > 光子エネルギー 0.8 eV——第1回の表の帰結)だから低損失の導波路になれるが、同じ理由で光源になれず、レーザーは III-V 族の貼り合わせで外付けする。データセンターのスイッチ帯域が電気 SerDes の限界(電磁気学第6回 §5)に迫るにつれ、光をパッケージ内へ引き込む**コパッケージド・オプティクス (CPO)** が銅と光の境界線を押し込んでいる——「どこから先を光にするか」は、損失・電力・コストの損益分岐が決める移動境界である (Agrawal 2010; Saleh & Teich 2019)。

## 工学への接続 (Engineering Connections)

- **カメラ画質の物理限界の読み方**: 画素サイズ・量子効率・暗電流・ショットノイズ (§2) は、スペックシートの「低照度性能」を第一原理から評価する語彙である。計算写真学(複数フレーム合成)は「N を時間方向に稼いで √N 則を延命する」統計的対策と読める。
- **銅 vs 光の損益分岐**: リンク距離×データレートの平面で、等化コスト(電磁気学第6回)と光変換コスト (§5) の交点が技術選択を決める——ラック内は銅、ラック間から光、そしてパッケージ内へ、と境界が短距離側に移動し続ける趨勢の物理的読み方。
- **照明・ディスプレイの効率革命**: 白熱灯(黒体放射の裾)→ 蛍光灯 → LED (§3) の系譜は「発光をバンド遷移1本に集中させる」進化であり、電力・発熱の分析(データセンター、車載)における照明負荷の激減として現れる。
- **LiDAR とセンシング**: パルスレーザー (§4) + 高感度受光 (SPAD, §2) + 飛行時間計測(光速——電磁気学第4回)という構成は、本シリーズと電磁気学シリーズの部品だけで組み立てられる——車載・ロボティクスのセンサ分析の物理的土台。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/peripherals/camera-image-sensors.md`](../../../hardware/peripherals/camera-image-sensors.md) | 本回はこの文書のデバイス物理層——量子効率・暗電流・ショットノイズ・画素構造 (§2) |
| [`hardware/peripherals/display-pipeline.md`](../../../hardware/peripherals/display-pipeline.md) | LED バックライト・OLED・microLED の発光原理 (§3)、色と Eg の対応 |
| [`hardware/networking/nic-switch-architecture.md`](../../../hardware/networking/nic-switch-architecture.md) | 光トランシーバ・VCSEL 短距離リンク・CPO への趨勢 (§4–5) |
| [`hardware/hw-sw-interface/bus-interconnect.md`](../../../hardware/hw-sw-interface/bus-interconnect.md) | 電気 SerDes の限界の先にある光インターコネクト (§5) |
| [`hardware/timekeeping/clocks-time-synchronization.md`](../../../hardware/timekeeping/clocks-time-synchronization.md) | レーザーによる原子の操作・光格子時計への展望 (§4)、光ファイバによる時刻配信 |
| [`engineering/automotive/ee-architecture.md`](../../../engineering/automotive/ee-architecture.md) | LiDAR・カメラという光センサ群の物理的基盤 (工学への接続 §4) |
| [`hardware/semiconductor-manufacturing/semiconductor-fabrication.md`](../../../hardware/semiconductor-manufacturing/semiconductor-fabrication.md) | シリコンフォトニクスの CMOS 統合 (§5)、化合物半導体の異種集積 |

## 参考文献 (References)

- Saleh, B. E. A., & Teich, M. C. (2019). *Fundamentals of Photonics* (3rd ed.). Wiley. — 吸収・放出の素過程、フォトニクス全域の主参照
- Sze, S. M., & Ng, K. K. (2007). *Physics of Semiconductor Devices* (3rd ed.). Wiley. DOI: 10.1002/0470068329 — フォトダイオード・LED・レーザーのデバイス物理の主参照
- Coldren, L. A., Corzine, S. W., & Mašanović, M. L. (2012). *Diode Lasers and Photonic Integrated Circuits* (2nd ed.). Wiley. DOI: 10.1002/9781118148167 — 半導体レーザー・集積フォトニクスの主参照
- Agrawal, G. P. (2010). *Fiber-Optic Communication Systems* (4th ed.). Wiley. DOI: 10.1002/9780470918524 — 光ファイバリンク・WDM の主参照
- Nakamura, S. (2015). Nobel Lecture: Background story of the invention of efficient blue InGaN light emitting diodes. *Reviews of Modern Physics*, 87(4), 1139–1151. DOI: 10.1103/RevModPhys.87.1139 — 青色 LED 実現の一次資料
