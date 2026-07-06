# 周辺デバイスと接続 (Peripherals & Connectivity)

SoC と外界をつなぐ周辺デバイスの中身と、その接続技術。`hw-sw-interface/`（バス・DMA・割り込み）の一段上に乗り、身近な機器（ゲームコントローラー、端末のオーディオ、Bluetooth 機器）を題材に「デバイスそのもの」を解剖する。`software/android/`・`software/ios/` の入力・オーディオスタックのハードウェア側の対になる。

## ドキュメント

- [`input-devices-hid.md`](input-devices-hid.md) — 入力デバイスとHID: ゲームコントローラーの解剖 (input devices & HID)
- [`audio-subsystem.md`](audio-subsystem.md) — オーディオサブシステム: 端末の音の出入り (audio subsystem)
- [`display-pipeline.md`](display-pipeline.md) — ディスプレイパイプライン: フレームバッファからパネルへ (display pipeline)
- [`usb-architecture.md`](usb-architecture.md) — USBアーキテクチャ: 有線接続の背骨 (USB architecture)
- [`bluetooth-pairing-protocol.md`](bluetooth-pairing-protocol.md) — Bluetoothとペアリング: 無線スタックの解剖 (Bluetooth & pairing)

読む順は上から。入力 (HID) と音声 (USB Audio / A2DP・LE Audio) は、有線の背骨 (USB) と
無線の背骨 (Bluetooth) の2本の伝送路の上で合流する構成になっている。表示 (display) は
専用リンク (DSI / eDP / DP / HDMI) を使う点でこの2本と対照的で、音声と同型の
「締切付きストリーム」問題を3桁大きい帯域で解く層として読む。

## 今後のテーマ候補

- カメラとイメージセンサ (camera & image sensors) — CMOS センサの物理、MIPI CSI、
  ISP パイプライン。モバイルプラットフォーム分析の撮像側の土台
- Wi-Fi (802.11) — 無線のもう一本の背骨。PHY/MAC、CSMA/CA、Wi-Fi 6/7。近距離
  (Bluetooth, 本ディレクトリ) と広域 (5G/GNSS, トップレベル候補「無線通信」) の間を埋める
