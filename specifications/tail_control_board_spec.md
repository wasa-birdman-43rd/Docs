# Tail Control Board Specification

更新日: **2026-10-03**

テール操舵基板の**現行仕様**を管理する文書です。

設計判断の理由・比較経緯は `../decisions/electrical_design_decisions.md` を参照してください。

---

## 1. Purpose

本基板は鳥人間機テール部の操舵専用基板である。

主機能:

- CockpitからCANで操舵指令を受信
- Rudder / ElevatorのKRS-5034HVをICSで制御
- ICS実位置を取得
- 操舵指令・実位置・電流等を記録/送信
- Measurement Boardが不在・故障しても操舵を維持する

---

## 2. MCU / Firmware

| Item | Current specification |
|---|---|
| MCU | ESP32-S3-WROOM-1-N16R8 |
| Development | PlatformIO |
| RTOS | FreeRTOS |
| Task priority | CAN RX > Steering + ICS > CAN TX |

基本タスク:

- CAN RX
- Steering + ICS
- CAN TX

---

## 3. Servo / ICS

| Item | Current specification |
|---|---|
| Rudder servo | KRS-5034HV |
| Elevator servo | KRS-5034HV |
| Servo power | 3S LiPo direct |
| Control protocol | Kondo ICS |
| Topology | 2 servos on same SIO bus, separate servo IDs |
| Position feedback | ICS readback |
| Board connector | JST XA 3pin per servo |
| Harness | Star from board to each servo |

### ICS interface

- SN74LV1T125系3-state bufferを基本とする
- MCU side: 3.3 V
- ICS side: 5 V
- ICS_SIG pull-up: 2.2 kΩ to 5V_ICS
- EN_INで送受信方向を制御
- 各ロジックICに0.1 µF decoupling

---

## 4. CAN

| Item | Current specification |
|---|---|
| Bitrate | 500 kbps |
| Physical scope | Cockpit ↔ Tail |
| Tail position | Physical endpoint |
| Termination | 120 Ω, always populated |
| Connector | JST XA 3pin |
| Pin signals | CANH / CANL / GND |
| Harness | AWG24 CANH/CANL twisted pair + GND |

### CAN ID

11-bit standard ID:

```text
[Priority 2bit][Node ID 4bit][Message Type 5bit]
```

Node IDs:

- 0: Reserved
- 1: Cockpit
- 2: Tail
- 3: Measurement
- 4–F: Expansion

### Steering command

- int16_t
- -10000 … +10000 = -100 … +100 %
- Cockpit command rate: approximately 50 Hz
- Sequence: uint16_t
- Timestamp synchronization from Cockpit: approximately 10 Hz

Tail側でnormalized commandへtrim / clamp / asymmetric rangeを適用し、ICS commandへ変換する。

### Failsafe

- Initial CAN timeout: 300 ms
- Timeout後、最後に有効だったtrim positionへ約0.7 sでsmooth return
- CAN復帰後もrate-limited recovery
- Normal controlにはrate limitを掛けない

### Protection

- CAN TVS: 搭載する。具体MPNはTBD
- PESD2CANFD24V-T classは候補として扱い、確定部品とはしない
- CMC: footprint provision only
- Initial assembly: CMC DNI, CANH/CANL each 0 Ω bypass

---

## 5. Main power

Input:

- 3S LiPo
- Full charge: 12.6 V

Main connector:

- JST VH 2pin
- BAT+ / GND

Distribution at battery input:

```text
3S LiPo
   |
 harness-side MAIN SW
   |
 JST VH
   |
   +-- Rudder branch
   +-- Elevator branch
   +-- Control branch
```

Main power switchは基板外ハーネス側に置く。

Servo high-current pathへ制御枝用逆接MOSFET / TVSを直列挿入しない。

---

## 6. Control branch protection

```text
3S
 |
1812L110/33MR PTC
 |
TSM2309CX RFG PMOS reverse protection
 |  Gate pull-down: 10 kΩ
 |  Gate-Source Zener: 15 V
 |
CONTROL_BAT
 |
 +-- SMAJ15A -> GND
 |
 +-- AP63200 -> 3.3V_MAIN
 |
 +-- AP7387-50SA-7 -> 5V_ICS
```

### PTC

**Littelfuse 1812L110/33MR**

- Hold: 1.1 A
- Trip: approximately 1.95 A
- Max voltage: 33 V
- Package: 1812
- Control branch only

### Reverse-polarity PMOS

**Taiwan Semiconductor TSM2309CX RFG**

- P-channel
- VDS: -60 V
- VGS absolute max: ±20 V
- ID: -3.1 A class
- RDS(on): 190 mΩ max @ VGS=-10 V
- Package: SOT-23

Connection:

