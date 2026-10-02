# CAN ID List

CAN ID の正本です。

> 未決定のIDを実装側だけで先に固定しないこと。IDを追加・変更した場合は、この表とファームウェア側の共通プロトコル定義を同時に更新します。

| CAN ID | Message | Sender | Receiver | DLC | Period / Trigger | Description | Status |
|---|---|---|---|---:|---|---|---|
| TBD | Rudder command | Cockpit | Tail | TBD | TBD | ラダー操舵指令 | 未決定 |
| TBD | Elevator command | Cockpit | Tail | TBD | TBD | エレベーター操舵指令 | 未決定 |
| TBD | Tail status | Tail | Cockpit | TBD | TBD | テール側状態・異常通知 | 未決定 |

## Rules

- ID の重複を禁止する
- Command と Telemetry / Status の役割を明確に分ける
- 周期送信かイベント送信かを明記する
- 異常時の扱いを各メッセージで定義する
