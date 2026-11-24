# Van Home Assistant packages

Home Assistant packages for the van (HAOS 2026.11, WiCAN OBD + Victron GX GPS).

| Package | What it does |
|---|---|
| `packages/van_mpg.yaml` | Engine-live detection, GPS speed, distance and odometer, fuel totals, and 10 min / 2 h / 12 h average MPG measured in **engine time** |
| `packages/wican_extra.yaml` | WiCAN readings that only the WiCAN integration provided (timing advance, relative throttle, time since engine start, distance with check-engine light on, warm-ups and distance since codes cleared), as MQTT sensors on the WiCAN device |

## Install

`configuration.yaml` already loads packages with:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Copy `packages/van_mpg.yaml` to `/config/packages/van_mpg/van_mpg.yaml` (or clone this repo
anywhere under `/config/packages/`; only `.yaml` files are loaded). Then
**Settings → System → Restart**, because the `integration` platform sensors need a restart, not a reload.

## How the MPG averages work

1. **Engine live**: `binary_sensor.wican_engine_live` is on only while RPM > 300 **and** the WiCAN
   reported in the last 30 s. The WiCAN goes quiet at shutdown and its last values get stuck, so
   freshness matters more than the value itself.
2. **Fuel total**: `sensor.wican_fuel_rate_engine_running` (g/s) is 0 unless the engine is live and
   the fuel reading is fresh. `sensor.fuel_use_total_grams` integrates it.
3. **Distance**: Victron GPS speed (m/s → mph, under 1 mph = 0) is integrated into
   `sensor.victron_nmea_0183_gps_acm0_gps_distance_driven` (mi).
   `sensor.gps_tracked_odometer` = `input_number.startingodometerreading` + that total.
4. **Accumulator**: `sensor.mpg_accumulator` is a trigger-based template that fires on every change of
   the fuel or distance total **while the engine is live**. It keeps exponentially weighted sums of
   fuel and miles with time constants of 10 min, 2 h and 12 h of engine time. The decay step uses
   the time since the last update, capped at 15 s, so parked time never counts.
5. **Average MPG** = weighted miles ÷ (weighted grams ÷ 2819 g/gal). Below 20 g of weighted fuel the
   sensor is `unknown`.

Why not average the instant MPG? Coasting reads 99 MPG (no fuel), and a time average of instant MPG
is badly inflated by those readings. Dividing miles by gallons weights each moment by the fuel
actually burned, which is what a fill-up calculation measures.

Why engine time? With clock-time windows (statistics helpers), a 2-hour window holds only whatever
happened in the last 2 hours. A short idle in the driveway then makes it read about 0 MPG. In engine time
the 2-hour figure is always your last ~2 hours of running.

## Entities

| Entity | Notes |
|---|---|
| `binary_sensor.wican_engine_live` | Engine running, from fresh WiCAN data |
| `sensor.nmea_0183_gps_acm0_victron_vehicle_speed` | mph |
| `sensor.gps_speed_for_distance` | Same value labelled mi/h, used as the integration source |
| `sensor.wican_fuel_rate_engine_running` | g/s, gated |
| `sensor.fuel_use_total_grams` | g |
| `sensor.victron_nmea_0183_gps_acm0_gps_distance_driven` | mi |
| `sensor.gps_tracked_odometer` | mi |
| `sensor.victron_nmea_0183_gps_acm0_gps_max_speed_10_min` | highest speed in the last 10 min (clock time) |
| `sensor.mpg_accumulator` | engine minutes logged; weighted sums in attributes |
| `sensor.average_mpg_10_min` / `_2_hour` / `_12_hour` | MPG |

Real-time (instant) MPG lives in `packages/wican/wican.yaml` (`sensor.real_time_fuel_economy`).

## Dashboard: MPG bars

Four horizontal bars (instant, 10 min, 2 h, 12 h) stacked vertically, as on the van's Drive tab.
They are built-in tile cards with the `bar-gauge` feature (scale 0–30 MPG). The `card_mod` styling
squeezes each tile to 51 px and bottom-aligns the stack; it needs
[card-mod](https://github.com/thomasloven/lovelace-card-mod) from HACS. Leave the `card_mod` blocks out
for standard-height tiles.

```yaml
type: vertical-stack
grid_options:
  columns: 13
  rows: auto
card_mod:
  style: |
    :host { height: 220px; --vertical-stack-card-gap: 5px; }
    #root { height: 100%; justify-content: flex-end; }
cards:
  - type: tile
    entity: sensor.real_time_fuel_economy
    name: Instant
    icon: mdi:gas-station
    color: green
    vertical: false
    features_position: inline
    features:
      - type: bar-gauge
        min: 0
        max: 30
    card_mod: &mpg_bar_style
      style: |
        ha-card { height: 51px !important; min-height: 0 !important; --tile-icon-size: 28px; }
        .container { padding: 0 8px !important; }
        .content { padding: 0 !important; min-height: 0 !important; height: 51px !important; }
        hui-card-features { --feature-height: 20px; padding: 0 12px 0 0 !important; }
  - type: tile
    entity: sensor.average_mpg_10_min
    name: 10 min
    icon: mdi:gas-station
    color: blue
    vertical: false
    features_position: inline
    features:
      - type: bar-gauge
        min: 0
        max: 30
    card_mod: *mpg_bar_style
  - type: tile
    entity: sensor.average_mpg_2_hour
    name: 2 hr
    icon: mdi:gas-station
    color: indigo
    vertical: false
    features_position: inline
    features:
      - type: bar-gauge
        min: 0
        max: 30
    card_mod: *mpg_bar_style
  - type: tile
    entity: sensor.average_mpg_12_hour
    name: 12 hr
    icon: mdi:gas-station
    color: purple
    vertical: false
    features_position: inline
    features:
      - type: bar-gauge
        min: 0
        max: 30
    card_mod: *mpg_bar_style
```

The YAML anchor (`&mpg_bar_style` / `*mpg_bar_style`) only works when you paste the card into the
dashboard's raw configuration editor; the UI card editor expands it on save. `grid_options` applies to
sections views. `:host { height: 220px }` sizes it for a 1480×320 display; change it for your screen
or remove it.
