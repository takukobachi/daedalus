# 周辺デバイスと接続 (Peripherals & Connectivity)

SoC と外界をつなぐ周辺デバイスの中身と、その接続技術。`hw-sw-interface/`（バス・DMA・割り込み）の一段上に乗り、身近な機器（ゲームコントローラー、端末のオーディオ、Bluetooth 機器）を題材に「デバイスそのもの」を解剖する。`software/android/`・`software/ios/` の入力・オーディオスタックのハードウェア側の対になる。

## ドキュメント

- [`input-devices-hid.md`](input-devices-hid.md) — 入力デバイスとHID: ゲームコントローラーの解剖 (input devices & HID)
- [`audio-subsystem.md`](audio-subsystem.md) — オーディオサブシステム: 端末の音の出入り (audio subsystem)
- [`display-pipeline.md`](display-pipeline.md) — ディスプレイパイプライン: フレームバッファからパネルへ (display pipeline)
- [`camera-image-sensors.md`](camera-image-sensors.md) — カメラとイメージセンサ: 光子からピクセルまで (camera & image sensors)
- [`usb-architecture.md`](usb-architecture.md) — USBアーキテクチャ: 有線接続の背骨 (USB architecture)
- [`bluetooth-pairing-protocol.md`](bluetooth-pairing-protocol.md) — Bluetoothとペアリング: 無線スタックの解剖 (Bluetooth & pairing)
- [`wifi-architecture.md`](wifi-architecture.md) — Wi-Fi (802.11): 無線LANの解剖 (Wi-Fi architecture)

読む順は上から。入力 (HID) と音声 (USB Audio / A2DP・LE Audio) は、有線の背骨 (USB) と
無線の背骨 (Bluetooth / Wi-Fi) の伝送路の上で合流する構成になっている。表示 (display) と
撮像 (camera) は専用リンク (DSI / CSI-2 / eDP / DP / HDMI) を使う点でこれらと対照的な
「データ→光」「光→データ」の対であり、音声と同型の「締切付きストリーム」問題を
3桁大きい帯域で解く層として読む。

## 今後のテーマ候補

- NFC と非接触決済 (NFC & contactless) — 13.56 MHz 近接結合、カードエミュレーション、
  セキュアエレメント/HCE。決済という「セキュリティが本体」のデバイス層
- センサハブと MEMS センサ (sensor hub & MEMS) — IMU・気圧・磁気センサの物理、
  I2C/SPI/I3C 接続、AP を起こさない常時オン処理。オーディオ (wake word)・
  カメラ (AOSC) と同じ低電力監視パターンの第三例
