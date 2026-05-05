# my-homeassistant

自宅の Home Assistant 構成。ニチコンの太陽光パワコン + マルチ入力 PCS を ECHONET Lite 経由で監視するためのもの。

## 構成

```
[ニチコン PV / PCS] --ECHONET Lite--> [echonetlite2mqtt] --MQTT--> [mosquitto] --MQTT--> [Home Assistant]
```

`compose.yml` で 3 つのサービスを立ち上げる:

| サービス | イメージ | 役割 |
|---|---|---|
| `homeassistant` | `ghcr.io/home-assistant/home-assistant:stable` | 母艦 |
| `mosquitto` | `eclipse-mosquitto:latest` | MQTT ブローカー (1883/9001) |
| `echonetlite2mqtt` | `banban525/echonetlite2mqtt` | ECHONET Lite ⇄ MQTT ブリッジ |

## 起動

```bash
docker compose up -d
```

- Home Assistant: `http://<host>:8123`
- echonetlite2mqtt の web UI: `http://<host>:3000`

設定変更後は `docker compose restart homeassistant`、もしくは HA の「開発者ツール → YAML」から該当セクションをリロード。

## 監視している ECHONET Lite デバイス

| デバイス | クラス | 内容 |
|---|---|---|
| pvPowerGeneration | 0x0279 | 太陽光パワコン |
| multipleInputPCS | 0x02A5 | PCS（系統側 I/O 計測点） |

蓄電池なし構成。

## センサー

### MQTT 由来（生値）

| エンティティ | 単位 | 説明 |
|---|---|---|
| `sensor.power_buy_total` | kWh | 積算買電量 |
| `sensor.power_sell_total` | kWh | 積算売電量 |
| `sensor.solar_total` | kWh | 積算発電電力量 |
| `sensor.power_instant` | W | 瞬時電力（**符号付き**: +買/-売） |
| `sensor.solar_instant` | W | 瞬時発電電力 |

### テンプレート（派生値）

| エンティティ | 単位 | 式 |
|---|---|---|
| `sensor.surplus_power`（余剰電力） | W | `max(0, -power_instant)` |
| `sensor.power_buy_instant`（瞬時買電） | W | `max(0, power_instant)` |
| `sensor.home_consumption`（家庭消費） | W | `solar_instant + power_instant` |

### utility_meter

`power_buy` / `power_sell` / `solar` それぞれに日次・月次バケット。`sensor.sell_price_daily` / `sell_price_monthly` で ¥16/kWh の売電金額換算。

## ポーリング戦略

echonetlite2mqtt v3 系は自動ポーリングをしない（`ECHONET_INTERVAL_TO_GET_PROPERTIES` は deprecated）ので、`automations.yaml` から MQTT publish で能動的に取得する。

- `echonetlite_polling_instant`: 10 秒間隔で瞬時値を取得
- `echonetlite_polling_cumulative`: 60 秒間隔で積算値を取得

リクエストトピック:
```
echonetlite2mqtt/elapi/v2/devices/{deviceId}/properties/{propertyName}/request
```
（payload は空）

## ライセンス

LICENSE 参照。
