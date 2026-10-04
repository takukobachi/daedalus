# 電気自動車 (EV / Electric Vehicles)

電気自動車を題材とした工学分析。EV パワートレインの基礎工学（電池・電動機・電力変換・回生・充電・熱管理）を土台に、**テスラ (Tesla)** の実装（バッテリ/4680・パワートレイン・E/E アーキ・製造）へ深掘りする。[`../in-vehicle-networks.md`](../in-vehicle-networks.md)（車載ネットワーク）・[`../functional-safety-iso26262.md`](../functional-safety-iso26262.md)（機能安全）が「制御と通信」を扱うのに対し、本サブディレクトリは「電動化そのもの（エネルギーと動力、作り方）」を扱う。

## ドキュメント

### 基礎 (Fundamentals)

- [`ev-fundamentals.md`](ev-fundamentals.md) — 電気自動車の基礎工学 (EV fundamentals)：電池（セル化学/形状/パック）、電動機（PMSM/誘導機）、電力変換（インバータ/SiC）、回生、充電（AC/DC・400V/800V）、BMS・熱、ICE との効率比較

### テスラ (Tesla)

- [`tesla-battery-powertrain.md`](tesla-battery-powertrain.md) — バッテリとパワートレイン：4680（タブレス/乾式電極）、構造バッテリパック、IPM-SynRM＋誘導機の使い分け、量産初の全 SiC インバータ
- [`tesla-software-manufacturing.md`](tesla-software-manufacturing.md) — ソフトウェアと製造：集中型 E/E＋ゾーン＋車載 Ethernet、OTA、独自シリコン（FSD コンピュータ）、ギガキャスト、アンボックス製造、垂直統合

## 今後のテーマ候補

- 充電インフラ (charging network) — Supercharger / NACS、系統連系、V2G
- 熱マネジメント深掘り — ヒートポンプ、オクトバルブ、プレコンディショニング

自動運転 (ADAS / autonomy) は EV 固有ではなく車両全体のテーマのため、親ディレクトリ
([`../README.md`](../README.md)) のロードマップで扱う（本サブディレクトリでは E/E 基盤に限定して言及）。
