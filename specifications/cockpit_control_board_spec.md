# 鳥人間 コックピット操舵基板 — Initial Hardware / Interface Design

更新日: 2026-10-04  
文書状態: **初版設計・レビュー待ち（Draft v0.1）**  
対象: Cockpit Steering Control Board  
設計段階: 回路図作成へ移行できる仕様案

---

## 0. この文書の扱い

本書は、これまで確定したテール操舵基板の設計思想と共通部品を踏襲して作成した、コックピット操舵基板の初版設計である。

現時点で未確認の項目は、設計を止めずに合理的な仮定を置き、`要レビュー` または `要実測` と明記した。レビュー後に確定仕様へ昇格させる。

本基板は安全上重要な操舵専用基板であり、GPS、IMU、高度計、SD、LoRaなどの計測・記録機能は搭載しない。計測系が停止・短絡・再起動しても、コックピットからテールまでの操舵経路を維持することを最優先とする。

---

# 1. 目的と機能範囲

## 1.1 主機能

- 2軸ジョイスティックをESP32-S3内蔵ADCで取得
- ジョイスティック値を校正・平滑化・正規化
- ラダー／エレベータの飛行中トリムを操作
- 正規化操舵指令をCAN 500 kbpsでテールへ50 Hz送信
- コックピット起動基準の共通時刻を10 Hzで送信
- テール状態をCAN受信し、操縦者・整備者へ異常表示
- 2S LiPoとUSB-Cのいずれでもロジック単体を起動
- 外部WDTでソフトウェア停止時にESP32-S3を再起動
- 電源電圧、ADC断線、CAN状態、タスク生存を監視

## 1.2 搭載しないもの

- KRSサーボ駆動回路
- ICSインターフェース
- GNSS、IMU、高度計、RPM計測
- MicroSD、LoRa、Wi-Fi／Bluetoothを使った本番機能
- 計測系の電源供給

---

# 2. 継承する設計思想

1. 操舵系と計測系は電源・MCU・ソフトウェアを分離する。
2. コックピット操舵基板とテール操舵基板は、計測基板なしで操舵を成立させる。
3. テール基板と可能な限り部品、回路、コネクタ、製造条件を共通化する。
4. メーカー推奨回路を優先し、独自回路を増やさない。
5. 単一故障で最大操舵が継続しないよう、入力異常時はトリム位置へ戻す。
6. 飛行中にフラッシュ／NVSへ頻繁な書込みを行わない。
7. 交換、測定、切り分けがしやすい配置とテストポイントを用意する。

---

# 3. システム構成

```text
専用2S LiPo
    │
    ├─ 入力保護 ─ AP63200 ─ 3.3 V_MAIN ─┐
    │                                     │
USB-C ─ AP7361C-33 ─ 3.3 V_USB ─────────┤ TPS2116
                                          │
                                     3.3 V_LOGIC
                                          │
       ┌──────────────────────────────────┼─────────────────────┐
       │                                  │                     │
ESP32-S3-WROOM-1-N16R8              TCAN3413               TPS3820
       │                                  │                     │
       ├─ Joystick ADC ×2                 └─ CANH/CANL/GND     └─ CHIP_PU
       ├─ Battery ADC                          │
       ├─ Trim encoders ×2                     ├─ Tail Steering Board
       ├─ Optional I2C display                 └─ Measurement Board（受信のみでも可）
       └─ Status LEDs
```

CANは物理的には多対多バスだが、操舵の主経路は `Cockpit → Tail` とする。計測基板は同じバスを受信してもよいが、計測基板の存在を操舵成立条件にしない。

---

# 4. MCU

## 4.1 採用品

**ESP32-S3-WROOM-1-N16R8**

テール操舵基板と同一型番とし、部品在庫、書込み手順、PlatformIO設定、予備基板を共通化する。

N16R8ではOctal PSRAMにGPIO33〜GPIO37を使用するため、これらを外部信号へ割り当てない。

## 4.2 開発環境

- PlatformIO
- Arduino frameworkまたはESP-IDFラッパーを使用可能
- CANはESP32-S3内蔵TWAIを使用
- 本番ではWi-Fi／Bluetoothを停止

## 4.3 GPIO割当案

