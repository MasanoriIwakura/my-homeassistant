# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack

Self-hosted Home Assistant deployment, configured entirely through YAML in `config/`. Three services run via `compose.yml`:

- **homeassistant** — `ghcr.io/home-assistant/home-assistant:stable`, host network, mounts `./config` to `/config`.
- **mosquitto** — MQTT broker on host ports 1883/9001, config at `config/mosquitto.conf`, persistent state at `./data` and `./log` (gitignored).
- **echonetlite2mqtt** — `banban525/echonetlite2mqtt`, host network, bridges ECHONET Lite devices on the LAN to MQTT via `MQTT_BROKER=mqtt://localhost:1883`. Web UI at `http://<host>:3000`.

## Commands

```bash
docker compose up -d        # start the stack
docker compose logs -f homeassistant
docker compose restart homeassistant   # required after editing configuration.yaml
```

To reload only the YAML files (faster than restart): in HA UI → 開発者ツール → YAML → reload the relevant section ("自動化", "テンプレート", or "MQTT エンティティ").

There is no test suite, linter, or build step. Validate YAML changes via `docker compose exec homeassistant python -m homeassistant --script check_config -c /config` or by reloading in the UI.

## Architecture

The repository monitors a Nichicon solar/V2H system. Two ECHONET Lite devices are exposed by echonetlite2mqtt and consumed by Home Assistant via MQTT:

| Device ID (truncated) | EOJ | Class | Role |
|---|---|---|---|
| `…42624b4a…` | `02a501` | 0x02A5 multipleInputPCS | grid I/O at the PCS connection point |
| `…4257357052…` | `027901` | 0x0279 pvPowerGeneration | solar PV inverter |

There is **no battery** in this installation, which simplifies the energy balance to `PV発電 + 買電 = 家庭消費 + 売電`.

### Data flow

1. echonetlite2mqtt **does not auto-poll** in v3+ (the legacy `ECHONET_INTERVAL_TO_GET_PROPERTIES` env var is deprecated). To force a fresh ECHONET Lite Get against a device, publish an empty payload to `echonetlite2mqtt/elapi/v2/devices/{deviceId}/properties/{propertyName}/request`.
2. `automations.yaml` runs two polling loops that publish to those request topics:
   - `echonetlite_polling_instant` — every 10s, refreshes `instantaneousElectricPower` (PCS) and `instantaneousElectricPowerGeneration` (PV).
   - `echonetlite_polling_cumulative` — every 60s, refreshes `normalDirectionElectricEnergy`, `reverseDirectionElectricEnergy`, `cumulativeElectricEnergyOfGeneration`. Cumulative reads are kept slow on purpose to avoid hammering Wi-SUN/PCS.
3. On every property change echonetlite2mqtt republishes to **`echonetlite2mqtt/elapi/v2/devices/{deviceId}/properties`** (flat `{shortName: value}` JSON) and `…/properties/{propertyName}` (raw value, retained). The device-level topic `…/devices/{deviceId}` is only populated at startup / device-list changes — do **not** subscribe to it for live values. MQTT sensors in `configuration.yaml` therefore use the `/properties` topic and pull individual properties out via `value_template`.
4. Template sensors then derive higher-level metrics:
   - `sensor.surplus_power` (= `余剰電力`, also the true surplus in this no-battery setup) and `sensor.power_buy_instant` split the signed `power_instant` into positive/negative halves.
   - `sensor.home_consumption` = `solar_instant + power_instant` (signed). Equivalent to `発電 + 買電 − 売電` because grid I/O is signed.
5. `utility_meter` integrations roll the cumulative kWh sensors into daily/monthly buckets, which `sell_price_*` templates multiply by ¥16/kWh.

### Sign convention to remember

`sensor.power_instant` (raw `instantaneousElectricPower` from PCS) is **signed**: positive = importing from grid (買電), negative = exporting (売電). Several derived sensors depend on this. If the upstream property ever flips sign, every template in the file breaks together.

## Conventions

- All MQTT and template entities have explicit `unique_id` **and** `default_entity_id`. The entity IDs referenced throughout the file (`sensor.power_instant`, etc.) are pinned by `default_entity_id`, **not** auto-derived from the Japanese `name`. Without it, HA's `slugify` transliterates Japanese to Chinese-reading romaji (e.g. `瞬時電力` → `sensor.shun_shi_dian_li`) and `買電量` / `売電量` even collide onto the same slug `mai_dian_liang`. Always set both keys on new MQTT/template entities. Note: `default_entity_id` only takes effect on **first** registration — for entities already in `core.entity_registry` under a wrong ID, rename them via the HA UI (the `unique_id` preserves history). `utility_meter` is unaffected because the dictionary key under `utility_meter:` is itself the entity_id slug.
- Energy sensors use `device_class: energy` + `state_class: total_increasing` so they appear in the HA Energy dashboard.
- Power sensors use `device_class: power` + `state_class: measurement`.

## Adding a new ECHONET Lite property

1. Look up the camelCase `shortName` in the MRA dictionary at `https://github.com/banban525/echonetlite2mqtt/blob/master/MRA_v1.3.1/devices/0x{class}.json`.
2. Add an MQTT sensor block in `configuration.yaml` whose `state_topic` is `…/devices/{deviceId}/properties` (with the `/properties` suffix) and whose `value_template` pulls `value_json.{shortName}`. Set `unique_id` and `default_entity_id: sensor.<ascii_name>` so the entity_id is deterministic regardless of the Japanese `name`.
3. Add an `mqtt.publish` action to the appropriate polling automation (instant vs cumulative) so the value actually refreshes — without this, the sensor will sit at its startup value.

## Git workflow

- Never commit `secrets.yaml`, the SQLite DB (`home-assistant_v2.db`), or `.storage/` — all are gitignored.
- The `data/` and `log/` directories created by mosquitto at runtime are gitignored as well.
