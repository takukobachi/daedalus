# エネルギー・電力 (Energy)

電力・発電を題材とした工学分析。一次エネルギーから電気への変換チェーンと、それを支える電力系統 (power system) を扱い、物理プラントが最終的に制御システム (SCADA/DCS) というソフトウェアに集約される接点までを追う。

チップ内の電源（[`../../hardware/hw-sw-interface/clock-power-reset.md`](../../hardware/hw-sw-interface/clock-power-reset.md)）が数ワット規模の動的電力を扱うのに対し、本ディレクトリは同じ「電力」をギガワット規模、約12桁上のスケールで扱う。

## ドキュメント

- [`generation-grid-fundamentals.md`](generation-grid-fundamentals.md) — 発電と電力系統の基礎 (power generation & grid fundamentals)
- [`plant-control-scada-security.md`](plant-control-scada-security.md) — 発電所・系統の制御システムとセキュリティ (plant control: SCADA/DCS/IEC 61850 & security)
- [`electricity-markets-dispatch.md`](electricity-markets-dispatch.md) — 電力市場と経済的ディスパッチ (electricity markets & economic dispatch)。周波数制御の階層が市場商品（需給調整市場・スポット・容量市場）へ写像される構造
- [`battery-energy-storage.md`](battery-energy-storage.md) — 系統用蓄電システム (battery energy storage, BESS)。セル〜PCS〜EMS の階層、LFP への収斂、レベニュースタッキング、Hornsdale / Moss Landing 事例
- [`hvdc-interconnection.md`](hvdc-interconnection.md) — 直流送電と地域間連系 (HVDC & interconnection)。LCC/VSC、日本の周波数変換設備 (FC)・北本連系、市場分断との接続
- [`power-electronics-grid-converters.md`](power-electronics-grid-converters.md) — 電力変換と系統連系インバータ (power electronics & grid-connected converters)。Si IGBT / SiC / GaN、2 レベル・NPC・MMC、PWM、損失と熱設計、IEEE 519、グリッドフォロイング/フォーミング制御、弱い系統
- [`grid-stability-frequency-control.md`](grid-stability-frequency-control.md) — 系統安定度と周波数制御 (power system stability & frequency control)。動揺方程式・RoCoF・最下点、一次〜三次制御と AGC、IEEE/CIGRE 安定度分類、UFLS、5 大停電の比較、連鎖故障モデル

## 今後のテーマ候補

物理（発電・系統）から事業（市場・運用）までを一つの連続したシステムとして扱う。特に「周波数制御の階層が市場商品として値付けされる」接点を軸に、電力事業側へ展開する。

### 事業・市場との接点 (grid economics & operation)

- 送配電網 (transmission & distribution) — 変電所、潮流計算、N-1 基準、系統混雑とノンファーム型接続、託送

### 分散化・需要側 (DER & demand side)

- 分散型エネルギー資源と VPP (DER / virtual power plant) — アグリゲーション、DERMS、デマンドレスポンス、OpenADR
- スマートメーターと AMI (advanced metering infrastructure) — 計量ハードウェア、Wi-SUN/PLC 通信、MDMS、セキュリティ

### 発電方式の深掘り

- 原子力発電 (nuclear power) — 炉型、多重防護、安全計装系（[`plant-control-scada-security.md`](plant-control-scada-security.md) の SIS と接続）
- 火力の高度化 (advanced thermal) — コンバインドサイクル、ガスタービン、水素・アンモニア混焼
- 風力・太陽光のハードウェア (wind & solar hardware) — 可変速風車 (DFIG/フルコンバータ)、ピッチ制御、PV と MPPT
