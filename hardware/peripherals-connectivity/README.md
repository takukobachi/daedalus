# 周辺デバイスと接続 (Peripherals & Connectivity)

SoC と外界をつなぐ周辺デバイスの中身と、その接続技術。`hw-sw-interface/`（バス・DMA・割り込み）の一段上に乗り、身近な機器（ゲームコントローラー、端末のオーディオ、Bluetooth 機器）を題材に「デバイスそのもの」を解剖する。`software/android/`・`software/ios/` の入力・オーディオスタックのハードウェア側の対になる。

## ドキュメント

- [`input-devices-hid.md`](input-devices-hid.md) — 入力デバイスとHID: ゲームコントローラーの解剖 (input devices & HID)
- [`audio-subsystem.md`](audio-subsystem.md) — オーディオサブシステム: 端末の音の出入り (audio subsystem)
- [`bluetooth-pairing-protocol.md`](bluetooth-pairing-protocol.md) — Bluetoothとペアリング: 無線スタックの解剖 (Bluetooth & pairing)

読む順は上から。入力 (HID) と音声 (A2DP/LE Audio) は、3本目の Bluetooth スタックの最上層で合流する構成になっている。
