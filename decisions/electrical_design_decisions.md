# Electrical Design Decisions

電装設計で「なぜそう決めたか」を残すための設計判断ログです。

現行仕様そのものは `specifications/` を正本とします。

---

## 2026-09-30 — テール操舵基板を4層基板とする

### Decision

テール操舵基板は4層基板で設計する。

基本スタックアップ:

- L1: 部品・信号
- L2: 全面 GND
- L3: 電源 + 低速信号
- L4: 信号・電源

### Reason

GNDプレーンを確保し、CAN・MCU・電源・サーボ系が同居する基板でリターンパスとノイズ耐性を確保しやすくするため。

---

## 2026-09-30 — 銅厚を1 ozとする

### Decision

テール操舵基板の銅厚は 1 oz とする。

### Reason

2 oz は基板コスト増が大きいため採用しない。サーボ電源の BAT+ / GND は、1 ozでも十分な配線幅・ポリゴン面積を確保し、電圧降下と発熱を抑える。

---

## 2026-09-30 — 基板厚を1.2 mmとする

### Decision

テール操舵基板の基板厚は 1.2 mm とする。

---

## 2026-09-30 — 取付穴を四隅4か所とする

### Decision

テール操舵基板は四隅に M3 用 NPTH 取付穴を4つ設ける。取付穴はGNDへ接続しない。

---

# 2026-10-01〜2026-10-02 — テール操舵基板 詳細設計

## MCU / Firmware

### Decision

- MCU: ESP32-S3-WROOM-1-N16R8
- 開発環境: PlatformIO
- FreeRTOSタスクを CAN RX / Steering + ICS / CAN TX に分離する
- 優先度は CAN RX > Steering + ICS > CAN TX を基本とする

### Reason

操舵通信とサーボ処理を計測処理から分離し、操舵を最優先で生存させるため。

---

## Servo / ICS

### Decision

- Rudder / Elevator とも KRS-5034HV
- 2台を同一ICS SIOバスへ接続し、サーボIDで区別する
- 実位置をICSから取得する
- サーボ電源は3S LiPoを直接使用する
- サーボ側はKRS純正コネクタ、基板側は JST XA 3pin
- 基板から2サーボへスター配線する
- サーボハーネスは VCC / GND / SIO の3本すべて **AWG20** で統一する
- Rudder / Elevator各枝に個別ヒューズは搭載しない
- ICS 3.3V/5V変換は近藤科学の3.3V対応回路を基本とし、SN74LV1T125系を使用する
- ICS_SIGは5Vへ2.2 kΩでプルアップする

### Reason

既存KRS資産との互換性と、実績のあるICS回路を優先する。独自簡略回路による5V信号のESP32への流入を避ける。

---

## CAN

### Decision

- Cockpit ↔ Tail は CAN 500 kbps
- Tailは物理終端ノードとし、120 Ω終端を常時実装する
- Tail側CANコネクタは JST XA 3pin（CANH / CANL / GND）
- CANH/CANLはAWG24ツイストペアを基本とする
- CAN TVSは **Nexperia PESD2CANFD24V-T** を使用する
- CAN CMCはフットプリントのみ用意し、初期実装はDNI、0 Ω×2でバイパスする

### Reason

500 kbpsで必要十分な帯域を確保しつつ、終端条件を固定して現場での設定ミスを減らす。CMCはEMI改善余地を残しつつ、初版で不要な直列故障点を増やさない。

---

## Main power distribution

### Decision

3S LiPo入力点から Rudder / Elevator / Control branch へスター分配する。

主電源スイッチは基板上に直列配置せず、3S LiPo〜基板間のハーネス側に置く。

サーボ大電流経路には制御枝用の逆接MOSFETやTVSを直列に入れない。

### Reason

操舵2系統共通の半導体故障点を増やさないため。主電源スイッチもハーネス側に置き、基板の大電流経路を単純化する。

---

## Control branch protection

### Decision

制御枝は以下の順を基本とする。

```text
3S LiPo
  -> 1812L110/33MR PTC
  -> TSM2309CX RFG P-channel MOSFET reverse-polarity protection
  -> CONTROL_BAT
       -> SMAJ15A TVS to GND
       -> AP63200 -> 3.3V_MAIN
       -> AP7387-50SA-7 -> 5V_ICS
```

- PTC: Littelfuse 1812L110/33MR
- Reverse-polarity PMOS: TSM2309CX RFG
- PMOS gate pull-down: 10 kΩ
- PMOS gate-source clamp: 15 V Zener（BZT52C15系、Cathode=Source / Anode=Gate）
- Control-bus TVS: SMAJ15A, unidirectional
- Control input bulk: 220 µF / 25 V + local ceramics

### Reason

PTCをTVSより上流に置き、TVS短絡故障時に制御枝だけを切り離せるようにする。PMOSは制御枝だけへ入れ、操舵2軸の共通故障点にしない。

---

## 3.3V logic supply

### Decision

- Buck: AP63200WU-7
- Vout: 3.3 V
- Switching frequency: 500 kHz
- Inductor: **TDK SPM6530T-6R8M**（6.8 µH, shielded, 4 A級, DCR約53 mΩ）
- FB: 196 kΩ / 62 kΩ, 1%
- Cff: 100 pF
- Cbst: 100 nF
- IC local Cin: 10 µF ceramic + 0.1 µF
- Cout: 22 µF ×2 ceramic
- ENは基本的に自動起動とする

---

## ICS 5V supply

### Decision

TPS7A2450から高耐圧LDOへ変更し、Diodes Incorporated AP7387-50SA-7 を採用する。

