# 伝送線路とシグナルインテグリティ (Transmission Lines and Signal Integrity)

## 概要 (Overview)

電磁気学シリーズの第6回(最終回)。前回の集中定数近似は「配線の長さ ≪ 波長」を要求した。GHz 級の信号では基板上の数 cm がすでにこの条件を破る——配線はもはや「理想の導線」ではなく、信号が**波として伝わり、反射し、減衰する媒質**になる。これを扱う枠組みが**伝送線路理論 (transmission line theory)** であり、その工学的実践が**シグナルインテグリティ (signal integrity, SI)**——「送った波形が読める形で届くこと」の設計——である。

理論の骨格は3つの量に集約される。(1) **特性インピーダンス Z₀**——線路を進む波が見る電圧/電流比。(2) **伝播遅延**——信号は媒質中の光速でしか進まない。(3) **反射係数 Γ**——インピーダンスが変わる場所で波の一部が跳ね返る。「50 Ω に合わせる」「終端する」「スタブを短く」という高速設計の掟は、すべてこの3つから導かれる。後半では実在の線路の非理想性(表皮効果・誘電損失)が波形を劣化させる機構と、それに対抗する現代の高速シリアルリンク(SerDes)の技術——差動伝送・イコライゼーション——までを扱い、マクスウェル方程式から PCIe の物理層までの道をつなげてシリーズを閉じる。

## 前提知識 (Prerequisites)

- [`maxwell-equations-em-waves.md`](maxwell-equations-em-waves.md) — 波動方程式、「信号は場として伝わる」
- [`circuit-theory.md`](circuit-theory.md) — 集中定数近似の限界 (ℓ ≪ λ)、インピーダンス、テブナン等価
- [`../../mathematics/calculus-analysis/ordinary-differential-equations.md`](../../mathematics/calculus-analysis/ordinary-differential-equations.md) — 波形の重ね合わせと過渡応答

## 本論 (Main Content)

### 1. 分布定数と電信方程式 (distributed elements and the telegrapher's equations)

