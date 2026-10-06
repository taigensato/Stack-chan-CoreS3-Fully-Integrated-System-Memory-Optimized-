<img width="1536" height="2048" alt="stack_chan_elizalike" src="https://github.com/user-attachments/assets/a574a296-a3bc-4e42-9841-831ccaa1c3ff" />
# Stack-chan-CoreS3-Fully-Integrated-System-Memory-Optimized-

M5Stack CoreS3をベースにした、完全オフライン動作のスタックちゃん統合ファームウェアです。外部ASR（音声認識）モジュール、AquesTalk Picoによる音声合成、SDカードからの高音質音楽再生、そしてM5Avatarとサーボモーターによる感情表現を、ESP32の内部SRAMのみ（PSRAM非依存）で安定稼働させるよう極限まで最適化されています。

## ✨ 主な機能 (Features)

* **完全オフライン会話システム (MicroDomainASR + ELIZA)**
* 外部ASRモジュールからのUARTバイナリパケット（`0xAA 0x55 ID 0x55 0xAA`）を解析。
* 軽量なELIZA風ルールベースエンジンを搭載。キーワードに対する重み付け（優先度）、連続被り回避、話題の記憶とフォールバック処理を実装。
* AquesTalk Pico（CAqTkPicoF）によるリップシンク付き音声合成。


* **ノイズフリー高音質オーディオプレイヤー**
* M5UnifiedのI2S出力をカスタマイズし、バッファリングを最適化。
* **PWM Detach制御**: 音楽再生中（`MODE_PLAYBACK`）はサーボのPWMパルスを物理的に切断（`ledcDetach`）し、スピーカーへの電気的ノイズ（ジー、ブチブチ音）を完全に遮断。


* **FreeRTOS マルチコアアーキテクチャ**
* **Core 0**: MP3/WAVの重いデコード処理（`audioTask`）。
* **Core 1**: サーボ制御、アバター描画、呼吸のようなアイドリングモーション（`servoAvatarTask`）。


* **インタラクティブなUIとセンサー連携**
* タッチパネルによる直感的な操作（音量調整、音楽再生/停止）。
* IMUセンサーを活用し、本体が揺らされると反応してビープ音とお辞儀モーションを実行。

## 🛠 ハードウェア構成 (Hardware Setup)

| コンポーネント | 接続先 / ピン番号 | 備考 |
| --- | --- | --- |
| **M5Stack CoreS3** | Main Unit | 内部マイク・スピーカー使用 |
| **Servo (Pan/左右)** | GPIO 1 | PWM制御 (50Hz) / Center: 74 |
| **Servo (Tilt/上下)** | GPIO 2 | PWM制御 (50Hz) / Center: 64 |
| **ASR Module (UART)** | Port C (RX: 18, TX: 17) | 115200bps / 独自バイナリパケット通信 |
| **MicroSD Card** | SPI (CS: GPIO 4) | `/` 直下の `.mp3`, `.wav` を自動スキャン |

## 📦 依存ライブラリ (Dependencies)

Arduino IDEのライブラリマネージャー、またはZIPから以下のライブラリをインストールしてください。

* **M5Unified**
* **M5Avatar**
* **ESP32-audioI2S** (AudioGeneratorMP3 / AudioGeneratorWAV / AudioFileSourceSD)
* **AquesTalk Pico** (CAqTkPicoF_ESP32)
* ライセンス (License)

This project is licensed under the MIT License - see the LICENSE file for details.
*(※AquesTalk Pico 等の外部ライブラリのライセンスは、それぞれの提供元の規約に従ってください。)*

## 🎮 操作方法 (Touch Controls)

画面のタッチ領域によって以下の操作が可能です。

* **画面上半分 (Y < 120)**
* 左側 (X < 160): ボリューム DOWN
* 右側 (X >= 160): ボリューム UP


* **画面下半分 (Y >= 120)**
* 左側 (X < 106): 音楽ランダム再生開始 (`MODE_PLAYBACK`)
* 中央〜右側 (X >= 106): 音楽停止 ＆ 待機モードへ移行 (`MODE_IDLE`)



## ⚠️ 開発者向けTips / 既知の仕様 (Important Notes)

**1. AquesTalk Picoのエラー105 (強制終了) 対策**
ローマ字文字列（`romaji`）に `!`（感嘆符）が含まれると、AquesTalkエンジンが未定義記号としてエラー105を吐き、音声合成が強制中断されます。感情を表現する場合は `.` (句点) や `?` (疑問符) で代用するよう `PSYCHOBABBLE` 構造体の設計がされています。ルールを追加する際はローマ字側に `!` を入れないよう注意してください。

**2. PSRAMエラーとArduino IDEのキャッシュの罠**
ESP32-S3環境において、コンパイル時に `octal_psram: PSRAM chip is not connected` や `mode:DIO` のエラーでクラッシュを繰り返す場合、Arduino IDEが古いブートローダのキャッシュを握っている可能性があります。

* **対策**: IDEを閉じ、OSの一時フォルダ（Windowsなら `%TEMP%`、Macなら `$TMPDIR`）内の `arduino` または `arduino_build` で始まるキャッシュフォルダをすべて手動で削除してからフルビルドしてください。本コードはPSRAMが無効な状態でも、内部SRAMのみで安全に動作するよう設計されています。

**3. メモリの動的確保の禁止**
会話エンジン（`analyze`関数内）で `std::vector` などの動的配列を `static` に初期化しようとすると、例外処理がキャッチできず `LoadProhibited` パニックを起こします。機能拡張時は必ずC言語の固定長配列（例: `int last_response_idx[16]`）を使用してください。
