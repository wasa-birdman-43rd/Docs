# 操舵系BBM試験計画

更新日: 2026-10-12
状態: 試験計画。コードは未作成、実機試験は未実施。

## 1. 目的と進め方

回路図・PCBを確定する前に、開発ボードとモジュールを用いたBBMで操舵経路を検証する。
まずCockpit–Tailの2ノードで通信を成立させ、その後Measurementを追加する。
本番の電源・保護・WDT回路は別途検証が必要であり、DevKitの試験成功だけで本番基板の検証完了とはしない。

## 2. 構成と調達状況

| 機能 | 使用部品 | 状態 |
|---|---|---|
| CAN物理層 | スイッチサイエンス SSCI-110044 TCAN3413搭載 CANFDトランシーバーモジュール（3.3V対応）×3 | 購入済み（2026-10-07報告） |
| MCU | ESP32-S3-DevKitC-1 ×最大3 | 使用する個体・型番・台数を確認 |
| 入力 | ラダー／エレベータ用ジョイスティック各1、トリムUP/DOWN | 現物仕様確認 |
| 出力 | ICS 3.3V/5Vインターフェース、KRS-5034HV ×2 | サーボ段階で追加 |
| 計測 | INA226モジュール | 後段で追加 |
| 電源 | USB／安定化電源、サーボ用電源 | 各段階の負荷と定格を確認 |

CAN FD対応なのはトランシーバである。ESP32-S3内蔵TWAIはClassic CAN用であり、今回の試験は500 kbps、最大8 byte/frameで実施する。SPI CANコントローラは追加しない。

## 3. TCANモジュールとESP32-S3の接続

| モジュール端子 | ESP32-S3側 |
|---|---|
| VCC | 3.3 V |
| VIO | 3.3 V |
| GND | GND |
| TXD | GPIO4 |
| RXD | GPIO5 |
| STB | GPIO40 |

- SJ1は開放。GPIO40でSTBを制御する場合、GNDへの固定接続と併用しない。
- SJ2はVIO–VCC接続用。ショートした場合、VIOへの別配線は不要。
- SJ1/SJ2の出荷時状態は現物で確認する。
- STBはHIGHでStandby、LOWでNormal。起動時Standby、TWAI初期化後Normalを試す。
- 電源は各ノードの3.3 Vから供給する。モジュールVCCへ5 VやLiPoを直接印加しない。
- ノード間でCANH同士、CANL同士、GND同士を接続する。別ノードの3.3 V出力同士は結ばない。
- モジュール搭載の120 Ω終端スイッチを使用し、外付け120 Ωを重複追加しない。
- Cockpit ON、Tail ON、Measurement OFF。全電源OFFでCANH–CANLが約60 Ωであることを確認する。
- Cockpit–Tailを幹線とし、MeasurementはCockpit付近から目標0.3 m以下のスタブ。
- 本番CANハーネスのコネクタはXA 4極、1=CANH、2=CANL、3=GND、4=NC。モジュール端子の物理順序とは区別する。

## 4. GPIO

| GPIO | 共通機能 | Cockpit固有 | Tail固有 |
|---:|---|---|---|
| 4 | CAN TX | — | — |
| 5 | CAN RX | — | — |
| 6 | — | Rudder ADC1 | ICS TX |
| 7 | — | Elevator ADC1 | ICS RX |
| 8 | — | — | ICS EN |
| 9 | TPS2116 ST（本番） | — | — |
| 10 | — | Trim UP | — |
| 11 | — | Trim DOWN | — |
| 15 | WDI（本番） | — | — |
| 16 | I2C SCL | — | — |
| 17 | I2C SDA | — | — |
| 19/20 | USB D−/D+ | — | — |
| 40 | TCAN STB | — | — |
| 41/42 | ERROR/STAT LED | — | — |
| 43/44 | Debug UART TX/RX | — | — |