- Output: 5.0 V
- Input voltage class: 60 V
- Output current class: 150 mA
- Package: SOT-23
- Cin: 1 µF ceramic
- Cout: 10 µF ceramic, X5R/X7R
- Source: CONTROL_BAT

### Reason

TPS7A2450の入力耐圧ではTVSクランプとのマージンが小さい。高耐圧LDOへ変更して、3Sの過渡保護設計を単純化する。

---

## USB / Power MUX

### Decision

- USB Type-C: HRO TYPE-C-31-M-12
- USB 2.0 Device only
- CC1 / CC2: 各5.1 kΩ to GND
- USB ESD: USBLC6-2SC6
- D+ / D-: 各22 Ω series
- USB給電はロジックのみ。サーボへは供給しない
- USB 5V -> AP7361C-33E-13 -> 3.3V_USB
- Main: AP63200 -> 3.3V_MAIN
- TPS2116で 3.3V_MAIN を優先、3.3V_USB をバックアップ
- MUX出力を 3.3V_LOGIC とする
- USB Type-C shell / shieldは **330 Ω ∥ 0.1 µF** でPCB GNDへ接続する

---

## INA226 current monitoring

### Decision

INA226を3個搭載し、Rudder / Elevator / 3.3V Logicを個別に計測する。

| Measurement | I2C address | Shunt |
|---|---:|---:|
| Rudder | 0x40 | 10 mΩ |
| Elevator | 0x41 | 10 mΩ |
| 3.3V Logic | 0x44 | 50 mΩ |

- INA226 ×3、I2C address、配置方針、Shunt値は確定
- Shunt抵抗の具体MPNは設計上固定しない。必要な抵抗値・定格・精度・サイズを満たすものを実装時に選定する
- Logic shuntはTPS2116の後段、3.3V_LOGIC負荷の手前に配置する
- INA226 supply: 3.3V_LOGIC
- I2C pull-up: SDA/SCL各4.7 kΩ to 3.3V_LOGIC
- ALERTは初版では使用しない
- シャントセンスはKelvin配線とする

### Reason

既存のINA226資産を再利用し、Rudder / Elevator / Logicの電流を時系列で記録できるようにする。Shunt値は 10 mΩ / 10 mΩ / 50 mΩ とする。

---

## Reset / Watchdog

### Decision

- 外付けWDT: **TPS3820-33DBVR**
- WDI: ESP32-S3 GPIO15（WOBCハードウェアを踏襲）
- WDI-GND 1 kΩはフットプリントのみ用意し、初期実装はDNP
- 起動中はWDIをHigh-Zとし、アプリケーション初期化後にGPIO出力化してWDT監視を開始する

### Reason

WDIを常時1 kΩでGNDへ落とすと、初回書込みやブートローダ動作中にもWDT制約が掛かり、書込み・起動を妨げる可能性があるため。起動後の実行中ハングを主な監視対象とする。

WDIをどのタスク健全性条件でkickするかはファームウェア設計で確定する。

---

## LED

### Decision

- PWR: Green, GPIOなし、3.3V_LOGIC表示
- STAT: Green, GPIO42
- ERROR: Red, GPIO41
- Blue LEDは使用しない
- STATはheartbeat / CAN activity等へ使用する
- ERRORはCAN timeout / failsafe / ICS異常等のエラー表示へ使用する

### Reason

WOBCのSTAT / ERROR構成を踏襲して診断性を確保する。青色LEDは暗所で眩しくなりやすいため採用しない。

---

## Dedicated test points

### Decision

専用テストポイントは以下の4点だけとする。

- TP_BAT+
- TP_3V3_LOGIC
- TP_5V_ICS
- TP_GND

CAN / ICS / I2C / WDI / shunt等は専用TPを設けず、コネクタまたは部品ランドから必要時に測定する。

### Reason

防湿・防滴を優先し、露出導体とコーティング処理箇所を増やさないため。最低限の電源診断能力のみ残す。

---

## 2026-10-03 — 整合性確認・追加決定

### Decision

- Main 3S input connectorは JST VH 2pin（BAT+ / GND）
- PCB外形は **60 × 60 mmを初期目標**とし、部品配置・大電流配線・放熱・コネクタアクセスに不足があれば必要方向へ拡張する
- Control-bus TVSは **SMAJ15A**
- CAN TVSは **Nexperia PESD2CANFD24V-T**
- AP63200用インダクタは **TDK SPM6530T-6R8M**
- INA226 Shunt値は Rudder 10 mΩ / Elevator 10 mΩ / 3.3V Logic 50 mΩで確定。具体MPNは固定しない
- 外部WDTは **TPS3820-33DBVR**
- Rudder / Elevator各サーボ枝に個別ヒューズは搭載しない
- サーボハーネスは VCC / GND / SIO の3本すべて **AWG20**
- USB Type-C shell / shieldは **330 Ω ∥ 0.1 µF** でPCB GNDへ接続する

### Reason

会話上の最新決定とGitHub文書の差分を解消し、確定事項と未決事項を明確に分離するため。

---

## 現時点の未決定事項

- 1 ozでのサーボ電源ポリゴン幅
- WDI kickのソフトウェア健全性条件
- ESP32-S3 antenna placement / keepoutの最終配置
- GPIO割当の最終ERC/strap確認
- Optional CAN CMCの具体MPN
- TPS2116周辺定数の最終確認
- Main power switchの具体型式
- JST VH基板側ヘッダの向き・具体MPN
- JST VH入力コネクタの実負荷温度・電圧降下検証
- KRS-5034HVの3S満充電12.6 V時の代表負荷試験
