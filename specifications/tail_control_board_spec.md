# Tail Control Board Specification

テール操舵基板の**現行仕様**を管理する文書です。

設計判断の理由・比較経緯は `../decisions/electrical_design_decisions.md` を参照してください。

## PCB

| Item | Current specification |
|---|---|
| Layer count | 4 layers |
| Copper weight | 1 oz |
| Board thickness | 1.2 mm |
| Mounting holes | M3 × 4, 基板四隅 |

## Layer stack

| Layer | Purpose |
|---|---|
| L1 | 部品・信号 |
| L2 | 全面 GND |
| L3 | 電源 + 低速信号 |
| L4 | 信号・電源 |

## Control / Communication

| Item | Current specification |
|---|---|
| Main MCU | ESP32-S3 |
| Communication | CAN |
| CAN bitrate | 500 kbps |
| CAN scope | コックピット–テール |

## Servo power

- BAT+ / GND は十分な配線幅またはポリゴン面積を確保する
- 1 oz銅厚を前提に電圧降下・発熱を評価する

## Under consideration / TBD

以下は現時点でこの仕様書上の確定事項として扱わない。

- CANトランシーバの最終採用品
- 電源回路の最終部品
- コネクタ
- 基板外形寸法
- CAN ID
- サーボ信号・電源コネクタの具体仕様
- 保護回路
