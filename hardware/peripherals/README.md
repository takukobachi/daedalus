# 周辺デバイスと接続 (Peripherals & Connectivity)

SoC と外界をつなぐ周辺デバイスの中身と、その接続技術。`hw-sw-interface/`（バス・DMA・割り込み）の一段上に乗り、身近な機器（ゲームコントローラー、端末のオーディオ、Bluetooth 機器）を題材に「デバイスそのもの」を解剖する。`software/android/`・`software/ios/` の入力・オーディオスタックのハードウェア側の対になる。

## ドキュメント

- [`input-devices-hid.md`](input-devices-hid.md) — HIDとゲームコントローラー: 入力デバイスの解剖 (HID & game controllers)
- [`keyboard-mouse-touch.md`](keyboard-mouse-touch.md) — キーボード・マウス・タッチ: キーとポインタの物理 (keyboard, mouse & touch)
- [`input-latency.md`](input-latency.md) — 入力遅延: 操作から画面までの時間 (input latency)
- [`sensor-hub-mems.md`](sensor-hub-mems.md) — センサハブとMEMSセンサ: 常時オンの感覚器 (sensor hub & MEMS)
- [`audio-subsystem.md`](audio-subsystem.md) — オーディオサブシステム: 端末の音の出入り (audio subsystem)
- [`display-pipeline.md`](display-pipeline.md) — ディスプレイパイプライン: フレームバッファからパネルへ (display pipeline)
- [`camera-image-sensors.md`](camera-image-sensors.md) — カメラとイメージセンサ: 光子からピクセルまで (camera & image sensors)
- [`usb-architecture.md`](usb-architecture.md) — USBアーキテクチャ: 有線接続の背骨 (USB architecture)
- [`bluetooth-pairing-protocol.md`](bluetooth-pairing-protocol.md) — Bluetoothとペアリング: 無線スタックの解剖 (Bluetooth & pairing)
- [`wifi-architecture.md`](wifi-architecture.md) — Wi-Fi (802.11): 無線LANの解剖 (Wi-Fi architecture)
- [`nfc-contactless-payments.md`](nfc-contactless-payments.md) — NFCと非接触決済: 数センチの無線と、その上の暗号 (NFC & contactless payments)

読む順は上から。入力 (HID) と音声 (USB Audio / A2DP・LE Audio) は、有線の背骨 (USB) と
無線の背骨 (Bluetooth / Wi-Fi) の伝送路の上で合流する構成になっている。表示 (display) と
撮像 (camera) は専用リンク (DSI / CSI-2 / eDP / DP / HDMI) を使う点でこれらと対照的な
「データ→光」「光→データ」の対であり、音声と同型の「締切付きストリーム」問題を
3桁大きい帯域で解く層として読む。
NFC は、同じ無線でも数センチの磁界結合という別の物理に乗り、その上の決済の暗号が
本体になる層として、無線の背骨の後に読む。