長い線路を微小区間 dx に刻むと、各区間が単位長あたりのインダクタンス L′ [H/m] と、対地(リターン導体)へのキャパシタンス C′ [F/m] を持つ——素子が線路全体に**分布**する。この LC 梯子の極限が**電信方程式 (telegrapher's equations)** であり、無損失なら電圧・電流は波動方程式に従う (Pozar 2011, Ch. 2)。

```text
∂V/∂x = −L′·∂I/∂t,   ∂I/∂x = −C′·∂V/∂t
⟹ 波動方程式。伝播速度 v = 1/√(L′C′) = c/√ε_r
   (FR-4 基板: v ≈ c/2 ≈ 15 cm/ns — 「1 ns で 15 cm」が基板設計の物差し)
```

第4回の「信号は導体周りの場として伝わる」の回路語訳である。集中定数(前回)との関係は連続的で、線路が波長より十分短ければ LC 梯子は1段で足り、集中定数に退化する——2つの理論は矛盾せず、**寸法/波長比**でつながっている。

### 2. 特性インピーダンス (characteristic impedance)

線路を一方向に進む波では、電圧と電流の比が場所によらず一定になる。

```text
Z₀ = √(L′/C′)   (無損失線路。抵抗ではなく「波が見る比率」)
同軸ケーブル: 50/75 Ω, PCB マイクロストリップ: 導体幅・誘電体厚で設計
```

Z₀ は**エネルギーを散逸しない**——抵抗と同じ単位を持つが、波を運んでいるだけである(無限長の線路は入口から見ると純抵抗 Z₀ に見える、という形でこの2つの顔がつながる)。実務上の意味は「配線の断面形状がインピーダンスを決める」こと: 導体幅、誘電体厚、リターンプレーンまでの距離を管理して Z₀ を一定(例: 50 Ω シングル、100 Ω 差動)に保つ**インピーダンス制御配線**が高速基板の基本仕様であり、基板メーカーへの製造指示はこの断面寸法の指定である。リターンパスがプレーンの分断で乱れると L′・C′ が急変し Z₀ が乱れる——「ベタグラウンドを切るな」という設計則の定量的な根拠になる。

### 3. 反射と終端 (reflection and termination)

インピーダンスが Z₀ から Z_L に変わる境界に波が到達すると、境界条件(電圧・電流の連続)を満たすために波の一部が反射する。

```text
反射係数: Γ = (Z_L − Z₀)/(Z_L + Z₀)
  整合 (Z_L = Z₀):  Γ = 0 — 反射なし。波は全て吸収される
  開放 (Z_L = ∞):   Γ = +1 — 全反射、電圧2倍 (未終端のバスで実際に起きる)
  短絡 (Z_L = 0):   Γ = −1 — 逆極性で全反射
```

反射波は線路を往復し、ドライバ側でも(出力インピーダンスが Z₀ と違えば)再反射する——この多重反射が、前回 §3 の2階系のリンギングとは別機構の**階段状・振動状の波形乱れ**を作る。往復時間が信号のエッジ時間より長い線路(=電気的に長い線路)では必ず顕在化し、対策が**終端 (termination)**——直列終端(ドライバ側に R を足して往路の再反射を殺す)、並列終端(受端を Z₀ で受ける)——である。コネクタ・ビア・分岐(スタブ)はすべて局所的なインピーダンス不連続=反射源であり、「スタブは短く、分岐は避け、ビアは減らす」という配線則は Γ の最小化と読める (Bogatin 2018, Ch. 7–8)。

### 4. 損失と符号間干渉 (loss, dispersion, and inter-symbol interference)

実在の線路は減衰し、しかも**減衰量が周波数に依存する**。

```text
表皮効果 (skin effect): 高周波電流は導体表面の深さ δ ∝ 1/√f にしか流れず、
  実効抵抗が √f で増える
誘電損失 (dielectric loss): 基板材料の分極損失。減衰が f にほぼ比例
⟹ 高周波成分ほど強く減衰・遅延 → エッジが鈍り、パルスが裾を引く
```

デジタル信号のエッジは高周波成分の集合体(フーリエ解析の視点)なので、この周波数依存減衰はパルスを時間方向に「にじませ」、前のビットの裾が次のビットに重なる——**符号間干渉 (inter-symbol interference, ISI)** が生じる。データレートが上がるほどビット時間が縮み、同じにじみが相対的に致命傷になる。SI の測定言語である**アイダイアグラム (eye diagram)**(全ビット遷移を重ね描きした図)は、この劣化を「目の開き」として一目で定量化する——目が閉じれば受信器はビットを判別できない (Hall & Heck 2009, Ch. 5)。

### 5. 高速シリアルリンクの対抗技術 (countermeasures: the SerDes toolkit)

現代の高速リンク (PCIe, USB, Ethernet, DDR I/F) は、§3–4 の物理的劣化を前提に、それを補償する技術の積層として設計されている。

```text
差動伝送 (differential signaling): 極性反転した2本で送り、受端で差を取る
  — 同相ノイズ・グラウンド変動をキャンセル。100 Ω 差動が標準
イコライゼーション (equalization): 線路の周波数特性の逆特性をかけて相殺
  送信側 FFE: エッジを事前に強調 (de-emphasis)
  受信側 CTLE: 高周波を持ち上げる連続時間フィルタ
  受信側 DFE: 判定済みビットから ISI を差し引く帰還フィルタ
その他: スクランブル (スペクトル平坦化)、クロック埋め込みと CDR、FEC
```

見取り図として重要なのは、これが**物理の制約とディジタル信号処理の交換**だという点である——銅線の損失は物理で決まっており減らせないから、送受信器のトランジスタ(計算)で波形を修復する。プロセス微細化で計算が安くなるほどこの交換は有利になり、PAM4(4値伝送)や FEC の採用など「より汚い波形を、より賢い受信器で読む」方向へ進化してきた。それでも銅の限界はデータレートとともに迫り、光インターコネクトへの移行圧力が生まれる——シリーズ第4回(電磁波)と本回が、その損益分岐の物理を与える。

## 工学への接続 (Engineering Connections)

- **高速規格の物理層仕様の読解**: PCIe/USB/Ethernet の電気仕様書に並ぶ用語——挿入損失 (insertion loss)、リターンロス、アイマスク、ジッタバジェット、イコライザ係数のネゴシエーション——は本回の概念の規格語訳であり、§1–5 が読解の辞書になる。
- **DDR メモリ配線則の由来**: 等長配線(スキュー=伝播遅延差の管理, §1)、インピーダンス制御 (§2)、ODT(オンダイ終端, §3)、fly-by トポロジ(スタブ回避, §3)——マザーボード設計規則の一つ一つが伝送線路理論の応用である。
- **車載・産業ネットワークの物理層**: CAN バスの両端 120 Ω 終端は §3 の並列終端そのもの。車載 Ethernet (10BASE-T1S 等) の1対差動線への集約は §5 の差動伝送+イコライゼーションの応用であり、ノイズだらけの車内環境で成立する理由でもある。
- **測定という設計行為**: TDR(時間領域反射測定)は「パルスを送って反射の到達時間と符号を読む」——Γ (§3) の実測であり、不連続の位置と種類(容量性か誘導性か)を波形から逆算する。VNA による S パラメータ測定は周波数側から同じ情報を取る。SI 設計はシミュレーションと測定がこの2つの言語で往復する。

## Daedalus内の応用 (Applications within Daedalus)

| 文書 | 接続点 |
|------|--------|
| [`hardware/signal-integrity/serdes-link-architecture.md`](../../../hardware/signal-integrity/serdes-link-architecture.md) | 本回の §5 を工学として展開する領域。符号化・PAM4・イコライゼーション・CDR・FEC |
| [`hardware/signal-integrity/signal-integrity-engineering.md`](../../../hardware/signal-integrity/signal-integrity-engineering.md) | 「測定という設計行為」の本体。ジッタ分解・S パラメータ予算・COM・TDR/VNA (§3 の実測) |
| [`hardware/hw-sw-interface/bus-interconnect.md`](../../../hardware/hw-sw-interface/bus-interconnect.md) | PCIe 等の高速シリアル物理層の全体 (§1–5) |
| [`hardware/peripherals/usb-architecture.md`](../../../hardware/peripherals/usb-architecture.md) | 差動ペア、終端、ケーブル損失とイコライゼーション (§3–5) |
| [`hardware/cpu/dram-memory-controllers.md`](../../../hardware/cpu/dram-memory-controllers.md) | DDR 配線則 (等長・ODT・fly-by) の伝送線路的由来 (工学への接続 §2)、信号トレーニング |
| [`engineering/automotive/in-vehicle-networks.md`](../../../engineering/automotive/in-vehicle-networks.md) | CAN の終端抵抗 (§3)、車載 Ethernet の差動物理層 (§5) |
| [`engineering/automotive/automotive-ethernet-tsn.md`](../../../engineering/automotive/automotive-ethernet-tsn.md) | 1対差動線での高速伝送を成立させる SI 技術 (§4–5) |
| [`hardware/networking/nic-switch-architecture.md`](../../../hardware/networking/nic-switch-architecture.md) | スイッチ間リンクの SerDes・FEC・PAM4 (§5)、銅と光の損益分岐 |
| [`hardware/peripherals/display-pipeline.md`](../../../hardware/peripherals/display-pipeline.md) | HDMI/DisplayPort の差動高速リンクとクロック埋め込み (§5) |

## 参考文献 (References)

- Bogatin, E. (2018). *Signal and Power Integrity — Simplified* (3rd ed.). Prentice Hall. — 反射・終端・実装設計則の工学側主参照
- Pozar, D. M. (2011). *Microwave Engineering* (4th ed.). Wiley. — 電信方程式・特性インピーダンス・S パラメータの理論側主参照
- Hall, S. H., & Heck, H. L. (2009). *Advanced Signal Integrity for High-Speed Digital Designs*. Wiley. DOI: 10.1002/9780470423899 — 損失・ISI・イコライゼーションの主参照
- Johnson, H., & Graham, M. (1993). *High-Speed Digital Design: A Handbook of Black Magic*. Prentice Hall. — 高速設計の古典的副読本
