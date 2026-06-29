# Power Generation

電力・発電を題材としたハードウェア分析。一次エネルギーから電気への変換チェーンと、それを支える電力系統 (power system) を扱い、物理プラントが最終的に制御システム (SCADA/DCS) というソフトウェアに集約される接点までを追う。

チップ内の電源（[`../hw-sw-interface/clock-power-reset.md`](../hw-sw-interface/clock-power-reset.md)）が数ワット規模の動的電力を扱うのに対し、本ディレクトリは同じ「電力」をギガワット規模、約12桁上のスケールで扱う。

## ドキュメント

- [`generation-and-grid-fundamentals.md`](generation-and-grid-fundamentals.md) — 発電と電力系統の基礎 (power generation & grid fundamentals)
- [`plant-control-scada-security.md`](plant-control-scada-security.md) — 発電所・系統の制御システムとセキュリティ (plant control: SCADA/DCS/IEC 61850 & security)

## 今後のトピック

- 電力変換 (power electronics) — インバータ、PWM、再エネ連系
- 系統安定度と周波数制御 (grid stability & frequency control) — 慣性、AGC、連鎖故障の定量モデル
