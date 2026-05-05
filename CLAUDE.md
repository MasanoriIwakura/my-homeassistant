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
| `…42624b4a…` | `02a501` | 0x02A5 multipleInputPCS | PV-only PCS (no battery / no V2H connected). `connectedDeviceList` (0xE8) reports just `0x027901`. |
| `…4257357052…` | `027901` | 0x0279 pvPowerGeneration | solar PV inverter; also exposes the **true cumulative sold-to-grid** counter |

There is **no battery** in this installation, but the energy balance still has a hole: **we have no real-time grid I/O measurement**. None of the available properties measures at the grid coupling point — see "Property gotchas" below.

### Data flow

1. echonetlite2mqtt **does not auto-poll** in v3+ (the legacy `ECHONET_INTERVAL_TO_GET_PROPERTIES` env var is deprecated). To force a fresh ECHONET Lite Get against a device, publish an empty payload to `echonetlite2mqtt/elapi/v2/devices/{deviceId}/properties/{propertyName}/request`.
2. `automations.yaml` runs two polling loops that publish to those request topics:
   - `echonetlite_polling_instant` — every 10s, refreshes `instantaneousElectricPower` (PCS) and `instantaneousElectricPowerGeneration` (PV).
   - `echonetlite_polling_cumulative` — every 60s, refreshes PCS `normalDirectionElectricEnergy`, PV `cumulativeElectricEnergyOfGeneration`, and PV `cumulativeElectricEnergySold`. Cumulative reads are kept slow on purpose to avoid hammering Wi-SUN/PCS.
3. On every property change echonetlite2mqtt republishes to **`echonetlite2mqtt/elapi/v2/devices/{deviceId}/properties`** (flat `{shortName: value}` JSON) and `…/properties/{propertyName}` (raw value, retained). The device-level topic `…/devices/{deviceId}` is only populated at startup / device-list changes — do **not** subscribe to it for live values. MQTT sensors in `configuration.yaml` therefore use the `/properties` topic and pull individual properties out via `value_template`.
4. `utility_meter` integrations roll the cumulative kWh sensors into daily/monthly buckets, which `sell_price_*` templates multiply by ¥16/kWh.

### Property gotchas (verified empirically against this hardware)

These are not in MRA — they were discovered by direct ECHONET Lite GET via the `es-e1-echonet-ts` CLI:

- **PCS `instantaneousElectricPower` (0xE7) is NOT grid I/O.** It tracks the PCS's own AC-side flow (≒ `-solar_instant`), so house consumption is invisible. At night with significant load it sits near 0 (e.g., 9W standby). Do not use it to derive `home_consumption`, `surplus_power`, or `power_buy_instant`.
- **PCS `reverseDirectionElectricEnergy` (0xE3) is NOT cumulative sold-to-grid.** It tracks total DC→AC conversion at the PCS, which equals total PV generation (≈ PV `E1`), not what was sold. On this device it overstates real sales by ≈77%. Use **PV `cumulativeElectricEnergySold` (0xE3 on `0x0279`)** instead — this is the true grid-export counter (PV E1 − PV E3 = lifetime self-consumption from solar).
- **PCS `normalDirectionElectricEnergy` (0xE0) is NOT cumulative purchased-from-grid.** It tracks AC→DC at the PCS itself (≒ PCS standby draw, accumulates very slowly). Real cumulative buy needs a smart meter (`0x0288` via Wi-SUN Route B) which this setup does not yet have.
- 0x02A5 declares an incomplete `getPropertyMap` (0x9F) — `0xE3` and `0xE7` work via direct GET despite not being listed. Don't rely on the property map alone when probing this device.

### Tesla solar-surplus charging logic

Because real grid I/O is not measurable, the Tesla automation runs in a **solar-only estimation** mode: `target_amps = (solar_instant − estimated_house_w − solar_charge_buffer_w) / 200`. `input_number.estimated_house_w` is a manual stand-in for whole-home consumption — bump it on hot days when AC runs. `solar_charge_min_amps` reflects the vehicle's hardware floor (10A on this setup); below that the loop stops charging instead of issuing setpoints the car ignores.

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