GPIO6/7の用途は基板ごとに異なる。WDI相当のGPIO出力だけでは外部WDT検証にならない。

## 5. 段階試験

| 段階 | 試験 | 記録・判定 |
|---:|---|---|
| 1 | 2ノード500 kbps、連番送受信 | 送信要求／送信完了／受信／欠落／重複／エラーを区別。起動・終了境界を除いた正常区間で欠落・bus-offなしを目標 |
| 2 | ハーネス延長・連続通信 | ケーブル長、時間、受信間隔最大値、TWAIエラーカウンタ |
| 3 | ジョイスティック2軸ADC | 抵抗値・ピン配置・出力電圧・中立・端点・ノイズ・フィルタ遅延 |
| 4 | 操舵値50 Hz送信、Tailで表示 | 正規化値、符号、更新間隔。まだサーボを動かさない |
| 5 | ICSサーボ1台→2台 | ID、正方向、可動範囲、応答、実位置、電源変動 |
| 6 | Elevator Trim UP/DOWN | 1押下、長押し、上限下限、同時押し無効 |
| 7 | 通信停止・復帰 | 300 ms timeout、約0.7 s trim位置復帰、復帰時のレート制限 |
| 8 | Measurement追加 | 不在、電源OFF、再起動、送信過多時の操舵影響 |
| 9 | USB起動・電源切替 | USB起動によるサーボ動作を許容仕様として確認 |
| 10 | INA226追加 | 電圧・電流、シャント校正、I2C異常でも操舵処理が停止しない |
| 11 | 数時間連続運転・代表負荷 | リセット、欠落、bus-off、温度、ADC/CANへのサーボノイズ |

Measurement枝の断線・短絡試験は、最初はサーボを外した通信のみの構成で行う。短絡時にも同一バスが通信継続するとは仮定しない。操舵停止と復帰・切離し挙動を記録する。
サーボ電源の大電流をブレッドボードや細いジャンパ線へ流さず、専用配線で供給する。

## 6. 試験コードと結果の管理方針

試験コードもGitHubで管理する。Docsは設計・試験手順・結果を管理し、実装はファームウェア用リポジトリで管理する案。
新規リポジトリの名称・配置先は未確定であり、今回リポジトリ作成やコード実装は行っていない。

想定構成:

| パス例 | 内容 |
|---|---|
| tests/can_basic/ | 独立したPlatformIOプロジェクト。2ノード通信 |
| tests/can_three_nodes/ | 3ノード・計測ノード故障試験 |
| tests/joystick_adc/ | ADC範囲・ノイズ試験 |
| tests/ics_servo/ | ICS・サーボ試験 |
| docs/ | 配線・実行方法・Docs内試験結果へのリンク |

各コードにREADME、platformio.ini、ソースを付属させる。PlatformIO／framework／依存ライブラリの版、使用ボード、GPIO、終端設定、実行方法、正常時表示を記録する。
.pioやビルド生成物、認証情報はコミットしない。
試験結果にはコードのcommit SHA、日時、試験者、配線・電源、ハーネス長、送信周期、ログ、判定、未解決事項を残す。
試験用ID／payloadは本番プロトコルの凍結と区別する。

## 7. 次段階への移行

BBMで得たジョイスティック条件、ADC定数、軸方向、トリム量、フェイルセーフ挙動、CAN条件を仕様書へ戻す。
その後、共通コア回路（ESP32-S3、USB、電源MUX、WDT、CAN）を作成し、Cockpit固有入力とTail固有ICS／サーボ電源を追加する。

## 8. 参照

- [Cockpit仕様](../specifications/cockpit_control_board_spec.md)
- [Tail仕様](../specifications/tail_control_board_spec.md)
- [共通CAN仕様](../specifications/steering_can_interface.md)
- [SSCI-110044公式製品情報](https://www.switch-science.com/products/11004)
- [ESP32-S3 TWAI公式資料](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/twai.html)