| GPIO | 機能 | 備考 |
|---:|---|---|
| 0 | BOOT | 10 kΩ pull-up、BOOTボタンでGND |
| 4 | ADC1 Rudder | ジョイスティック・ラダー軸 |
| 5 | ADC1 Elevator | ジョイスティック・エレベータ軸 |
| 6 | ADC1 Battery Sense | 保護後2S電圧の監視 |
| 8 | I2C SDA | オプション表示器 |
| 9 | I2C SCL | オプション表示器 |
| 10 | Joystick Switch | オプション、未使用時pull-up |
| 11 | Rudder Encoder A | トリム入力 |
| 12 | Rudder Encoder B | トリム入力 |
| 13 | Rudder Encoder SW | 長押しでラダートリム初期値へ |
| 14 | Elevator Encoder A | トリム入力 |
| 15 | Elevator Encoder B | トリム入力 |
| 16 | Elevator Encoder SW | 長押しでエレベータトリム初期値へ |
| 17 | TWAI TX | TCAN3413 TXD |
| 18 | TWAI RX | TCAN3413 RXD |
| 19 | USB D− | ESP32-S3 Native USB固定 |
| 20 | USB D+ | ESP32-S3 Native USB固定 |
| 21 | External WDT WDI | TPS3820-33 |
| 38 | CAN LED | 受送信表示、低優先度 |
| 39 | Fault LED | 異常表示 |
| 40 | TCAN STB | 10 kΩ pull-downで通常時Normal |
| 41 | Power-good input | TPS2116状態監視、回路確定時に確認 |
| 42 | Spare GPIO | テストパッド |
| 43 | UART0 TX | デバッグヘッダ、499 Ω直列候補 |
| 44 | UART0 RX | デバッグヘッダ |
| 47 | Spare GPIO | テストパッド |
| 48 | Heartbeat LED | 動作確認用 |

GPIO3、45、46はストラップ影響を避けて通常機能に使用しない。GPIO33〜37はN16R8で使用しない。

---

# 5. 主電源

## 5.1 入力

- 電池: **操舵専用2S LiPo**
- 通常入力範囲: **約6.0〜8.4 V**
- コネクタ: **JST VH 2極**
- Pin 1: BAT+
- Pin 2: GND

テールの3S電源とは別バッテリーとする。CAN GND線は信号基準として接続するが、CANケーブルから相手基板へ電力供給しない。

## 5.2 入力保護案

```text
JST VH BAT+
  → PTC
  → P-channel MOSFET逆接保護
  → TVS
  → 220 µF / 25 V
  → 10〜22 µF ceramic
  → AP63200 VIN
```

| 部品 | 初版案 | 状態 |
|---|---|---|
| PTC | Littelfuse 1812L110/33MR | テールと共通、採用 |
| 逆接PMOS | DMP3010LK3-13相当、30 V品 | 暫定・在庫確認 |
| 入力TVS | SMBJ10A相当 | 暫定・波形確認 |
| バルク | 220 µF / 25 V、低ESR | 実機で100〜470 µF調整可 |

AO3401は過去に半開・発煙事例があるため、本基板の初版候補から外す。

## 5.3 3.3 V Buck

- IC: **AP63200WU-7**
- 入力: 保護後2S
- 出力: 3.3 V_MAIN
- スイッチング周波数: 500 kHz
- 最大出力: 2 A
- インダクタ: 6.8 µH、Isat 3 A以上推奨、DCR 100 mΩ以下、シールド型
- FB上側: 312 kΩ 1%
- FB下側: 100 kΩ 1%
- 理論出力: 約3.30 V（VREF 0.8 V前提）
- CFF: フットプリント確保、初期DNP
- 出力: 22 µF ×2を基本

AP63200のSWノード、入力コンデンサ、インダクタ、出力コンデンサはメーカー推奨配置に従い、ホットループを最小化する。

## 5.4 USB給電とPower MUX

```text
USB VBUS 5 V
  → AP7361C-33E-13
  → 3.3 V_USB

3.3 V_MAIN ─→ TPS2116 IN1（優先）
3.3 V_USB  ─→ TPS2116 IN2（バックアップ）
TPS2116 OUT → 3.3 V_LOGIC
```

