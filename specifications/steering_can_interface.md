# Steering CAN Interface — Shared Status

更新日: 2026-10-05  
文書状態: **Draft / 未凍結**  
対象: Cockpit Steering Board / Tail Steering Board / Measurement Board

---

## 1. この文書の役割

本書は、3基板で共通に扱う操舵CANの「確定済み範囲」と「未決定範囲」を一か所にまとめる。個別基板仕様と記述が衝突した場合、本書の状態表示を優先する。

## 2. 確定済みの物理層

- CAN 2.0、500 kbps
- Cockpit、Tail、Measurementの3ノードを同一バスへ接続
- CockpitとTailを物理端とし、各120 Ω終端
- MeasurementはCockpit付近から目標0.3 m以下の短いスタブ、終端なし
- CANH / CANLはAWG24ツイストペア、GNDを併走
- コネクタはJST XA 4極
  - Pin 1: CANH
  - Pin 2: CANL
  - Pin 3: CAN_GND
  - Pin 4: NC（誤挿入防止用キー極）
- トランシーバはTCAN3413DR
- TVSはPESD2CANFD24V-T
- CMCはフットプリントのみ、初期DNI・0 Ωバイパス
- CockpitとTailはGPIO4=TX、GPIO5=RX、GPIO40=STB
- STBはpull-upで起動時Standby、TWAI初期化後にNormalへ移行

## 3. 確定済みの機能条件

- Measurementが未接続、電源OFF、再起動中でもCockpit–Tail操舵が成立する
- Measurementの応答や時刻同期完了を操舵成立条件にしない
- CockpitからTailへの操舵指令は約50 Hz
- 正規化操舵値はint16_t、-10000〜+10000を-100〜+100 %として扱う
- sequenceはuint16_tを基本とする
- Cockpitからの時刻同期は約10 Hz
- CAN timeout初期値は300 ms
- timeout後はTailが最後の有効trim位置へ約0.7 sでsmooth return
- 通信復帰時はrate-limited recovery
- USB給電でCockpitが起動した場合も有効指令を送信し、サーボが動く可能性を許容する

## 4. 現在の候補（未凍結）

- 11-bit standard ID
- Node ID: Reserved=0、Cockpit=1、Tail=2、Measurement=3
- ID構成: Priority 2 bit / Node ID 4 bit / Message Type 5 bit
- Cockpit仕様書に記載された0x020〜0x223のID案
- Steering Stick Command、Trim Command、Power Status、Time Syncの各payload案

これらは回路図作成を妨げない範囲の作業案であり、ファームウェア共通ヘッダ作成時に確定する。

## 5. 未決定事項

- 全CAN IDと優先順位
- byte order
- signed値、単位、固定小数点スケール
- validity / armed / fault flagのビット割当
- sequence wrapと片側再起動時の扱い
- boot_id / session_idの要否
- trim状態の正本と再同期手順
- Rudder / Elevatorの正方向
- Tail status、実舵角、電流、faultのpayload
- Measurementノードの送信周期・帯域上限
- protocol versionと設定version / CRC
- bus-off復帰方針とerror counterの診断送信

## 6. 結合試験で確認する故障

- Measurement未接続
- Measurement電源OFF
- Measurementの連続再起動
- Measurement送信過多
- Measurement枝のCANHまたはCANL断線
- Measurement枝のCANH–CANL短絡
- Measurement枝のCAN線–GND短絡
- Cockpit USB接続中のTail起動・停止
- 全ノード接続時のCANH–CANL約60 Ω
- 最大ハーネス長・実配置での波形、error counter、bus-off発生有無