- Drain: upstream / battery side
- Source: downstream / CONTROL_BAT side
- Gate pull-down: 10 kΩ to GND
- Gate-Source clamp: 15 V Zener, Cathode=Source / Anode=Gate
- Zener family: BZT52C15 class

### TVS

**SMAJ15A**, unidirectional

- VRWM: 15 V
- VBR: approximately 16.7–18.5 V
- VC: approximately 24.4 V class
- Placement: CONTROL_BAT to GND, downstream of PTC and PMOS

### Input bulk

- 220 µF / 25 V
- ceramic bulk/local decoupling in parallel

---

## 7. 3.3V MAIN Buck

Buck:

**AP63200WU-7**

| Parameter | Value |
|---|---|
| Input | CONTROL_BAT |
| Output | 3.3 V |
| Switching frequency | 500 kHz |
| Inductor | 6.8 µH |
| Inductor Isat | ≥2.7 A, preferably ≥3 A |
| Inductor DCR | <100 mΩ target |
| FB upper | 196 kΩ, 1% |
| FB lower | 62 kΩ, 1% |
| Cff | 100 pF |
| Cbst | 100 nF |
| Local Cin | 10 µF ceramic + 0.1 µF |
| Cout | 22 µF ×2 ceramic |
| EN | Autostart |

Inductorはshielded type。具体MPNはTBD。

---

## 8. ICS 5V supply

Adopted LDO:

**Diodes Incorporated AP7387-50SA-7**

- Fixed 5.0 V
- SOT-23
- Vin: 5–60 V class
- Iout: 150 mA class
- Source: CONTROL_BAT

Local capacitors:

- Cin: **1 µF ceramic**
- Cout: **10 µF ceramic, X5R/X7R**

用途はICS interfaceのみ。Servo本体電源には使用しない。

---

## 9. USB-C / USB power

Connector:

**HRO TYPE-C-31-M-12**

USB mode:

- USB 2.0 Device only

CC:

- CC1: 5.1 kΩ to GND
- CC2: 5.1 kΩ to GND

Data:

- D- -> ESP32-S3 GPIO19
- D+ -> ESP32-S3 GPIO20
- USBLC6-2SC6 ESD protection
- 22 Ω series resistor each
- Optional shunt capacitor footprints: initial NC

Power:

```text
USB 5V
  -> AP7361C-33E-13
  -> 3.3V_USB
```

USB only powers logic. Servo power is never sourced from USB.

USB shield connection method is still TBD.

---

## 10. Power MUX

Main:

```text
3S -> AP63200 -> 3.3V_MAIN
```

Backup:

```text
USB 5V -> AP7361C-33E-13 -> 3.3V_USB
```

MUX:

**TPS2116**

Priority:

- 3.3V_MAIN preferred
- 3.3V_USB backup

Output:

**3.3V_LOGIC**

TPS2116 ST outputはESPで取得する方向。GPIO9をcurrent candidateとする。

---

## 11. Current monitoring

INA226 ×3:

| Channel | Address | Shunt |
|---|---:|---:|
| Rudder | 0x40 | 10 mΩ candidate |
| Elevator | 0x41 | 10 mΩ candidate |
| 3.3V Logic | 0x44 | 50 mΩ candidate |

Shunt値は現時点では候補。想定最大電流・分解能・電圧降下・損失を確認して最終確定し、具体MPNを選定する。

### Placement

Servo:

```text
Battery / branch protection
  -> shunt
  -> servo
```

Logic:

```text
3.3V_MAIN --\
             > TPS2116 -> 50mΩ shunt -> 3.3V_LOGIC loads
3.3V_USB  --/
```

Logic INA226 therefore measures actual 3.3V logic current regardless of selected source.

ICS 5V current is not included in the Logic current value.

### Interface

- INA226 VS: 3.3V_LOGIC
- SDA/SCL shared
- SDA pull-up: 4.7 kΩ to 3.3V_LOGIC
- SCL pull-up: 4.7 kΩ to 3.3V_LOGIC
- ALERT: unused in first revision
- Shunt sense routing: Kelvin

---

## 12. Reset / Watchdog

Buttons:

- BOOT
- RESET

External watchdog:

**TPS3820 family**

Current plan:

- WDI -> GPIO15
- WDI-GND 1 kΩ footprint present but **DNP**
- During boot / flashing: WDI remains High-Z
- After application initialization: GPIO15 becomes output and watchdog kicking starts

TPS3820 exact suffix is TBD.

Watchdog kick condition in firmware is TBD. It must not be a meaningless unconditional heartbeat if steering tasks have failed.

---

## 13. GPIO allocation

Current assignment / candidate:

| GPIO | Function | Status |
|---:|---|---|
| 0 | BOOT | Fixed use |
| 4 | CAN_TX | Candidate |
| 5 | CAN_RX | Candidate |
| 6 | ICS_TX | Candidate |
| 7 | ICS_RX | Candidate |
| 8 | ICS_EN | Candidate |
| 9 | TPS2116_ST | Candidate |
| 15 | WDI | WOBC-aligned |
| 16 | I2C_SCL | WOBC-aligned |
| 17 | I2C_SDA | WOBC-aligned |
| 19 | USB_D- | Native USB |
| 20 | USB_D+ | Native USB |
| 41 | ERROR LED | WOBC-aligned |
| 42 | STAT LED | WOBC-aligned |
| 43 | Debug UART TX reserve | Reserved |
| 44 | Debug UART RX reserve | Reserved |

GPIO3 / GPIO45 / GPIO46等のstrap-related pinsは主要機能に使用しない方針。

Final assignmentはschematic freeze前にESP32-S3 strapping / boot / USB / JTAG条件を再確認する。

---

## 14. LEDs

| LED | Color | Drive | Purpose |
|---|---|---|---|
| PWR | Green | 3.3V_LOGIC direct | Logic power present |
| STAT | Green | GPIO42 | Heartbeat / communication activity |
| ERROR | Red | GPIO41 | Error / failsafe indication |

Blue LEDは使用しない。

STATはCAN trafficに応じたactivity表示を含めるが、高頻度通信をそのまま駆動せず、人が視認できるblinkへ整形する。

---

## 15. Dedicated test points

専用TPは最小限とする。

- TP_BAT+
- TP_3V3_LOGIC
- TP_5V_ICS
- TP_GND

以下には専用TPを設けない。

- CANH / CANL
- ICS_SIG
- I2C
- WDI / RESET
- INA226 shunt sense

必要時はコネクタまたは部品ランドから測定する。

本番機では防湿処理を前提とする。

---

## 16. PCB

| Item | Current specification |
|---|---|
| Layer count | 4 layers |
| Copper weight | 1 oz |
| Board thickness | 1.2 mm |
| Mounting holes | M3 ×4, NPTH, GND non-connected |
| Size target | **60 × 60 mm initial target; expand only if layout requires** |

### Layer stack

| Layer | Purpose |
|---|---|
| L1 | Components / Signal |
| L2 | Solid GND |
| L3 | Power + Low-speed signal |
| L4 | Signal + Power |

### Layout priority

1. Servo high-current path
2. Buck hot-loop minimization
3. Continuous GND reference
4. USB D+/D- routing
5. CAN pair routing
6. ESP antenna keepout
7. ESD close to connectors
8. Separate switching node from CAN / ICS / USB
9. Short low-impedance decoupling loops
10. Repairability / connector accessibility

---

## 17. Connectors

| Function | Connector |
|---|---|
| Main 3S input | JST VH 2pin |
| Rudder | JST XA 3pin |
| Elevator | JST XA 3pin |
| CAN | JST XA 3pin |
| Programming / debug | USB Type-C |

External connectors should generally be placed at board edges. Exact edge assignment is layout-driven.

---

## 18. Open hardware items

- Servo branch individual fuse: adoption / rating / part
- AP63200 inductor exact MPN
- Servo harness wire gauge
- Servo power copper width / polygon geometry
- TPS3820 exact suffix
- TPS2116 peripheral constants final check
- USB shield connection
- ESP32-S3 antenna placement / keepout
- GPIO final review
- Optional CAN CMC exact part
- CAN TVS exact MPN
- INA226 shunt final values / exact MPNs
- Main power switch exact part
- JST VH board-header orientation / exact MPN
- JST VH actual current / temperature / voltage-drop validation
- KRS-5034HV operation at full 3S 12.6 V under representative load

---

## 19. Verification checklist

### Before schematic freeze

- Voltage rating of all parts
- Absolute maximum ratings
- Pin assignment / strapping pins
- USB pins
- WDI / reset
- CAN TX/RX
- ICS UART/SIO
- ESP antenna keepout
- Connector pinout
- INA226 power-off condition
- PMOS orientation / body diode
- TVS polarity / clamp path

### First power-up

1. Servo disconnected
2. Current-limited supply
3. CONTROL_BAT
4. 3.3V_MAIN
5. 5V_ICS
6. USB power
7. TPS2116
8. ESP boot
9. TPS3820
10. CAN
11. ICS interface
12. One servo
13. Two servos

### Before flight

- 12.6 V full-charge test
- Two-servo representative high-load test
- Servo connector / VH temperature and voltage-drop test
- CAN long-harness test
- USB + main power simultaneous test
- Watchdog fault injection
- Brownout test
- CAN timeout / failsafe test
- Thermal test
- Continuous operation test
- Moisture protection / coating inspection