- USBだけでもMCU、CANロジック、ADC、UIを動作可能
- USBから2S LiPoへ逆流させない
- 2S接続中は3.3 V_MAINを優先
- TPS2116の逆電流阻止を利用
- USB接続中でも外部電源の着脱でESPが不用意にリセットしないことを実測する

## 5.5 電池電圧監視

- 監視点: 逆接保護・PTC後の2Sノード
- 分圧: 100 kΩ / 27 kΩ、1%
- ADC側: 100 nF to GND
- GPIO: GPIO6 / ADC1
- 8.4 V入力時のADC電圧: 約1.79 V
- ソフトウェアで個体校正係数を保持

低電圧警告値は使用する2S LiPo容量・許容終止電圧と実負荷試験後に確定する。警告のみで直ちに操舵を停止しない。

---

# 6. ジョイスティック入力

## 6.1 前提

42代と同じ2軸ジョイスティックを使用する。ただし、現時点では型番・全抵抗値・ピン配列を未確認のため、以下を初版仮定とする。

- 2軸可変抵抗式
- 両軸で3.3 VとGNDを共通使用
- 各軸にワイパ出力1本
- ESP32-S3 ADC1で測定可能

能動出力式または5 V専用品だった場合は、コネクタ以降の入力段だけを変更する。

## 6.2 コネクタ

**JST XA 5極**

| Pin | Signal | 備考 |
|---:|---|---|
| 1 | 3V3_JOY | アナログ用フィルタ後電源 |
| 2 | GND_JOY | 基板GND |
| 3 | RUDDER_WIPER | ラダー軸 |
| 4 | ELEVATOR_WIPER | エレベータ軸 |
| 5 | JOYSTICK_SW | オプション、未使用可 |

最終ピン番号は現物ハーネスとの誤接続防止レビュー後に凍結する。

## 6.3 アナログ電源

```text
3.3 V_LOGIC
 → Ferrite beadまたは0 Ω選択
 → 10 µF + 1 µF + 0.1 µF
 → 470 Ω（実装選択、0 Ωへ変更可）
 → 3V3_JOY

GND_JOY
 → 470 Ω（実装選択、0 Ωへ変更可）
 → GND
```

両端の470 Ωは、可変抵抗の端点がADC電源レールへ完全一致しないようにし、断線・短絡検出用の余白を作る目的で置く。ジョイスティック抵抗値を実測後、0 Ω／220 Ω／470 Ωから決定する。

## 6.4 各軸入力回路

```text
WIPER connector
  → 1 kΩ series
  → ADC pin
       ├─ 100 nF → GND
       ├─ 470 kΩ → 3.3 V_LOGIC（断線時Highへ誘導）
       └─ low-leakage clamp / ESD footprint → 3.3 V / GND
```

- Rudder: GPIO4 / ADC1
- Elevator: GPIO5 / ADC1
- RC値はESP32-S3推奨の0.1 µFを基準とする
- 低リークのクランプ部品を選び、ADC誤差を実測する
- ADCノードとコネクタの両方にテストポイントを設ける
- ADC配線はSWノード、USB、CANから離す
- ADC下には連続GND面を確保する

## 6.5 信号処理

```text
ADC raw
 → 1 kHz sampling
 → median / spike rejection
 → IIR low-pass
 → min / center / max calibration
 → -10000〜+10000へ正規化
 → dead zone
 → expo / gain（初期は線形）
 → range clamp
 → CAN command 50 Hz
```

初期値:

- Dead zone: 中立の±2%
- Expo: 0（線形）
- 出力範囲: `int16_t -10000〜+10000`
- 校正値: 整備モードでのみNVSへ保存
- 飛行中のNVS書込み: なし

## 6.6 入力異常判定

以下を異常候補とし、100 ms以上継続した場合に確定異常とする。

- 校正済み正常範囲を超える電圧
- 断線バイアス側へ張り付き
- GND短絡相当の張り付き
- 物理的に不可能な急変
- 起動時に中立付近へ一度も入らない
- 両軸が同時に電源レール側へ移動

異常時:

1. Fault LED点灯
2. CAN状態フレームへ入力異常を設定
3. 該当軸のstick指令を0へ移行
4. テール側は最後に有効だったトリム位置へレート制限付きで戻す

故障したADC値を最大操舵として送信し続けない。

---

# 7. トリム操作系

## 7.1 操作器

ラダー、エレベータそれぞれに、押しボタン付き機械式ロータリーエンコーダを1個使用する。

理由:

- スイッチ固着でトリムが連続増加しにくい
- クリック単位で変更量を制限できる
- 押しボタン長押しで初期値へ戻せる
- 飛行中にPCを必要としない

## 7.2 UIコネクタ

**JST XA 8極**

| Pin | Signal |
|---:|---|
| 1 | 3.3 V_UI |
| 2 | GND |
| 3 | RUD_TRIM_A |
| 4 | RUD_TRIM_B |
| 5 | RUD_TRIM_SW |
| 6 | ELE_TRIM_A |
| 7 | ELE_TRIM_B |
| 8 | ELE_TRIM_SW |

各入力:

- 10 kΩ pull-up to 3.3 V
- 1 kΩ series
- 10 nF to GND用フットプリント（初期実装、実機で調整）
- ソフトウェアデバウンス

## 7.3 トリム仕様初期値

- 1 detent: 25 count = 0.25%
- 操作範囲: ±2000 count = ±20%
- 回転速度による加速: 初版では使用しない
- 押しボタン1秒長押し: その軸をデフォルトトリムへ復帰
- 短押し: 何もしない
- 現在値: RAM保持
- 再起動時: `config.hpp` のデフォルトトリム
- 飛行中のNVS保存: なし

## 7.4 表示

基板上:

- POWER LED
- CAN activity LED
- FAULT LED
- HEARTBEAT LED

オプション:

- JST SHまたはXA 4極 I2C表示器コネクタ
- 3.3 V / GND / SDA / SCL
- OLED故障・未接続でも操舵タスクを止めない
- I2Cアクセスはタイムアウト付き、最低優先度タスク

表示器は操舵成立条件にしない。

---

# 8. CAN物理層

## 8.1 トランシーバ

**TCAN3413DR**（SOIC-8を優先）

- VCC: 3.3 V_LOGIC
- VIO: 3.3 V_LOGIC
- TXD: GPIO17
- RXD: GPIO18
- STB: GPIO40、10 kΩでGNDへpull-down
- 0.1 µF + 数µFをIC直近へ配置

テール基板と同じ非絶縁構成とし、CANH/CANL/GNDを接続する。専用2Sとテール3Sは独立電源のまま、CAN GNDを信号基準として共有する。

## 8.2 コネクタ

**JST XA 3極**

| Pin | Signal |
|---:|---|
| 1 | CANH |
| 2 | CANL |
| 3 | CAN_GND |

テール基板とピン番号を完全統一する。

## 8.3 終端・保護

- 120 Ω終端を基板上に搭載
- 0 ΩリンクまたはソルダージャンパでEnable/Disable
- コックピットとテールをバス両端として、両方の終端を有効
- 電源OFF状態でCANH-CANL間を測り、約60 Ωを確認
- CAN用TVS: PESD2CANFD24V相当を初版候補
- CMC: フットプリントのみ、初期0 Ω／DNP
- TVSとCMCはコネクタ直近

## 8.4 ハーネス

- CANH/CANL: AWG24ツイストペア
- GND: AWG24、ツイストペアに沿わせる
- 電源線はCANコネクタへ含めない
- コックピット〜テール間で途中スタブを短くする

---

# 9. CANメッセージ初版

標準11 bit ID:

```text
CAN_ID = (Priority << 9) | (NodeID << 5) | MessageType
```

Cockpit Node ID = `0x1`。

## 9.1 ID案

| CAN ID | Priority | Type | 周期 | 内容 |
|---:|---:|---:|---:|---|
| 0x020 | 0 | 0x00 | 50 Hz | Steering Stick Command |
| 0x021 | 0 | 0x01 | 10 Hz + change | Trim Command |
| 0x022 | 0 | 0x02 | event + 10 Hz | Cockpit Fault / Validity |
| 0x220 | 1 | 0x00 | 10 Hz | Time Sync |
| 0x221 | 1 | 0x01 | 10 Hz | Cockpit Status / Battery |
| 0x222 | 1 | 0x02 | 10 Hz | Heartbeat |

IDは共有CAN定義ファイル `firmware/common/protocol/can_messages.hpp` にのみ定義し、各ノードへ数値を重複記述しない。

## 9.2 Steering Stick Command（8 byte）

| Byte | 型 | 内容 |
|---:|---|---|
| 0–1 | int16_t | Rudder stick `-10000〜+10000` |
| 2–3 | int16_t | Elevator stick `-10000〜+10000` |
| 4–5 | uint16_t | sequence |
| 6–7 | uint16_t | validity / mode flags |

## 9.3 Trim Command（8 byte）

| Byte | 型 | 内容 |
|---:|---|---|
| 0–1 | int16_t | Rudder trim |
| 2–3 | int16_t | Elevator trim |
| 4–5 | uint16_t | trim sequence |
| 6–7 | uint16_t | flags / reserved |

トリムはテール側でstickへ加算する。ICS生値はCANへ出さない。

## 9.4 Time Sync（8 byte以内）

- `uint32_t timestamp_ms`
- `uint16_t sequence`
- flags / reserved
- コックピットESP起動時を0 ms
- 10 Hz送信

---

# 10. USB-C

テール基板と同一回路・同一コネクタを使用する。

- Connector: **HRO TYPE-C-31-M-12**
- USB 2.0 Device only
- CC1: 5.1 kΩ to GND
- CC2: 5.1 kΩ to GND
- D−: GPIO19
- D+: GPIO20
- D+/D−: 各22 Ω直列、33 Ωへ交換可能
- 対GND capacitor: フットプリントのみ、初期DNP
- ESD: **USBLC6-2SC6**
- シールド: 0 Ω／RC／直接GNDを選択可能なフットプリントとし、初版は0 Ω接続案

D+/D−は短く、同層・連続GND参照・等長を意識し、Buck SWノードとCANから離す。

---

# 11. Reset / 外部Watchdog

## 11.1 採用品

**TPS3820-33DBVR**

- 3.3 V監視しきい値: 公称2.93 V
- Reset出力: active-low push-pull
- Watchdog timeout: 約0.2 s
- Manual Reset対応

## 11.2 接続

```text
GPIO21 → WDI
TPS3820 /RESET → ESP32-S3 CHIP_PU
RESET button → TPS3820 /MR
BOOT button → GPIO0 to GND
```

WDIを単純な周期タイマだけで反転させない。以下の全条件が直近100 ms以内に成立した場合のみSafety SupervisorがWDIを更新する。

- ADC sampling taskが更新
- Control taskが更新
- CAN TX taskが更新
- main loopの共有データ整合性が正常

UI、OLED、CAN status受信の停止だけではWDIを止めない。

---

# 12. ソフトウェアタスク

## 12.1 タスク案

| Task | 周期 | 優先度 | 主処理 |
|---|---:|---:|---|
| Safety Supervisor | 50 ms | 最高 | 各タスク生存確認、WDI更新 |
| ADC / Input | 1 ms | 高 | ADC取得、フィルタ、異常候補検出 |
| Control | 10 ms | 高 | 校正、正規化、dead zone、trim統合状態 |
| CAN TX | 20 ms | 高 | 操舵指令50 Hz、他周期送信 |
| CAN RX | event | 中 | Tail status受信、最新値mailbox更新 |
| Trim Input | 5 ms | 中 | Encoder decode、debounce、範囲制限 |
| UI / LED | 50〜100 ms | 低 | 表示更新、LEDパターン |
| Diagnostics | 1 s | 最低 | カウンタ、電圧、エラー集計 |

## 12.2 データ受渡し

- 操舵値は長さ1のmailboxまたはatomic snapshotで最新値のみ保持
- 古い操舵指令をQueueへ蓄積しない
- CAN送信に失敗しても制御タスクをblockさせない
- UIタスクから制御用データを直接書き換えない
- 設定値更新時は範囲検査後に一括反映

---

# 13. フェイルセーフ

## 13.1 コックピット内部異常

| 異常 | 動作 |
|---|---|
| 片軸ADC異常 | 該当stickを0、fault flag送信 |
| 両軸ADC異常 | 両stickを0、fault flag送信 |
| Encoder異常 | 最後の有効trimを保持 |
| OLED/I2C異常 | 表示を停止、操舵継続 |
| Battery low | 警告、操舵継続 |
| Control/CAN task停止 | WDTを更新せずESP再起動 |
| TCAN bus-off | 復旧試行、状態記録、一定時間で再初期化 |

## 13.2 Tailとの整合

- Cockpit steering command timeout: Tail側300 ms初期値
- Tailは短い欠落で最後の指令を保持
- timeout後は最後の有効trim位置へ約0.7 sで復帰
- Cockpit入力異常flag受信時も、該当stick=0としてtrim位置へ移行
- CAN復帰時はTail側で最新指令へレート制限付き復帰

---

# 14. コネクタ一覧

| Ref | 用途 | 型式 | 極数 |
|---|---|---|---:|
| J1 | 2S LiPo | JST VH | 2 |
| J2 | CAN | JST XA | 3 |
| J3 | Joystick | JST XA | 5 |
| J4 | Trim encoders | JST XA | 8 |
| J5 | Optional I2C display | JST SHまたはXA | 4 |
| J6 | UART debug | 2.54 mm header / Tag-Connect候補 | 4〜6 |
| J7 | USB | HRO TYPE-C-31-M-12 | USB-C |

すべての外部コネクタはシルクでPin 1、信号名、電源方向を明記する。J1とJ2は外形・色・位置を離し、誤挿入しにくくする。

---

# 15. テストポイント

最低限、以下を設ける。

- BAT_PROTECTED
- 3V3_MAIN
- 3V3_USB
- 3V3_LOGIC
- GND ×3以上
- RUDDER_WIPER connector side
- RUDDER_ADC MCU side
- ELEVATOR_WIPER connector side
- ELEVATOR_ADC MCU side
- BAT_ADC
- CAN_TXD / CAN_RXD
- CANH / CANL
- CHIP_PU
- WDI
- GPIO0
- UART0 TX/RX

テストポイントはプローブを当てたまま隣接点を短絡しにくい間隔を確保する。

---

# 16. PCB仕様

- 4層
- 1 oz
- 1.2 mm厚
- 初版外形目標: **80 mm × 60 mm以内**
- M3 NPTH取付穴 ×4
- 取付穴GND非接続
- 本番用3Dプリントケースは耐熱材料を使用

基本スタック:

- L1: 部品・信号
- L2: Solid GND
- L3: 電源 + 低速信号
- L4: 信号・電源

## 16.1 配置優先順位

1. ESP32-S3アンテナを基板端に置き、全層keepoutを確保
2. AP63200ホットループを最小化
3. USB-CとESDを近接
4. CANコネクタ、TVS、CMC、TCAN3413を直線的に配置
5. ADC入力部をBuck・USB・CANから離す
6. L2 GNDを分断しない
7. 外部コネクタを基板外周へ並べ、抜差し可能にする
8. BOOT/RESETとLEDをケース開口から確認可能にする
9. テストポイントと部品リファレンスを読める向きに置く

アナログGNDを別ネットに分断せず、連続GND面を使う。アナログ入力の戻り電流経路を短くし、Buckの大電流ループがADC周辺を通らない配置でノイズを抑える。

---

# 17. 主要BOM

| 機能 | 部品 |
|---|---|
| MCU | ESP32-S3-WROOM-1-N16R8 |
| Main Buck | AP63200WU-7 |
| USB LDO | AP7361C-33E-13 |
| Power MUX | TPS2116 |
| CAN | TCAN3413DR |
| External WDT | TPS3820-33DBVR |
| USB ESD | USBLC6-2SC6 |
| USB connector | HRO TYPE-C-31-M-12 |
| Control PTC | Littelfuse 1812L110/33MR |
| Main input connector | JST VH 2極 |
| Signal connectors | JST XA series |

部品選定はJLCPCB/LCSC在庫だけに依存せず、国内入手または代替実装が可能なフットプリントを優先する。

---

# 18. 回路図シート構成案

KiCad回路図は以下の階層に分ける。

1. `00_Top`
2. `01_Power_2S_USB_Mux`
3. `02_ESP32S3_Reset_WDT`
4. `03_Joystick_ADC`
5. `04_Trim_UI`
6. `05_CAN`
7. `06_USB_Debug`
8. `07_Connectors_Testpoints`

ネット名、部品リファレンス、DNP部品をシート間で統一し、回路図上に電圧・信号方向・初期実装状態を記載する。

---

# 19. 初回試作の通電・検証順序

## 19.1 電源のみ

1. ESP、TCAN、WDTを未実装または負荷切離しで2S入力確認
2. 逆接保護の向きと電圧降下確認
3. 3.3 V_MAIN無負荷確認
4. 電流制限電源で6.0 V、7.4 V、8.4 V入力確認
5. USB 5 Vから3.3 V_USB確認
6. TPS2116の優先・切替・逆流確認
7. 2SとUSB同時接続、抜差し波形確認

## 19.2 MCU・USB

1. CHIP_PU、GPIO0電圧確認
2. USB書込み
3. BOOT/RESET動作
4. TPS3820 undervoltage reset
5. WDI停止による再起動
6. 連続動作時3.3 Vリップル・温度

## 19.3 ADC

1. 固定抵抗または可変抵抗で0〜3.3 V範囲確認
2. 42代ジョイスティックの全抵抗値・端子確認
3. 実可動端のmin/center/max記録
4. 1 kHzサンプリング時ノイズ測定
5. USB給電時と2S給電時を比較
6. Buck近傍、CAN通信中のADCノイズ確認
7. WIPER断線、電源断線、GND断線、短絡試験
8. 起動時に操縦桿を倒した場合の挙動確認

## 19.4 CAN

1. TCAN3413モジュール試験と同じ500 kbps設定
2. Cockpit–Tail直結
3. 両端終端で約60 Ω確認
4. 操舵指令50 Hz、time sync 10 Hz
5. sequence欠落・重複・rollover
6. 長ハーネス、実機配索、サーボ高負荷時
7. GND線断、CANH/L断、短絡、片側電源OFF
8. bus-off復帰
9. 300 ms timeoutと0.7 s fail-safe復帰

## 19.5 UI・フェイルセーフ

1. Encoder正逆転、チャタリング、早回し
2. Encoder A/B/SWの各断線・GND短絡
3. 1秒長押しリセットの誤操作性
4. I2C display切離し・SDA/SCL固着
5. ADC task、CAN task、UI taskを個別停止
6. WDTが必要な停止だけを検出することを確認

---

# 20. レビューで優先して確認する項目

1. 42代ジョイスティックの型番、全抵抗値、ピン配列
2. ジョイスティックのラダー／エレベータ軸方向と極性
3. JST XA 5極で既存ハーネスへ対応できるか
4. トリム操作器をロータリーエンコーダ2個とするか
5. オプションOLEDが必要か、LEDのみでよいか
6. 専用2S LiPoの容量、コネクタ、運用終止電圧
7. 80 mm × 60 mmの基板外形と取付穴位置
8. 逆接PMOS、2S TVS、CAN TVSの在庫・実装性
9. CAN message typeとpayload
10. ADC異常時に該当軸を0へ戻す方針
11. TPS3820の0.2 s watchdog timeoutでよいか
12. Measurement Nodeを同じCANへ接続する最終トポロジ

---

# 21. 参照資料

- Espressif, ESP32-S3 Hardware Design Guidelines  
  <https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/>
- Espressif, ESP32-S3-WROOM-1 Datasheet  
  <https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.html>
- Diodes Incorporated, AP63200  
  <https://www.diodes.com/part/view/AP63200>
- Texas Instruments, TCAN3413  
  <https://www.ti.com/product/TCAN3413>
- Texas Instruments, TPS2116  
  <https://www.ti.com/product/TPS2116>
- Texas Instruments, TPS3820  
  <https://www.ti.com/product/TPS3820>
- 42代電装引き継ぎ資料  
  <https://rsk1910.github.io/denso_42handover/>

---

# 22. 初版結論

コックピット操舵基板は、テール操舵基板と共通のESP32-S3、TCAN3413、USB-C、3.3 V電源、Power MUX、外部WDTを採用し、専用2S LiPoで独立動作させる。

ジョイスティックはESP32-S3 ADC1へ保護・RC・断線検出付きで入力し、飛行中トリムは2個のロータリーエンコーダで変更する。操舵指令は正規化値として50 HzでCAN送信し、計測・表示機能の故障を操舵停止条件にしない。

初版回路図へ進む前の最大の確認点は、42代ジョイスティックの実物電気特性とコネクタ配列である。それ以外の主要ブロックは、この仕様のまま回路図化可能である。
