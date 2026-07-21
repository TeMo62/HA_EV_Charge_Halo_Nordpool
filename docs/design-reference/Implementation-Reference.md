# EV Charging Planner Implementation Reference

## 1. Purpose

### Post-snapshot corrections

The reference primarily documents the exported Home Assistant snapshot. The
following runtime corrections were verified after that export:

- On 2026-07-22, `EV Charge Stop` was updated to read the registered entity
  `sensor.sensor_ev_session_energy`. The earlier reference to
  `sensor.ev_session_energy` no longer applies to the live automation.
- `EV Restore Default Values` and `EV Restore Defaults After Day Charging
  Window` were compared with the live YAML on 2026-07-22 and match the
  implementations reproduced in Sections 6.1 and 5.5.

Historical sections below retain the exported names so the snapshot remains
auditable. When operating the live system, the corrections above take
precedence.

This document is an implementation-driven reference for the EV Charging Planner contained in `config.zip`, supplemented by the EV default-value restore changes applied on 2026-07-21. It records implemented configuration rather than intended future behaviour.

Implementation snapshot facts:

- Home Assistant version: `2026.7.2`.
- Primary YAML entry point: `configuration.yaml`.
- Automation include: `automations.yaml`.
- Script include: `scripts.yaml`.
- Scene include: `scenes.yaml`.
- GUI-created helpers, template entities, integration entries, entity registrations, and the EV Energy dashboard are stored below `.storage/`.
- OCPP custom integration version: `v0.10.15`.
- Nord Pool custom integration version: `0.0.18`.

When a GUI-created object has no YAML representation, this reference includes its exact storage-backed configuration instead of inventing YAML. When a relationship cannot be established from the exported implementation, it is stated as **Not verified from implementation.**

The restored states quoted in this document are observations from `.storage/core.restore_state`. They are not configuration defaults.

Post-export change recorded on 2026-07-21:

- `script.ev_restore_default_values` restores the four active day/night target and current helpers from their default helpers without changing `input_number.ev_max_charge_current`.
- `automation.ev_restore_defaults_after_day_charging_window` calls the script when the selected day block ends.
- The EV Energy Settings view contains a confirmation-protected button that calls the same script.
- Manual script execution was verified successfully by the user. Automatic end-of-window execution has not yet been runtime-observed.

## 2. Configuration Overview

### 2.1 Root configuration

`configuration.yaml` contains:

- `default_config`.
- frontend theme loading from `themes`.
- includes for `automations.yaml`, `scripts.yaml`, and `scenes.yaml`.
- debug logging for OCPP and WebSockets.
- one YAML-defined input number, `input_number.halo_charge_current`.
- unrelated Emulated Hue configuration.
- a top-level block named `EV Charger Fail Safe On Startup`.
- YAML template entities for actual current, L1 current, and tomorrow-price availability.

The `EV Charger Fail Safe On Startup` block is physically present at the top level of `configuration.yaml`; it is not nested below `automation:` and no corresponding automation entity is present in `.storage/core.entity_registry`. Its registration as an executable automation is therefore **Not verified from implementation.** The exact block is reproduced in Section 5.8.

### 2.2 Included YAML files

| File | Implementation content relevant to this project |
|---|---|
| `automations.yaml` | Five enabled EV runtime automations and five disabled/restored-off EV automations. The added runtime automation invokes the default-value restore script when the selected day block ends. |
| `scripts.yaml` | `EV Restore Default Values` plus two unrelated lighting scripts. |
| `scenes.yaml` | Included by the root configuration; no EV reference was found. |

### 2.3 Storage-backed configuration

| Storage file | EV-related content |
|---|---|
| `.storage/input_boolean` | Smart-charging enable helper. |
| `.storage/input_datetime` | Legacy start/stop times, day/night windows, and plan update timestamp. |
| `.storage/input_number` | Runtime targets, currents, defaults, current limit, and legacy energy/power helpers. |
| `.storage/input_text` | Selected day and night charge blocks. |
| `.storage/core.config_entries` | Template helpers, OCPP configuration, and two Nord Pool config entries. |
| `.storage/core.entity_registry` | Final entity IDs, including renamed helpers and template entities. |
| `.storage/core.restore_state` | Last restored runtime states and enabled/disabled automation states. |
| `.storage/lovelace.ev_energy` | EV Energy dashboard definition. |
| `.storage/lovelace_dashboards` | Dashboard registration. |
| `.storage/lovelace_resources` | ApexCharts Card frontend resource. |

### 2.4 Integration entries

The Nord Pool sensor used by the planner is registered to config entry `01KRYCJHDFSZ5H5EDSPW15Z4CT` with region `SE3`, currency `SEK`, price type `kWh`, VAT enabled, precision `3`, `price_in_cents: false`, and additional cost template `{{0.0|float}}`.

A second enabled Nord Pool config entry, `01KS06ZB4TP1XETVMMJFM3C7SQ`, is also stored with `price_in_cents: true`. The registered entity `sensor.nordpool_kwh_se3_sek_3_10_025` belongs to the first entry. No EV planner entity was found for the second entry.

The OCPP entry `01KXVKY1EFZ2W608DFWEMKXMV0` is reproduced in Section 8.

## 3. Active Helpers

### 3.1 Configuration and runtime helpers

No storage-backed EV helper contains an explicit `initial` value. Home Assistant restores their last state. The “Restored snapshot” column is therefore observational, not a declared default.

| Entity ID | Type and configured constraints | Purpose in current implementation | Explicit default | Restored snapshot | Read by | Written by | Implementation location |
|---|---|---|---|---|---|---|---|
| `input_boolean.ev_smart_charging_enabled` | Boolean | Gates `EV Charge Start` and `EV Charge Stop`. | Not configured | `on` | Start and stop automations; dashboard | User/dashboard | `.storage/input_boolean` |
| `input_datetime.ev_day_window_start` | Time | Start of the day candidate window. | Not configured | `10:00:00` | `EV Build Charge Plan`; day cost template | User/dashboard | `.storage/input_datetime` |
| `input_datetime.ev_day_window_end` | Time | End of the day candidate window. | Not configured | `21:00:00` | `EV Build Charge Plan`; day cost template | User/dashboard | `.storage/input_datetime` |
| `input_datetime.ev_night_window_start` | Time | Start of the night candidate window. | Not configured | `00:15:00` | `EV Build Charge Plan`; night cost template | User/dashboard | `.storage/input_datetime` |
| `input_datetime.ev_night_window_end` | Time | End of the night candidate window. | Not configured | `08:00:00` | `EV Build Charge Plan`; night cost template | User/dashboard | `.storage/input_datetime` |
| `input_datetime.ev_plan_last_updated` | Date and time | Records completion time of the latest plan-build action. | Not configured | `2026-07-19 23:00:00` | EV Energy ApexCharts series | `EV Build Charge Plan` | `.storage/input_datetime` |
| `input_number.ev_day_target_kwh` | Number, 0–11 kWh, step 1, slider | Day energy target. Storage item ID is `ev_day_target_kwh_2`; entity registry exposes the ID shown here. | Not configured | `7.0` | Plan builder, day cost, stop trigger/condition | User/dashboard; `EV Restore Default Values` | `.storage/input_number`; `.storage/core.entity_registry` |
| `input_number.ev_night_target_kwh` | Number, 0–11 kWh, step 1, slider | Night energy target. Storage item ID is `ev_night_target_kwh_2`; entity registry exposes the ID shown here. | Not configured | `6.0` | Plan builder, night cost | User/dashboard; `EV Restore Default Values` | `.storage/input_number`; `.storage/core.entity_registry` |
| `input_number.ev_day_charge_current` | Number, 6–16 A, step 1, box | Day profile current. | Not configured | `10.0` | Plan builder; `EV Charge Start`; day cost | User/dashboard; `EV Restore Default Values` | `.storage/input_number` |
| `input_number.ev_night_charge_current` | Number, 6–15 A, step 1, box | Night profile current. | Not configured | `10.0` | Plan builder; `EV Charge Start`; night cost | User/dashboard; `EV Restore Default Values` | `.storage/input_number` |
| `input_number.ev_default_day_target_kwh` | Number, 0–11 kWh, step 1, box | Source used when the restore script resets the day target. | Not configured | `10.0` | `EV Restore Default Values` | User/dashboard | `.storage/input_number` |
| `input_number.ev_default_night_target_kwh` | Number, 0–11 kWh, step 1, box | Source used when the restore script resets the night target. | Not configured | `11.0` | `EV Restore Default Values` | User/dashboard | `.storage/input_number` |
| `input_number.ev_default_day_charge_current` | Number, 6–10 A, step 1, box | Source used when the restore script resets day current. | Not configured | `10.0` | `EV Restore Default Values` | User/dashboard | `.storage/input_number` |
| `input_number.ev_default_night_charge_current` | Number, 6–10 A, step 1, box | Source used when the restore script resets night current; also used separately by `EV Charge Stop` for maximum current. | Not configured | `10.0` | `EV Restore Default Values`; `EV Charge Stop` | User/dashboard | `.storage/input_number` |
| `input_number.ev_max_charge_current` | Number, 6–16 A, step 1, slider | Runtime current limit that triggers the OCPP charge-rate automation. | Not configured | `10.0` | `Set Halo charge current` | `EV Charge Start`; `EV Charge Stop`; user if edited directly | `.storage/input_number` |
| `input_text.ev_day_selected_charge_block` | Text, max 20 | Stores `HH:MM–HH:MM` or `none` for the selected day block. | Not configured | `12:30–16:30` | Day recommendation; dashboard graph/status; plan-build condition | `EV Build Charge Plan`; disabled day lock automation | `.storage/input_text` |
| `input_text.ev_night_selected_charge_block` | Text, max 20 | Stores `HH:MM–HH:MM` or `none` for the selected night block. | Not configured | `03:15–06:15` | Night recommendation; dashboard graph/status | `EV Build Charge Plan`; disabled night lock automation | `.storage/input_text` |

### 3.2 Registered legacy or currently unreferenced helpers

These helpers are registered and available, but no reference from the active planner/start/stop chain was found.

| Entity ID | Type and configured constraints | Stored purpose evidence | Explicit default | Restored snapshot | References found | Implementation location |
|---|---|---|---|---|---|---|
| `input_datetime.ev_charge_start` | Time | Named as a charge start time. | Not configured | `22:00:00` | No project consumer found. | `.storage/input_datetime` |
| `input_datetime.ev_charge_stop` | Time | Named as a charge stop time. | Not configured | `07:00:00` | No project consumer found. | `.storage/input_datetime` |
| `input_number.ev_required_energy` | Number, 1–12 kWh, step 0.1, slider | Input to historical general planner templates. | Not configured | `10.0` | Historical template entries. | `.storage/input_number` |
| `input_number.ev_charger_power` | Number, 1–4, step 0.1, slider, unit stored as `kWh` | Input to historical general planner templates. | Not configured | `2.3` | Historical template entries. | `.storage/input_number` |
| `input_number.ev_charge_power_kw` | Number, 0.1–4 kW, step 0.1, slider | Name indicates charge power. | Not configured | `2.1` | No project consumer found. | `.storage/input_number` |
| `input_number.halo_charge_current` | YAML input number, 6–16 A, step 1 | Name indicates Halo current. | Not configured | `10.0` | No project consumer found. | `configuration.yaml` |

## 4. Template Entities

### 4.1 YAML-defined template entities

#### `sensor.ev_actual_current`

- **Purpose:** Reports `sensor.halo_l1` while connector 1 status equals `Charging`; otherwise reports `0`.
- **Dependencies:** `sensor.charger_connector_1_status_connector`, `sensor.halo_l1`.
- **Implementation location:** `configuration.yaml`.

```yaml
  - sensor:
      - name: "EV_Actual_Current"
        unique_id: ev_actual_current
        unit_of_measurement: "A"
        device_class: current
        state_class: measurement
        state: >
             {% if states('sensor.charger_connector_1_status_connector') == 'Charging' %}
             {{ states('sensor.halo_l1') }}
                 {% else %}
                  0
                {% endif %}
```

#### `sensor.halo_l1`

- **Purpose:** Exposes the `L1` attribute from connector 1 current import.
- **Dependencies:** `sensor.charger_connector_1_current_import`.
- **Implementation location:** `configuration.yaml`.

```yaml
  - sensor:
      - name: "Halo L1"
        unique_id: halo_current_l1
        unit_of_measurement: "A"
        device_class: current
        state_class: measurement
        state: >
          {{ state_attr('sensor.charger_connector_1_current_import', 'L1') }}
```

#### `binary_sensor.tomorrow_prices_available`

- **Purpose:** True when Nord Pool marks tomorrow valid and exposes at least 96 raw tomorrow entries.
- **Dependencies:** `sensor.nordpool_kwh_se3_sek_3_10_025` attributes `tomorrow_valid` and `raw_tomorrow`.
- **Implementation location:** `configuration.yaml`.

```yaml
  - binary_sensor:
      - name: "Tomorrow Prices Available"
        unique_id: tomorrow_prices_available
        state: >
          {% set np = 'sensor.nordpool_kwh_se3_sek_3_10_025' %}
          {{
            state_attr(np, 'tomorrow_valid') == true
            and
            ((state_attr(np, 'raw_tomorrow') or []) | count) >= 96
          }}
```

### 4.2 Storage-backed template entities used by the current chain

GUI-created Template helpers are stored as JSON config entries. No YAML exists for these entities in the export. The exact stored `options` object is therefore reproduced without converting it to YAML.

| Config entry ID | Stored title | Registered entity | Template type |
|---|---|---|---|
| `01KRSFAAD82Y8Q1C89Y4BSFF62` | `sensor.ev_session_energy` | `sensor.sensor_ev_session_energy` | sensor |
| `01KRSG90P05RY2Q4F99J9XN4XV` | `EV_is_Charging` | `binary_sensor.ev_is_charging` | binary sensor |
| `01KRVS8WN2TW187MK711H566KP` | `EV_Charger_Available` | `switch.ev_charger_available` | switch |
| `01KRVSDTMQ4FMC7JW7YZMBR6AV` | `EV_Charge_Control` | `switch.ev_charge_control` | switch |
| `01KRVVC882PWEDF45PQNDC67SH` | `EV_Charger_Status` | `sensor.ev_charger_status` | sensor |
| `01KRYB7BRZ9YDWDCGCRWPSN9SW` | `ev_requiered_charging_hours` | `sensor.ev_requiered_charging_hours` | sensor |
| `01KRYC8J5JGA138E8DSRFAQR0X` | `ev_cheapest_charging_hours` | `sensor.ev_cheapest_charge_slots` | sensor |
| `01KRYGS796KJK2WR0GA59TT8P9` | `ev_cheapest_charge_slots` | `sensor.ev_cheapest_charge_slots_2` | sensor |
| `01KRYH2BSV13SREXJDPX6QWK6C` | `ev_estimated_charge_cost` | `sensor.ev_estimated_charge_cost` | sensor |
| `01KRYH7EN06B4B40FME6SJY8V8` | `EV_Cheapest_Continuous_Charge_Block` | `sensor.ev_cheapest_continuous_charge_block` | sensor |
| `01KS55FP5W46BH3YSN56ZWEWC2` | `ev_day_estimated_charge_cost` | `sensor.ev_day_estimated_charge_cost` | sensor |
| `01KS55RAJME0ZESE2P4RH2WP5H` | `ev_night_estimated_charge_cost` | `sensor.ev_night_estimated_charge_cost` | sensor |
| `01KS5AW758JSRVZ0214EBNYAAQ` | `ev_day_charge_recommended_now` | `binary_sensor.ev_day_charge_recommended_now` | binary sensor |
| `01KS5CR7DJ9RH00XY8QB3XFF9R` | `ev_night_charge_recommended_now` | `binary_sensor.ev_night_charge_recommended_now` | binary sensor |
| `01KS5CWSD6FG5K8YB6BR2Z3A1P` | `ev_charge_recommended_now` | `binary_sensor.ev_charge_recommended_now` | binary sensor |
| `01KS5FFJJ9NW834TKA7T5QSDTJ` | `ev_day_charge_visual` | `sensor.ev_day_charge_visual` | sensor |
| `01KS5FGYN3Q4TRE40A366D39AV` | `ev_night_charge_visual` | `sensor.ev_night_charge_visual` | sensor |
| `01KS5G76BE4NMAGKW208R1G5MQ` | `sensor.ev_day_visual_block` | `sensor.ev_day_visual_block` | sensor |
| `01KS5G8FQNXKZCGZZVGM1E4XVD` | `ev_night_visual_block` | `sensor.ev_night_visual_block` | sensor |

#### Session energy template

- **Stored title:** `sensor.ev_session_energy`.
- **Registered entity ID:** `sensor.sensor_ev_session_energy`.
- **Purpose:** Mirrors OCPP connector 1 session energy.
- **Dependencies:** `sensor.charger_connector_1_energy_session`.
- **Implementation note:** `EV Charge Stop` reads `sensor.ev_session_energy`, but that entity ID is absent from the exported entity registry. The template entry is registered as `sensor.sensor_ev_session_energy`. The stop automation's energy comparison therefore has an unresolved entity reference in this snapshot.

```json
{
  "name": "sensor.ev_session_energy",
  "state": "{{ states('sensor.charger_connector_1_energy_session') }}",
  "state_class": "total_increasing",
  "template_type": "sensor",
  "unit_of_measurement": "kWh"
}
```

#### `switch.ev_charger_available`

- **Purpose:** Template-switch facade over OCPP charger availability.
- **Dependencies:** `switch.charger_availability`.

```json
{
  "name": "EV_Charger_Available",
  "template_type": "switch",
  "turn_off": [
    {
      "action": "switch.turn_off",
      "data": {},
      "metadata": {},
      "target": {"entity_id": "switch.charger_availability"}
    }
  ],
  "turn_on": [
    {
      "action": "switch.turn_on",
      "data": {},
      "metadata": {},
      "target": {"entity_id": "switch.charger_availability"}
    }
  ],
  "value_template": "{{ is_state('switch.charger_availability', 'on') }}"
}
```

#### `switch.ev_charge_control`

- **Purpose:** Template-switch facade over OCPP connector 1 charge control.
- **Dependencies:** `switch.charger_connector_1_charge_control`.

```json
{
  "name": "EV_Charge_Control",
  "template_type": "switch",
  "turn_off": [
    {
      "action": "switch.turn_off",
      "data": {},
      "metadata": {},
      "target": {"entity_id": "switch.charger_connector_1_charge_control"}
    }
  ],
  "turn_on": [
    {
      "action": "switch.turn_on",
      "data": {},
      "metadata": {},
      "target": {"entity_id": "switch.charger_connector_1_charge_control"}
    }
  ],
  "value_template": "{{ is_state('switch.charger_connector_1_charge_control', 'on') }}"
}
```

#### `binary_sensor.ev_day_charge_recommended_now`

- **Purpose:** True while current time is within the selected day block.
- **Dependencies:** `input_text.ev_day_selected_charge_block`, current time.

```jinja
{% set block = states('input_text.ev_day_selected_charge_block') %}

{% if '–' not in block %}
  false
{% else %}
  {% set parts = block.split('–') %}
  {% set start = today_at(parts[0]) %}
  {% set end = today_at(parts[1]) %}

  {{ now() >= start and now() < end }}
{% endif %}
```

#### `binary_sensor.ev_night_charge_recommended_now`

- **Purpose:** True while current time is within the selected night block, with explicit midnight and next-day handling.
- **Dependencies:** `input_text.ev_night_selected_charge_block`, current time.

```jinja
{% set block = states('input_text.ev_night_selected_charge_block') %}

{% if '–' not in block %}
  false
{% else %}
  {% set parts = block.split('–') %}
  {% set start = today_at(parts[0]) %}
  {% set end = today_at(parts[1]) %}

  {% if end <= start %}
    {% set end = end + timedelta(days=1) %}
  {% endif %}

  {% if now().hour >= 12 %}
    {% set start = start + timedelta(days=1) %}
    {% set end = end + timedelta(days=1) %}
  {% endif %}

  {{ now() >= start and now() < end }}
{% endif %}
```

#### `binary_sensor.ev_charge_recommended_now`

- **Purpose:** Logical OR of the day and night recommendation sensors.
- **Dependencies:** `binary_sensor.ev_day_charge_recommended_now`, `binary_sensor.ev_night_charge_recommended_now`.

```jinja
{{ is_state('binary_sensor.ev_day_charge_recommended_now', 'on')
   or is_state('binary_sensor.ev_night_charge_recommended_now', 'on') }}
```

#### `sensor.ev_day_estimated_charge_cost`

- **Purpose:** Calculates the cost of the lowest-price continuous future block in the relevant day window using day target and day current.
- **Dependencies:** day target/current/window helpers, Nord Pool raw today/tomorrow, current time.

```jinja
{% set _ = now() %}
{% set energy = states('input_number.ev_day_target_kwh') | float(0) %}
{% set amps = states('input_number.ev_day_charge_current') | float(6) %}
{% set power = [((amps - 1) * 230 / 1000), 0.1] | max %}
{% set needed_slots = ((energy / (power * 0.25)) | round(0, 'ceil')) | int %}

{% set np = 'sensor.nordpool_kwh_se3_sek_3_10_025' %}
{% set all_prices = (state_attr(np, 'raw_today') or []) + (state_attr(np, 'raw_tomorrow') or []) %}

{% set window_start = today_at(states('input_datetime.ev_day_window_start')) %}
{% set window_end = today_at(states('input_datetime.ev_day_window_end')) %}

{% if now() >= window_end %}
  {% set window_start = window_start + timedelta(days=1) %}
  {% set window_end = window_end + timedelta(days=1) %}
{% endif %}

{% set start_ts = [as_timestamp(window_start), as_timestamp(now())] | max %}
{% set end_ts = as_timestamp(window_end) %}

{% set ns = namespace(slots=[]) %}

{% for item in all_prices %}
  {% set dt = as_datetime(item['start']) %}
  {% set ts = as_timestamp(dt) | int %}
  {% if ts >= start_ts and ts < end_ts %}
    {% set ns.slots = ns.slots + [(ts, item['value'] | float)] %}
  {% endif %}
{% endfor %}

{% set slots = ns.slots | sort(attribute=0) %}
{% set best = namespace(cost=999999, found=false) %}

{% for i in range(0, (slots | count) - needed_slots + 1) %}
  {% set block = slots[i:i + needed_slots] %}
  {% set expected_end = block[0][0] + ((needed_slots - 1) * 900) %}

  {% if block[-1][0] == expected_end %}
    {% set sum = namespace(value=0) %}
    {% for slot in block %}
      {% set sum.value = sum.value + (slot[1] * power * 0.25) %}
    {% endfor %}

    {% if sum.value < best.cost %}
      {% set best.cost = sum.value %}
      {% set best.found = true %}
    {% endif %}
  {% endif %}
{% endfor %}

{% if best.found %}
  {{ best.cost | round(2) }}
{% else %}
  unknown
{% endif %}
```

The stored unit is `SEK`.

#### `sensor.ev_night_estimated_charge_cost`

- **Purpose:** Calculates the cost of the lowest-price continuous future block in the relevant night window using night target and night current.
- **Dependencies:** night target/current/window helpers, Nord Pool raw today/tomorrow, current time.

```jinja
{% set _ = now() %}
{% set energy = states('input_number.ev_night_target_kwh') | float(0) %}
{% set amps = states('input_number.ev_night_charge_current') | float(10) %}
{% set power = [((amps - 1) * 230 / 1000), 0.1] | max %}
{% set needed_slots = ((energy / (power * 0.25)) | round(0, 'ceil')) | int %}

{% set np = 'sensor.nordpool_kwh_se3_sek_3_10_025' %}
{% set all_prices = (state_attr(np, 'raw_today') or []) + (state_attr(np, 'raw_tomorrow') or []) %}

{% set window_start = today_at(states('input_datetime.ev_night_window_start')) %}
{% set window_end = today_at(states('input_datetime.ev_night_window_end')) %}

{% if now() >= window_end %}
  {% set window_start = window_start + timedelta(days=1) %}
  {% set window_end = window_end + timedelta(days=1) %}
{% endif %}

{% set start_ts = [as_timestamp(window_start), as_timestamp(now())] | max %}
{% set end_ts = as_timestamp(window_end) %}

{% set ns = namespace(slots=[]) %}

{% for item in all_prices %}
  {% set dt = as_datetime(item['start']) %}
  {% set ts = as_timestamp(dt) | int %}
  {% if ts >= start_ts and ts < end_ts %}
    {% set ns.slots = ns.slots + [(ts, item['value'] | float)] %}
  {% endif %}
{% endfor %}

{% set slots = ns.slots | sort(attribute=0) %}
{% set best = namespace(cost=999999, found=false) %}

{% for i in range(0, (slots | count) - needed_slots + 1) %}
  {% set block = slots[i:i + needed_slots] %}
  {% set expected_end = block[0][0] + ((needed_slots - 1) * 900) %}

  {% if block[-1][0] == expected_end %}
    {% set sum = namespace(value=0) %}
    {% for slot in block %}
      {% set sum.value = sum.value + (slot[1] * power * 0.25) %}
    {% endfor %}

    {% if sum.value < best.cost %}
      {% set best.cost = sum.value %}
      {% set best.found = true %}
    {% endif %}
  {% endif %}
{% endfor %}

{% if best.found %}
  {{ best.cost | round(2) }}
{% else %}
  unknown
{% endif %}
```

The stored unit is `SEK`.

### 4.3 Other enabled storage-backed templates

All entries below have `disabled_by: null`. No active automation or EV Energy dashboard reference was found for them unless stated otherwise.

#### `binary_sensor.ev_is_charging`

```jinja
{{ states('sensor.ev_actual_current') | float > 0 }}
```

Dependency: `sensor.ev_actual_current`.

#### `sensor.ev_charger_status`

```jinja
{{ states('sensor.charger_connector_1_status_connector') }}
```

Dependency: OCPP connector 1 status.

#### `sensor.ev_day_charge_visual`

```jinja
{% if is_state('binary_sensor.ev_day_charge_recommended_now', 'on') %}
  2.5
{% else %}
  0
{% endif %}
```

#### `sensor.ev_night_charge_visual`

```jinja
{% if is_state('binary_sensor.ev_night_charge_recommended_now', 'on') %}
  2.2
{% else %}
  0
{% endif %}
```

#### `sensor.ev_day_visual_block`

The stored entry title and name are both `sensor.ev_day_visual_block`; the registered entity ID is `sensor.ev_day_visual_block`.

```jinja
{% set block = states('input_text.ev_day_selected_charge_block') %}

{% if '–' not in block %}
  0
{% else %}
  {% set parts = block.split('–') %}
  {% set start = today_at(parts[0]) + timedelta(days=1) %}
  {% set end = today_at(parts[1]) + timedelta(days=1) %}

  {% if now() >= start and now() < end %}
    2.5
  {% else %}
    0
  {% endif %}
{% endif %}
```

#### `sensor.ev_night_visual_block`

```jinja
{% set block = states('input_text.ev_night_selected_charge_block') %}

{% if '–' not in block %}
  0
{% else %}
  {% set parts = block.split('–') %}
  {% set start = today_at(parts[0]) + timedelta(days=1) %}
  {% set end = today_at(parts[1]) + timedelta(days=1) %}

  {% if end <= start %}
    {% set end = end + timedelta(days=1) %}
  {% endif %}

  {% if now() >= start and now() < end %}
    2.2
  {% else %}
    0
  {% endif %}
{% endif %}
```

### 4.4 Historical general-planner template entries

The Phase 2 report classifies the older general planning entities as historical artifacts. Their config entries remain enabled (`disabled_by: null`) and are reproduced here.

#### `sensor.ev_requiered_charging_hours`

```jinja
{% set energy = states('input_number.ev_required_energy') | float(0) %}
{% set power = states('input_number.ev_charger_power') | float(1) %}

{{ (energy / power * 0.25) | round(1) }}
```

Stored unit: `h`.

#### `sensor.ev_cheapest_charge_slots`

This entity comes from the storage entry titled `ev_cheapest_charging_hours` and depends on `sensor.electricity_prices_today` and `sensor.electricity_prices_tomorrow`. Those two entity IDs are absent from the exported entity registry.

```jinja
{% set energy = states('input_number.ev_required_energy') | float(0) %}
{% set power = states('input_number.ev_charger_power') | float(1) %}

{# Antal 15-minutersblock #}
{% set needed_slots =
    ((energy / (power * 0.25)) | round(0, 'ceil')) | int
%}

{% set today =
    state_attr('sensor.electricity_prices_today', 'data')
    or []
%}

{% set tomorrow =
    state_attr('sensor.electricity_prices_tomorrow', 'data')
    or []
%}

{% set all_prices = today + tomorrow %}

{% set ns = namespace(slots=[]) %}

{% for item in all_prices %}

  {% set dt = as_datetime(item.start) %}
  {% set hour = dt.hour %}

  {% if hour >= 22 or hour < 7 %}

    {% set ns.slots = ns.slots + [ {
      'time': dt.strftime('%H:%M'),
      'price': item.value | float
    } ] %}

  {% endif %}

{% endfor %}

{% set cheapest =
    ns.slots
    | sort(attribute='price')
%}

{% set selected = cheapest[:needed_slots] %}

{% set result = [] %}

{% for slot in selected | sort(attribute='time') %}

  {% set result = result + [
    slot.time
  ] %}

{% endfor %}

{{ result | join(', ') }}
```

#### `sensor.ev_cheapest_charge_slots_2`

This entity comes from the storage entry titled `ev_cheapest_charge_slots` and uses the active Nord Pool sensor, but only `raw_today` and the hard-coded hour rule `>= 22 or < 7`.

```jinja
{% set _ = now() %}
{% set energy = states('input_number.ev_required_energy') | float(0) %}
{% set power = states('input_number.ev_charger_power') | float(1) %}
{% set needed_slots = ((energy / (power * 0.25)) | round(0, 'ceil')) | int %}

{% set np = 'sensor.nordpool_kwh_se3_sek_3_10_025' %}
{% set all_prices = state_attr(np, 'raw_today') or [] %}

{% set ns = namespace(slots=[]) %}

{% for item in all_prices %}
  {% set dt = as_datetime(item['start']) %}

  {% if dt.hour >= 22 or dt.hour < 7 %}
    {% set ns.slots = ns.slots + [(
      dt.strftime('%H:%M'),
      item['value'] | float
    )] %}
  {% endif %}
{% endfor %}

{% set sorted_slots = ns.slots | sort(attribute=1) %}
{% set selected = sorted_slots[:needed_slots] | sort %}

{% set ns2 = namespace(times=[]) %}

{% for slot in selected %}
  {% set ns2.times = ns2.times + [slot[0]] %}
{% endfor %}

{{ ns2.times | join(', ') }}
```

#### `sensor.ev_estimated_charge_cost`

```jinja
{% set _ = now() %}
{% set energy = states('input_number.ev_required_energy') | float(0) %}
{% set power = states('input_number.ev_charger_power') | float(1) %}
{% set needed_slots = ((energy / (power * 0.25)) | round(0, 'ceil')) | int %}

{% set np = 'sensor.nordpool_kwh_se3_sek_3_10_025' %}
{% set all_prices = state_attr(np, 'raw_today') or [] %}

{% set ns = namespace(slots=[]) %}

{% for item in all_prices %}
  {% set dt = as_datetime(item['start']) %}

  {% if dt.hour >= 1 or dt.hour < 7 %}
    {% set ns.slots = ns.slots + [(
      dt.strftime('%H:%M'),
      item['value'] | float
    )] %}
  {% endif %}
{% endfor %}

{% set selected = (ns.slots | sort(attribute=1))[:needed_slots] %}

{% set ns2 = namespace(cost=0) %}

{% for slot in selected %}
  {% set ns2.cost = ns2.cost + (slot[1] * power * 0.25) %}
{% endfor %}

{{ ns2.cost | round(2) }}
```

Stored unit: `sek`.

#### `sensor.ev_cheapest_continuous_charge_block`

```jinja
{% set _ = now() %}
{% set energy = states('input_number.ev_required_energy') | float(0) %}
{% set power = states('input_number.ev_charger_power') | float(1) %}
{% set needed_slots = ((energy / (power * 0.25)) | round(0, 'ceil')) | int %}

{% set np = 'sensor.nordpool_kwh_se3_sek_3_10_025' %}
{% set all_prices = state_attr(np, 'raw_today') or [] %}

{% set window_start = today_at('09:00') %}
{% set window_end = today_at('22:00') %}

{% set now_ts = as_timestamp(now()) %}
{% set start_ts = [as_timestamp(window_start), now_ts] | max %}
{% set end_ts = as_timestamp(window_end) %}

{% set ns = namespace(slots=[], seen=[]) %}

{% for item in all_prices %}
  {% set dt = as_datetime(item['start']) %}
  {% set ts = as_timestamp(dt) | int %}

  {% if ts >= start_ts and ts < end_ts and ts not in ns.seen %}
    {% set ns.seen = ns.seen + [ts] %}
    {% set ns.slots = ns.slots + [(
      dt.strftime('%H:%M'),
      ts,
      item['value'] | float
    )] %}
  {% endif %}
{% endfor %}

{% set slots = ns.slots | sort(attribute=1) %}
{% set best = namespace(cost=999999, start='', end='', found=false) %}

{% for i in range(0, (slots | count) - needed_slots + 1) %}
  {% set block = slots[i:i + needed_slots] %}
  {% set expected_end = block[0][1] + ((needed_slots - 1) * 900) %}

  {% if block[-1][1] == expected_end %}
    {% set sum = namespace(value=0) %}

    {% for slot in block %}
      {% set sum.value = sum.value + slot[2] %}
    {% endfor %}

    {% if sum.value < best.cost %}
      {% set best.cost = sum.value %}
      {% set best.start = block[0][0] %}
      {% set best.end = (block[-1][1] + 900) | timestamp_custom('%H:%M', true) %}
      {% set best.found = true %}
    {% endif %}
  {% endif %}
{% endfor %}

{% if best.found %}
  {{ best.start }}–{{ best.end }}
{% else %}
  No future block found / {{ slots | count }} slots available / need {{ needed_slots }}
{% endif %}
```

## 5. Automations

### 5.1 Runtime status and post-export additions

| Entity ID | Alias | Restored state | Last triggered |
|---|---|---:|---|
| `automation.ev_build_charge_plan` | EV Build Charge Plan | `on` | `2026-07-19T21:00:00.056634+00:00` |
| `automation.ev_charge_start` | EV Charge Start | `on` | `2026-07-20T10:30:00.386105+00:00` |
| `automation.ev_charge_stop` | EV Charge Stop | `on` | `2026-07-20T04:15:00.276969+00:00` |
| `automation.ev_restore_defaults_after_day_charging_window` | EV Restore Defaults After Day Charging Window | Configured `on`; automatic trigger not yet runtime-verified | Not yet observed |
| `automation.set_halo_charge_current` | Set Halo charge current | `on` | `2026-07-19T08:52:19.362518+00:00` |
| `automation.ev_day_lock_selected_charge_block` | EV Day Lock Selected Charge Block | `off` | `2026-05-21T18:46:43.352449+00:00` |
| `automation.ev_night_lock_selected_charge_block` | EV Night Lock Selected Charge Block | `off` | `2026-05-21T23:45:00.140980+00:00` |
| `automation.ev_planned_charging_test` | EV Planned Charging Test | `off` | `2026-05-20T22:36:29.075859+00:00` |
| `automation.nattladdning_pa` | EV_Nattladdning_på | `off` | `2026-05-20T22:30:00.453602+00:00` |
| `automation.nattladdning_av` | EV_Nattladdning_av | `off` | `2026-05-21T04:00:00.436002+00:00` |

Except for `EV Restore Defaults After Day Charging Window`, the enabled/disabled classification in this section is taken from `.storage/core.restore_state`; the YAML itself contains no `enabled: false` markers. The restore automation was added after the export and is documented from the applied configuration.

### 5.2 `Set Halo charge current`

- **Purpose:** Sends the current value of `input_number.ev_max_charge_current` to OCPP.
- **Trigger:** Any state change of the maximum-current helper.
- **Conditions:** None.
- **Called service:** `ocpp.set_charge_rate` with `devid: charger` and integer `limit_amps`.
- **Reads:** `input_number.ev_max_charge_current`.
- **Writes/external effect:** OCPP charge-rate command.
- **Dependencies:** OCPP service registration and charger identifier `charger`.
- **Implementation location:** `automations.yaml`.

```yaml
- alias: Set Halo charge current
  trigger:
  - platform: state
    entity_id: input_number.ev_max_charge_current
  action:
  - service: ocpp.set_charge_rate
    data:
      devid: charger
      limit_amps: '{{ states(''input_number.ev_max_charge_current'') | int }}'
  mode: restart
  id: 24b0f6ae3c934c04ba914ff787766759
```

### 5.3 `EV Charge Start`

- **Purpose:** Selects day or night current and makes the charger available when the combined recommendation turns on.
- **Trigger:** `binary_sensor.ev_charge_recommended_now` changes to `on`.
- **Condition:** `input_boolean.ev_smart_charging_enabled` is `on`.
- **Called services/actions:** `input_number.set_value`, five-second delay, `switch.turn_on`.
- **Reads:** combined/day/night recommendation sensors, day/night current helpers.
- **Writes:** `input_number.ev_max_charge_current`, `switch.ev_charger_available`.
- **Dependencies:** `Set Halo charge current` reacts to the maximum-current helper change.
- **Implementation location:** `automations.yaml`.

```yaml
- id: '1779385896248'
  alias: EV Charge Start
  description: ''
  triggers:
  - entity_id: binary_sensor.ev_charge_recommended_now
    to: 'on'
    trigger: state
  conditions:
  - condition: state
    entity_id: input_boolean.ev_smart_charging_enabled
    state: 'on'
  actions:
  - choose:
    - conditions:
      - condition: state
        entity_id: binary_sensor.ev_day_charge_recommended_now
        state: 'on'
      sequence:
      - target:
          entity_id: input_number.ev_max_charge_current
        data:
          value: '{{ states(''input_number.ev_day_charge_current'') | int }}'
        action: input_number.set_value
    - conditions:
      - condition: state
        entity_id: binary_sensor.ev_night_charge_recommended_now
        state: 'on'
      sequence:
      - target:
          entity_id: input_number.ev_max_charge_current
        data:
          value: '{{ states(''input_number.ev_night_charge_current'') | int }}'
        action: input_number.set_value
  - delay: 00:00:05
  - target:
      entity_id: switch.ev_charger_available
    action: switch.turn_on
  mode: single
```

### 5.4 `EV Charge Stop`

- **Purpose:** Stops charging at recommendation end or when the day energy comparison becomes true, then resets the maximum current. Profile-value restoration is no longer part of this automation.
- **Triggers:** Combined recommendation changes to `off`; template comparing day recommendation, session energy, and day target.
- **Conditions:** Smart charging enabled and EV charger availability on.
- **Called services/actions:** `switch.turn_off`, one-minute delay, `input_number.set_value`.
- **Reads:** recommendation sensors, `sensor.ev_session_energy`, default night-current helper, availability state.
- **Writes:** EV charge-control and availability switches; maximum-current helper.
- **Unresolved dependency:** `sensor.ev_session_energy` is not registered in the exported entity registry; the session-energy template is registered as `sensor.sensor_ev_session_energy`.
- **Implementation location:** `automations.yaml`.

```yaml
- id: '1779386393138'
  alias: EV Charge Stop
  description: ''
  triggers:
  - entity_id: binary_sensor.ev_charge_recommended_now
    to: 'off'
    trigger: state
  - value_template: "{{\n  is_state('binary_sensor.ev_day_charge_recommended_now',
      'on')\n  and\n  states('sensor.ev_session_energy') | float(0)\n    >=\n  states('input_number.ev_day_target_kwh')
      | float(0)\n}}\n"
    trigger: template
  conditions:
  - condition: state
    entity_id: input_boolean.ev_smart_charging_enabled
    state: 'on'
  - condition: state
    entity_id: switch.ev_charger_available
    state: 'on'
  actions:
  - target:
      entity_id: switch.ev_charge_control
    action: switch.turn_off
  - delay: 00:01:00
  - target:
      entity_id: switch.ev_charger_available
    action: switch.turn_off
  - target:
      entity_id: input_number.ev_max_charge_current
    data:
      value: '{{ states(''input_number.ev_default_night_charge_current'') | int }}'
    action: input_number.set_value
  mode: single
```

### 5.5 `EV Restore Defaults After Day Charging Window`

- **Purpose:** Invokes the shared restore script when the selected day charging block ends.
- **Trigger:** `binary_sensor.ev_day_charge_recommended_now` changes from `on` to `off`.
- **Conditions:** None.
- **Called action:** `script.turn_on` for `script.ev_restore_default_values`.
- **Reads:** Day recommendation state.
- **Writes:** No helpers directly; all helper writes are owned by the script.
- **Runtime verification:** The automation is configured, but an automatic end-of-window execution has not yet been observed. The called script has been manually verified by the user.
- **Implementation location:** `automations.yaml`.

```yaml
- alias: EV Restore Defaults After Day Charging Window
  description: Återställer EV-inställningarna när det planerade dagladdningsfönstret avslutas.
  triggers:
  - trigger: state
    entity_id:
    - binary_sensor.ev_day_charge_recommended_now
    from: 'on'
    to: 'off'
  conditions: []
  actions:
  - action: script.turn_on
    target:
      entity_id: script.ev_restore_default_values
  mode: single
```

### 5.6 `EV Build Charge Plan`

- **Purpose:** Builds separate lowest-summed-price continuous 15-minute blocks for the next day's configured night and day windows.
- **Trigger:** Daily at `23:00:00`.
- **Condition:** Tomorrow prices valid and the stored day block absent or already ended according to the template.
- **Guard action:** Aborts with a persistent notification unless tomorrow is valid and contains at least 96 raw price entries.
- **Inputs:** day/night targets, day/night currents, day/night windows, `raw_tomorrow`, `tomorrow_valid`, existing selected day block.
- **Outputs:** selected night block, selected day block, plan-last-updated timestamp.
- **Calculation facts:** effective power is `max((amps - 1) * 230 / 1000, 0.1)`; slot length is 0.25 h; consecutive slots are validated at 900-second spacing; candidate comparison sums price values; output is `HH:MM–HH:MM` or `none`.
- **Called services/actions:** `persistent_notification.create`, `input_text.set_value`, `input_datetime.set_datetime`, `stop`.
- **Implementation location:** `automations.yaml`.

```yaml
- id: '1779432172940'
  alias: EV Build Charge Plan
  description: ''
  triggers:
  - at: '23:00:00'
    trigger: time
  conditions:
  - condition: template
    value_template: "{% set tomorrow_valid =\n   state_attr('sensor.nordpool_kwh_se3_sek_3_10_025',\n
      \             'tomorrow_valid') == true %}\n\n{% set block =\n   states('input_text.ev_day_selected_charge_block')
      %}\n\n{% if not tomorrow_valid %}\n  false\n\n{% elif '–' not in block %}\n
      \ true\n\n{% else %}\n  {% set end_time = block.split('–')[1] %}\n  {% set day_end
      = today_at(end_time) %}\n\n  {{ now() >= day_end }}\n{% endif %}\n"
  actions:
  - if:
    - condition: template
      value_template: '{% set np = ''sensor.nordpool_kwh_se3_sek_3_10_025'' %} {%
        set tomorrow_valid = state_attr(np, ''tomorrow_valid'') | string | lower %}
        {% set tomorrow_count = (state_attr(np, ''raw_tomorrow'') or []) | count %}
        {{ not (tomorrow_valid == ''true'' and tomorrow_count >= 96) }}'
    then:
    - action: persistent_notification.create
      data:
        title: EV Build Plan
        message: ABORTED - Tomorrow prices not available
    - stop: Tomorrow prices are not available
  - target:
      entity_id: input_text.ev_night_selected_charge_block
    data:
      value: "{% set energy = states('input_number.ev_night_target_kwh') | float(0)
        %} {% set amps = states('input_number.ev_night_charge_current') | float(10)
        %} {% set power = [((amps - 1) * 230 / 1000), 0.1] | max %} {% set needed_slots
        = ((energy / (power * 0.25)) | round(0, 'ceil')) | int %}\n{% set np = 'sensor.nordpool_kwh_se3_sek_3_10_025'
        %} {% set all_prices = state_attr(np, 'raw_tomorrow') or [] %}\n{% set window_start
        = states('input_datetime.ev_night_window_start') %} {% set window_end = states('input_datetime.ev_night_window_end')
        %}\n{% set ns = namespace(slots=[], seen=[]) %}\n{% for item in all_prices
        %}\n  {% set dt = as_datetime(item['start']) %}\n  {% set ts = as_timestamp(dt)
        | int %}\n  {% set t = dt.strftime('%H:%M:%S') %}\n\n  {% if t >= window_start
        and t < window_end and ts not in ns.seen %}\n    {% set ns.seen = ns.seen
        + [ts] %}\n    {% set ns.slots = ns.slots + [(\n      dt.strftime('%H:%M'),\n
        \     ts,\n      item['value'] | float\n    )] %}\n  {% endif %}\n{% endfor
        %}\n{% set slots = ns.slots | sort(attribute=1) %} {% set best = namespace(cost=999999,
        start='', end='', found=false) %}\n{% for i in range(0, (slots | count) -
        needed_slots + 1) %}\n  {% set block = slots[i:i + needed_slots] %}\n  {%
        set expected_end = block[0][1] + ((needed_slots - 1) * 900) %}\n\n  {% if
        block[-1][1] == expected_end %}\n    {% set sum = namespace(value=0) %}\n\n
        \   {% for slot in block %}\n      {% set sum.value = sum.value + slot[2]
        %}\n    {% endfor %}\n\n    {% if sum.value < best.cost %}\n      {% set best.cost
        = sum.value %}\n      {% set best.start = block[0][0] %}\n      {% set best.end
        = (block[-1][1] + 900) | timestamp_custom('%H:%M', true) %}\n      {% set
        best.found = true %}\n    {% endif %}\n  {% endif %}\n{% endfor %}\n{% if
        best.found %}\n  {{ best.start }}–{{ best.end }}\n{% else %}\n  none\n{% endif
        %}\n"
    action: input_text.set_value
  - target:
      entity_id: input_text.ev_day_selected_charge_block
    data:
      value: "{% set energy = states('input_number.ev_day_target_kwh') | float(0)
        %} {% set amps = states('input_number.ev_day_charge_current') | float(6) %}
        {% set power = [((amps - 1) * 230 / 1000), 0.1] | max %}  {% set needed_slots
        = ((energy / (power * 0.25)) | round(0, 'ceil')) | int %}\n{% set np = 'sensor.nordpool_kwh_se3_sek_3_10_025'
        %} {% set all_prices = state_attr(np, 'raw_tomorrow') or [] %}\n{% set window_start
        = states('input_datetime.ev_day_window_start') %} {% set window_end = states('input_datetime.ev_day_window_end')
        %}\n{% set ns = namespace(slots=[], seen=[]) %}\n{% for item in all_prices
        %}\n  {% set dt = as_datetime(item['start']) %}\n  {% set ts = as_timestamp(dt)
        | int %}\n  {% set t = dt.strftime('%H:%M:%S') %}\n\n  {% if t >= window_start
        and t < window_end and ts not in ns.seen %}\n    {% set ns.seen = ns.seen
        + [ts] %}\n    {% set ns.slots = ns.slots + [(\n      dt.strftime('%H:%M'),\n
        \     ts,\n      item['value'] | float\n    )] %}\n  {% endif %}\n{% endfor
        %}\n{% set slots = ns.slots | sort(attribute=1) %} {% set best = namespace(cost=999999,
        start='', end='', found=false) %}\n{% for i in range(0, (slots | count) -
        needed_slots + 1) %}\n  {% set block = slots[i:i + needed_slots] %}\n  {%
        set expected_end = block[0][1] + ((needed_slots - 1) * 900) %}\n\n  {% if
        block[-1][1] == expected_end %}\n    {% set sum = namespace(value=0) %}\n\n
        \   {% for slot in block %}\n      {% set sum.value = sum.value + slot[2]
        %}\n    {% endfor %}\n\n    {% if sum.value < best.cost %}\n      {% set best.cost
        = sum.value %}\n      {% set best.start = block[0][0] %}\n      {% set best.end
        = (block[-1][1] + 900) | timestamp_custom('%H:%M', true) %}\n      {% set
        best.found = true %}\n    {% endif %}\n  {% endif %}\n{% endfor %}\n{% if
        best.found %}\n  {{ best.start }}–{{ best.end }}\n{% else %}\n  none\n{% endif
        %}\n"
    action: input_text.set_value
  - target:
      entity_id: input_datetime.ev_plan_last_updated
    data:
      datetime: '{{ now().strftime(''%Y-%m-%d %H:%M:%S'') }}'
    action: input_datetime.set_datetime
  mode: single
```

### 5.7 Configured but restored-off EV automations

The following automation entities exist and their YAML remains in `automations.yaml`, but `.storage/core.restore_state` records them as `off`.

#### `EV_Nattladdning_på`

- **Entity ID:** `automation.nattladdning_pa`.
- **Behaviour:** Fixed daily availability-on action at 00:30.

```yaml
- id: '1778674022926'
  alias: EV_Nattladdning_på
  description: ''
  triggers:
  - trigger: time
    at: 00:30:00
    weekday:
    - mon
    - tue
    - wed
    - thu
    - fri
    - sat
    - sun
  conditions: []
  actions:
  - action: switch.turn_on
    metadata: {}
    target:
      entity_id: switch.ev_charger_available
    data: {}
  mode: single
```

#### `EV_Nattladdning_av`

- **Entity ID:** `automation.nattladdning_av`.
- **Behaviour:** Fixed daily charge-control-off, one-minute delay, availability-off action at 06:00.

```yaml
- id: '1778674154151'
  alias: EV_Nattladdning_av
  description: ''
  triggers:
  - trigger: time
    at: 06:00:00
    weekday:
    - mon
    - tue
    - wed
    - thu
    - fri
    - sat
    - sun
  conditions: []
  actions:
  - action: switch.turn_off
    metadata: {}
    data: {}
    target:
      entity_id: switch.ev_charge_control
  - delay:
      hours: 0
      minutes: 1
      seconds: 0
      milliseconds: 0
  - action: switch.turn_off
    metadata: {}
    data: {}
    target:
      entity_id: switch.ev_charger_available
  mode: single
```

#### `EV Planned Charging Test`

- **Entity ID:** `automation.ev_planned_charging_test`.
- **Dependency:** `binary_sensor.ev_charging_planned_active`, which remains in the entity registry as a YAML-template historical entity but has no current definition in `configuration.yaml`.

```yaml
- id: '1779312784510'
  alias: EV Planned Charging Test
  description: ''
  triggers:
  - entity_id: binary_sensor.ev_charging_planned_active
    trigger: state
  actions:
  - data:
      name: EV Planner
      message: 'Charging planned state changed to {{ states(''binary_sensor.ev_charging_planned_active'')
        }}

        '
    action: logbook.log
  mode: single
```

#### `EV Day Lock Selected Charge Block`

- **Entity ID:** `automation.ev_day_lock_selected_charge_block`.
- **Dependency:** `sensor.ev_day_cheapest_continuous_charge_block`, absent from the exported entity registry and template config entries.

```yaml
- id: '1779368998858'
  alias: EV Day Lock Selected Charge Block
  triggers:
  - entity_id: sensor.ev_day_cheapest_continuous_charge_block
    trigger: state
  - event: start
    trigger: homeassistant
  conditions:
  - condition: template
    value_template: "{% set new_block = states('sensor.ev_day_cheapest_continuous_charge_block')
      %} {% set selected = states('input_text.ev_day_selected_charge_block') %}\n{%
      if '–' not in new_block %}\n  false\n{% elif '–' not in selected %}\n  true\n{%
      else %}\n  {% set parts = selected.split('–') %}\n  {% set selected_end = today_at(parts[1])
      %}\n  {{ now() >= selected_end }}\n{% endif %}\n"
  actions:
  - target:
      entity_id: input_text.ev_day_selected_charge_block
    data:
      value: '{{ states(''sensor.ev_day_cheapest_continuous_charge_block'') }}'
    action: input_text.set_value
  mode: single
```

#### `EV Night Lock Selected Charge Block`

- **Entity ID:** `automation.ev_night_lock_selected_charge_block`.
- **Dependency:** `sensor.ev_night_cheapest_continuous_charge_block`, absent from the exported entity registry and template config entries.

```yaml
- id: '1779369289452'
  alias: EV Night Lock Selected Charge Block
  description: ''
  triggers:
  - entity_id: sensor.ev_night_cheapest_continuous_charge_block
    trigger: state
  - event: start
    trigger: homeassistant
  conditions:
  - condition: template
    value_template: '{{ ''–'' in states(''sensor.ev_night_cheapest_continuous_charge_block'')
      }}

      '
  actions:
  - target:
      entity_id: input_text.ev_night_selected_charge_block
    data:
      value: '{{ states(''sensor.ev_night_cheapest_continuous_charge_block'') }}'
    action: input_text.set_value
  mode: single
```

### 5.8 Top-level fail-safe block in `configuration.yaml`

This code is present exactly as follows. It is not under the `automation:` include and no entity registration or restored automation state for this alias exists. Executability is **Not verified from implementation.** It also waits on `sensor.ev_status`, which is absent from the exported entity registry.

```yaml
alias: EV Charger Fail Safe On Startup
trigger:
  - platform: homeassistant
    event: start
action:
  - delay: "00:02:30"
    
  - service: switch.turn_off
    target:
       entity_id: switch.ev_charge_control
    
  - wait_template: >
      {{ states('sensor.ev_status') != 'Charging' }}
    timeout: "00:00:30"
    
  - service: switch.turn_off
    target:
       entity_id: switch.ev_charger_available
    
mode: single
```

## 6. Scripts

`scripts.yaml` contains the EV-specific restore script plus two unrelated lighting scripts.

### 6.1 `EV Restore Default Values`

- **Entity ID:** `script.ev_restore_default_values`.
- **Purpose:** Copies the four configured default profile values into the active day/night target and current helpers.
- **Called by:** `EV Restore Defaults After Day Charging Window` and the restore button on the EV Energy Settings view.
- **Reads:** The four `input_number.ev_default_*` helpers.
- **Writes:** `input_number.ev_day_target_kwh`, `input_number.ev_night_target_kwh`, `input_number.ev_day_charge_current`, and `input_number.ev_night_charge_current`.
- **Explicit exclusion:** The script does not read or write `input_number.ev_max_charge_current`.
- **Runtime verification:** Manual execution was verified successfully by the user on 2026-07-21.

```yaml
ev_restore_default_values:
  alias: EV Restore Default Values
  description: Återställer dag- och nattvärden från konfigurerade standardvärden.
  sequence:
  - action: input_number.set_value
    target:
      entity_id: input_number.ev_day_target_kwh
    data:
      value: "{{ states('input_number.ev_default_day_target_kwh') | float(0) }}"
  - action: input_number.set_value
    target:
      entity_id: input_number.ev_night_target_kwh
    data:
      value: "{{ states('input_number.ev_default_night_target_kwh') | float(0) }}"
  - action: input_number.set_value
    target:
      entity_id: input_number.ev_day_charge_current
    data:
      value: "{{ states('input_number.ev_default_day_charge_current') | float(0) }}"
  - action: input_number.set_value
    target:
      entity_id: input_number.ev_night_charge_current
    data:
      value: "{{ states('input_number.ev_default_night_charge_current') | float(0) }}"
  mode: single
```

### 6.2 Other scripts

The two pre-existing lighting scripts remain unrelated to EV charging:

```yaml
kvallsbelysning:
  sequence:
  - sequence:
    - action: automation.trigger
      metadata: {}
      target:
        entity_id: automation.kvallsbelysning
      data:
        skip_condition: true
  alias: Kvällsbelysning
  description: ''
godnatt:
  sequence:
  - sequence:
    - action: automation.trigger
      metadata: {}
      target:
        entity_id: automation.vardagskvall
      data:
        skip_condition: true
  alias: 'Godnatt '
  description: ''
```

## 7. Dashboard

### 7.1 Registration

The storage dashboard is registered in `.storage/lovelace_dashboards` as:

| Field | Value |
|---|---|
| ID | `ev_energy` |
| Title | `EV Energy` |
| URL path | `ev-energy` |
| Mode | `storage` |
| Sidebar | `true` |
| Admin required | `false` |
| Icon | `mdi:car-electric` |

The dashboard body is stored in `.storage/lovelace.ev_energy` and has two section-based views: `Status` and `Settings`.

### 7.2 Status view

#### Manual plan-build button

The first card is a button bound to `automation.ev_build_charge_plan`. Its tap action bypasses the automation's conditions by setting `skip_condition: true`.

```json
{
  "show_name": true,
  "show_icon": false,
  "show_state": false,
  "type": "button",
  "entity": "automation.ev_build_charge_plan",
  "name": "EV Build Charge Plan",
  "tap_action": {
    "action": "perform-action",
    "perform_action": "automation.trigger",
    "target": {"entity_id": "automation.ev_build_charge_plan"},
    "data": {"skip_condition": true}
  }
}
```

#### Status entities card

| Display name | Entity/action |
|---|---|
| Smart charging enabled | `input_boolean.ev_smart_charging_enabled` — editable toggle |
| Day target kWh | `input_number.ev_day_target_kwh` — editable helper |
| Dag planerat laddfönster | `input_text.ev_day_selected_charge_block` — editable helper |
| Day estimated cost | `sensor.ev_day_estimated_charge_cost` |
| Day charge now | `binary_sensor.ev_day_charge_recommended_now` |
| Night target kWh | `input_number.ev_night_target_kwh` — editable helper |
| Natt planerat laddfönster | `input_text.ev_night_selected_charge_block` — editable helper |
| Night estimated cost | `sensor.ev_night_estimated_charge_cost` |
| Night charge now | `binary_sensor.ev_night_charge_recommended_now` |
| Charge recommended now | `binary_sensor.ev_charge_recommended_now` |
| Charge control | `switch.ev_charge_control` — manual switch action available |
| Charger available | `switch.ev_charger_available` — manual switch action available |
| Tomorrow prices available | `binary_sensor.tomorrow_prices_available` |

#### ApexCharts card

The chart uses `custom:apexcharts-card`, a 48-hour span starting at day boundary, a current-time marker, and a column chart. Header-only series show the Nord Pool `min`, `current_price`, and `max` attributes. The plotted price series is generated from `raw_today` plus `raw_tomorrow` when `tomorrow_valid` is true. A hidden `input_datetime.ev_plan_last_updated` series provides an additional entity dependency.

The ApexCharts resource is registered as `/hacsfiles/apexcharts-card/apexcharts-card.js?hacstag=331701152223`, type `module`.

The exact plotted-series generator is:

```javascript
const rawToday = entity.attributes.raw_today || [];
const rawTomorrow = entity.attributes.tomorrow_valid
  ? (entity.attributes.raw_tomorrow || [])
  : [];

const dayBlock = hass.states["input_text.ev_day_selected_charge_block"]?.state || "";
const nightBlock = hass.states["input_text.ev_night_selected_charge_block"]?.state || "";

function getColor(v) {
  if (v <= 0.50) return '#00b050';
  if (v <= 1.00) return '#92d050';
  if (v <= 1.50) return '#ffd966';
  if (v <= 2.00) return '#f4b183';
  if (v <= 2.50) return '#ff6666';
  return '#cc0000';
}

function sameDay(a, b) {
  return a.getFullYear() === b.getFullYear()
      && a.getMonth() === b.getMonth()
      && a.getDate() === b.getDate();
}

function inBlock(date, block, targetDate) {
  if (!block || !block.includes("–")) return false;
  if (!sameDay(date, targetDate)) return false;

  const [startStr, endStr] = block.split("–");
  const [sh, sm] = startStr.split(":").map(Number);
  const [eh, em] = endStr.split(":").map(Number);

  const start = new Date(targetDate);
  start.setHours(sh, sm, 0, 0);

  const end = new Date(targetDate);
  end.setHours(eh, em, 0, 0);

  if (end <= start) {
    end.setDate(end.getDate() + 1);
  }

  return date >= start && date < end;
}

  const now = new Date();

  const todayTarget = new Date(now);
  todayTarget.setHours(12, 0, 0, 0);

  const tomorrowTarget = new Date(now);
  tomorrowTarget.setDate(tomorrowTarget.getDate() + 1);
  tomorrowTarget.setHours(12, 0, 0, 0);

  const tomorrowValid =
    entity.attributes.tomorrow_valid === true;

  const buildPlanState =
    hass.states["automation.ev_build_charge_plan"];

  let buildPlanRanToday = false;

  if (buildPlanState && buildPlanState.attributes.last_triggered) {
    const lastTriggered = new Date(buildPlanState.attributes.last_triggered);

    buildPlanRanToday =
      lastTriggered.getFullYear() === now.getFullYear() &&
      lastTriggered.getMonth() === now.getMonth() &&
      lastTriggered.getDate() === now.getDate();
  }

  const selectedBlocksAreForTomorrow =
    tomorrowValid && buildPlanRanToday;

  const dayTargetDate =
    selectedBlocksAreForTomorrow ? tomorrowTarget : todayTarget;

  const nightTargetDate =
    selectedBlocksAreForTomorrow ? tomorrowTarget : todayTarget;


return rawToday.concat(rawTomorrow).map((item) => {
  const d = new Date(item.start);
  let color = getColor(item.value);

  if (inBlock(d, nightBlock, nightTargetDate)) {
    color = '#008ffb';
  }

  if (inBlock(d, dayBlock, dayTargetDate)) {
    color = '#008ffb';
  }

  return {
    x: d.getTime(),
    y: item.value,
    fillColor: color
  };
});
  
```

### 7.3 Settings view

The settings entities card exposes these helpers for direct editing:

- `input_number.ev_default_day_target_kwh`
- `input_number.ev_default_night_target_kwh`
- `input_number.ev_default_day_charge_current`
- `input_number.ev_default_night_charge_current`
- `input_number.ev_day_charge_current`
- `input_number.ev_night_charge_current`
- `input_datetime.ev_day_window_start`
- `input_datetime.ev_day_window_end`
- `input_datetime.ev_night_window_start`
- `input_datetime.ev_night_window_end`

#### Restore-defaults button

The Settings view also contains a button that invokes the same shared restore script used by the end-of-day-block automation. It asks for confirmation before execution.

```yaml
type: button
name: Återställ standardvärden
icon: mdi:restore
show_name: true
show_icon: true
tap_action:
  action: perform-action
  perform_action: script.turn_on
  target:
    entity_id: script.ev_restore_default_values
confirmation:
  text: Vill du återställa dag- och nattvärdena till standardvärden?
```

## 8. OCPP Configuration

### 8.1 Installed integration

The installed custom integration manifest identifies:

| Field | Value |
|---|---|
| Domain | `ocpp` |
| Name | Open Charge Point Protocol (OCPP) |
| Version | `v0.10.15` |
| I/O class | `local_push` |
| Requirements | `ocpp>=2.1.0`, `websockets>=14.1` |

`configuration.yaml` enables debug logging for `custom_components.ocpp`, `ocpp`, `websockets`, and `homeassistant.components.ocpp`.

### 8.2 Exact config entry

The active OCPP config entry is titled `central` and contains:

```json
{
  "cpids": [
    {
      "1808002273M": {
        "cpid": "charger",
        "force_smart_charging": false,
        "idle_interval": 900,
        "max_current": 10,
        "meter_interval": 60,
        "monitored_variables": "Current.Export,Current.Import,Current.Offered,Energy.Active.Export.Interval,Energy.Active.Export.Register,Energy.Active.Import.Interval,Energy.Active.Import.Register,Energy.Reactive.Export.Interval,Energy.Reactive.Export.Register,Energy.Reactive.Import.Interval,Energy.Reactive.Import.Register,Frequency,Power.Active.Export,Power.Active.Import,Power.Factor,Power.Offered,Power.Reactive.Export,Power.Reactive.Import,RPM,SoC,Temperature,Voltage",
        "monitored_variables_autoconfig": true,
        "num_connectors": 1,
        "skip_schema_validation": false
      }
    }
  ],
  "csid": "central",
  "host": "<HOME_ASSISTANT_IP>",
  "port": 9000,
  "ssl": false,
  "ssl_certfile_path": "/config/fullchain.pem",
  "ssl_keyfile_path": "/config/privkey.pem",
  "websocket_close_timeout": 10,
  "websocket_ping_interval": 20,
  "websocket_ping_timeout": 20,
  "websocket_ping_tries": 2
}
```

### 8.3 Service used by the project

Only `ocpp.set_charge_rate` is called by the EV implementation. The custom integration defines that service as follows:

```yaml
# Service ID
set_charge_rate:
  # Service name as shown in UI
  name: Set maximum charge rate
  # Description of the service
  description: Sets the maximum charge rate in Amps or Watts (dependent on charger support)
  # If the service accepts entity IDs, target allows the user to specify entities by entity, device, or area. If `target` is specified, `entity_id` should not be defined in the `fields` map. By default it shows only targets matching entities from the same domain as the service, but if further customization is required, target supports the entity, device, and area selectors (https://www.home-assistant.io/docs/blueprint/selectors/). Entity selector parameters will automatically be applied to device and area, and device selector parameters will automatically be applied to area.
  #target:
  #   entity:
  #     integration: ocpp
  # Different fields that your service accepts
  fields:
    # Key of the field
    devid:
      name: Charger identifier
      description: Either HA charger id or Ocpp id
      required: false
      advanced: true
      example: charger
    limit_amps:
      # Field name as shown in UI
      name: Limit (A)
      # Description of the field
      description: Maximum charge rate in Amps (optional)
      # Whether or not field is required (default = false)
      required: false
      # Advanced fields are only shown when the advanced mode is enabled for the user (default = false)
      advanced: false
      # Example value that can be passed for this field
      example: 16
      # The default field value
      default: 32
    limit_watts:
      name: Limit (W)
      description: Maximum charge rate in Watts (optional)
      required: false
      advanced: true
      example: 1500
      default: 22000
    conn_id:
      name: Connector identifier
      description: Optional, 0 = all connectors (default), 1 is first connector
      required: false
      advanced: true
      example: 0
      default: 0
    custom_profile:
      name: Custom profile
      description: Used to send a custom charge profile to charger (for advanced users only use >- or '' to ensure profile is a string variable)
      required: false
      advanced: true
      example: '{"chargingProfileId":8,"stackLevel":0,"chargingProfileKind":"Relative","chargingProfilePurpose":"ChargePointMaxProfile","chargingSchedule":{"chargingRateUnit":"A","chargingSchedulePeriod":[{"startPeriod":0,"limit":16}]}}'
```

The project call supplies `devid: charger` and `limit_amps`; it does not supply `conn_id`, `limit_watts`, or `custom_profile`.

The installed integration also registers `ocpp.trigger_custom_message`, `ocpp.clear_profile`, `ocpp.update_firmware`, `ocpp.configure`, `ocpp.get_configuration`, `ocpp.get_diagnostics`, and `ocpp.data_transfer`. No call to those services was found in the EV project implementation.

### 8.4 OCPP switches and control facades

| Entity | Registry name | Project usage |
|---|---|---|
| `switch.charger_availability` | Availability | Underlying switch for `switch.ev_charger_available`. |
| `switch.charger_charge_control` | Charge Control | Registered but not referenced by the EV project. |
| `switch.charger_connector_1_charge_control` | Charge Control | Underlying switch for `switch.ev_charge_control`. |
| `switch.charger_connector_1_connnector_availability` | Connector Availability | Registered but not referenced by the EV project. |
| `switch.charger_connector_2_charge_control` | Charge Control | Registered but not referenced by the EV project. |
| `switch.charger_connector_2_connnector_availability` | Connector Availability | Registered but not referenced by the EV project. |
| `switch.ev_charger_available` | Template facade | Written by start/stop and exposed on dashboard. |
| `switch.ev_charge_control` | Template facade | Written by stop and exposed on dashboard. |

The OCPP config entry currently states `num_connectors: 1`, while connector 2 entities remain in the entity registry. Their present charger-side validity is **Not verified from implementation.**

### 8.5 OCPP sensors used directly or indirectly

| Entity | Consumer |
|---|---|
| `sensor.charger_connector_1_current_import` | `sensor.halo_l1` reads its `L1` attribute. |
| `sensor.charger_connector_1_status_connector` | `sensor.ev_actual_current` and `sensor.ev_charger_status`. |
| `sensor.charger_connector_1_energy_session` | Storage template registered as `sensor.sensor_ev_session_energy`. |

The following additional OCPP entities are registered and not disabled in the entity registry. No direct reference from the current planner/start/stop/dashboard implementation was found unless listed above.

- **Buttons:** `button.charger_connector_1_unlock`, `button.charger_connector_2_unlock`, `button.charger_reset`, `button.charger_unlock`.
- **Current-limit numbers:** `number.charger_connector_1_maximum_current`, `number.charger_connector_2_maximum_current`, `number.charger_maximum_current`.
- **Connector 1 telemetry:** `sensor.charger_connector_1_energy_active_import_register`, `sensor.charger_connector_1_energy_meter_start`, `sensor.charger_connector_1_error_code_connector`, `sensor.charger_connector_1_stop_reason`, `sensor.charger_connector_1_time_session`, `sensor.charger_connector_1_transaction_id`, `sensor.charger_connector_1_voltage`.
- **Connector 2 telemetry:** `sensor.charger_connector_2_current_import`, `sensor.charger_connector_2_energy_active_import_register`, `sensor.charger_connector_2_energy_meter_start`, `sensor.charger_connector_2_energy_session`, `sensor.charger_connector_2_error_code_connector`, `sensor.charger_connector_2_status_connector`, `sensor.charger_connector_2_stop_reason`, `sensor.charger_connector_2_time_session`, `sensor.charger_connector_2_transaction_id`, `sensor.charger_connector_2_voltage`.
- **Charger telemetry and diagnostics:** `sensor.charger_connectors`, `sensor.charger_current_export`, `sensor.charger_current_import`, `sensor.charger_current_offered`, `sensor.charger_energy_active_export_interval`, `sensor.charger_energy_active_export_register`, `sensor.charger_energy_active_import_interval`, `sensor.charger_energy_active_import_register`, `sensor.charger_energy_meter_start`, `sensor.charger_energy_reactive_export_interval`, `sensor.charger_energy_reactive_export_register`, `sensor.charger_energy_reactive_import_interval`, `sensor.charger_energy_reactive_import_register`, `sensor.charger_energy_session`, `sensor.charger_error_code`, `sensor.charger_error_code_connector`, `sensor.charger_features`, `sensor.charger_frequency`, `sensor.charger_heartbeat`, `sensor.charger_id`, `sensor.charger_id_tag`, `sensor.charger_latency_ping`, `sensor.charger_latency_pong`, `sensor.charger_model`, `sensor.charger_power_active_export`, `sensor.charger_power_active_import`, `sensor.charger_power_factor`, `sensor.charger_power_offered`, `sensor.charger_power_reactive_export`, `sensor.charger_power_reactive_import`, `sensor.charger_reconnects`, `sensor.charger_rpm`, `sensor.charger_serial`, `sensor.charger_soc`, `sensor.charger_status`, `sensor.charger_status_connector`, `sensor.charger_status_firmware`, `sensor.charger_stop_reason`, `sensor.charger_temperature`, `sensor.charger_time_session`, `sensor.charger_timestamp_config_response`, `sensor.charger_timestamp_data_response`, `sensor.charger_timestamp_data_transfer`, `sensor.charger_transaction_id`, `sensor.charger_vendor`, `sensor.charger_version_firmware`, and `sensor.charger_voltage`.

## 9. Entity Dependency Reference

“Owner” in this table identifies the component that defines the entity. “Written by” records actual service writers found in the implementation; template entities are recalculated by Home Assistant.

| Entity | Owner/type | Written by | Read by | Purpose | Storage/definition |
|---|---|---|---|---|---|
| `sensor.nordpool_kwh_se3_sek_3_10_025` | Nord Pool sensor | Nord Pool integration | Plan builder, cost templates, tomorrow-price sensor, dashboard chart | SE3 price state and raw price attributes | `.storage/core.config_entries`; entity registry |
| `input_boolean.ev_smart_charging_enabled` | Input boolean | User/dashboard | Start and stop automations | Execution gate | `.storage/input_boolean` |
| `input_number.ev_day_target_kwh` | Input number | User/dashboard; restore script | Plan builder, day cost, stop energy logic | Day energy target | `.storage/input_number`; renamed in entity registry |
| `input_number.ev_night_target_kwh` | Input number | User/dashboard; restore script | Plan builder, night cost | Night energy target | `.storage/input_number`; renamed in entity registry |
| `input_number.ev_day_charge_current` | Input number | User/dashboard; restore script | Plan builder, start, day cost | Day current input | `.storage/input_number` |
| `input_number.ev_night_charge_current` | Input number | User/dashboard; restore script | Plan builder, start, night cost | Night current input | `.storage/input_number` |
| `input_number.ev_default_day_target_kwh` | Input number | User/dashboard | Restore script | Day target reset source | `.storage/input_number` |
| `input_number.ev_default_night_target_kwh` | Input number | User/dashboard | Restore script | Night target reset source | `.storage/input_number` |
| `input_number.ev_default_day_charge_current` | Input number | User/dashboard | Restore script | Day current reset source | `.storage/input_number` |
| `input_number.ev_default_night_charge_current` | Input number | User/dashboard | Restore script and stop maximum-current reset | Night current reset source and separate default maximum-current source | `.storage/input_number` |
| `input_number.ev_max_charge_current` | Input number | Start, stop, direct user edit | Set Halo charge current | Runtime OCPP current-command source | `.storage/input_number` |
| `input_datetime.ev_day_window_start` | Input datetime | User/dashboard | Plan builder, day cost | Day window start | `.storage/input_datetime` |
| `input_datetime.ev_day_window_end` | Input datetime | User/dashboard | Plan builder, day cost | Day window end | `.storage/input_datetime` |
| `input_datetime.ev_night_window_start` | Input datetime | User/dashboard | Plan builder, night cost | Night window start | `.storage/input_datetime` |
| `input_datetime.ev_night_window_end` | Input datetime | User/dashboard | Plan builder, night cost | Night window end | `.storage/input_datetime` |
| `input_text.ev_day_selected_charge_block` | Input text | Plan builder; disabled historical day lock | Day recommendation, dashboard, plan-build condition | Selected day block | `.storage/input_text` |
| `input_text.ev_night_selected_charge_block` | Input text | Plan builder; disabled historical night lock | Night recommendation, dashboard | Selected night block | `.storage/input_text` |
| `input_datetime.ev_plan_last_updated` | Input datetime | Plan builder | Dashboard hidden series | Plan update timestamp | `.storage/input_datetime` |
| `binary_sensor.ev_day_charge_recommended_now` | Template binary sensor | Home Assistant template evaluation | Combined recommendation, start, stop, restore automation, visual sensor, dashboard | Day active-now decision | `.storage/core.config_entries` |
| `binary_sensor.ev_night_charge_recommended_now` | Template binary sensor | Home Assistant template evaluation | Combined recommendation, start, visual sensor, dashboard | Night active-now decision | `.storage/core.config_entries` |
| `binary_sensor.ev_charge_recommended_now` | Template binary sensor | Home Assistant template evaluation | Start, stop, dashboard | Combined active-now decision | `.storage/core.config_entries` |
| `sensor.ev_day_estimated_charge_cost` | Template sensor | Home Assistant template evaluation | Dashboard | Day continuous-block cost | `.storage/core.config_entries` |
| `sensor.ev_night_estimated_charge_cost` | Template sensor | Home Assistant template evaluation | Dashboard | Night continuous-block cost | `.storage/core.config_entries` |
| `binary_sensor.tomorrow_prices_available` | YAML template binary sensor | Home Assistant template evaluation | Dashboard | Tomorrow data readiness display | `configuration.yaml` |
| `switch.ev_charger_available` | Template switch | Start, stop, dashboard; disabled fixed-time automations | Start/stop conditions, dashboard | EV-facing availability facade | `.storage/core.config_entries` |
| `switch.charger_availability` | OCPP switch | `switch.ev_charger_available` facade | Same facade value template | Charger availability | OCPP integration/entity registry |
| `switch.ev_charge_control` | Template switch | Stop, dashboard, top-level fail-safe block; disabled fixed-time stop | Dashboard | EV-facing connector 1 charge-control facade | `.storage/core.config_entries` |
| `switch.charger_connector_1_charge_control` | OCPP switch | `switch.ev_charge_control` facade | Same facade value template | Connector charge control | OCPP integration/entity registry |
| `sensor.charger_connector_1_energy_session` | OCPP sensor | OCPP integration | Session-energy template | Connector session energy | OCPP integration/entity registry |
| `sensor.sensor_ev_session_energy` | Template sensor | Home Assistant template evaluation | No consumer found under this registered ID | Mirrors connector session energy | `.storage/core.config_entries` |
| `sensor.ev_session_energy` | Unresolved reference | Not verified from implementation. | Stop automation | Intended day energy comparison input | No registry entry in export |
| `sensor.charger_connector_1_current_import` | OCPP sensor | OCPP integration | `sensor.halo_l1` | Per-phase current attributes | OCPP integration/entity registry |
| `sensor.halo_l1` | YAML template sensor | Home Assistant template evaluation | `sensor.ev_actual_current` | L1 current | `configuration.yaml` |
| `sensor.charger_connector_1_status_connector` | OCPP sensor | OCPP integration | Actual-current and status templates | Connector status | OCPP integration/entity registry |
| `sensor.ev_actual_current` | YAML template sensor | Home Assistant template evaluation | `binary_sensor.ev_is_charging` | Actual current or zero | `configuration.yaml` |
| `automation.ev_build_charge_plan` | Automation | Home Assistant automation engine | Dashboard button and chart last-triggered test | Writes day/night plan | `automations.yaml` |
| `automation.ev_charge_start` | Automation | Home Assistant automation engine | No project consumer | Start sequence | `automations.yaml` |
| `automation.ev_charge_stop` | Automation | Home Assistant automation engine | No project consumer | Charger stop sequence and maximum-current reset | `automations.yaml` |
| `automation.ev_restore_defaults_after_day_charging_window` | Automation | Home Assistant automation engine | No project consumer | Calls the restore script when the selected day block ends | `automations.yaml` |
| `automation.set_halo_charge_current` | Automation | Home Assistant automation engine | No project consumer | OCPP current service call | `automations.yaml` |
| `script.ev_restore_default_values` | Script | Automation and Settings dashboard button invoke it | Four default helpers | Restores active day/night target and current helpers; does not change maximum current | `scripts.yaml` |

## 10. File Reference

This list covers every file in the export that directly defines, registers, stores, or supplies a dependency used by the EV Charging Planner. Installed library source files that merely implement the OCPP, Nord Pool, or ApexCharts packages are dependency code rather than project configuration; their relevant manifests/service schema are listed separately.

| File | Purpose | EV content | Referenced by |
|---|---|---|---|
| `.HA_VERSION` | Exported Home Assistant version | `2026.7.2` | Snapshot metadata |
| `configuration.yaml` | Root HA configuration | Includes, OCPP logging, YAML input number, top-level fail-safe block, three template entities | Home Assistant startup |
| `automations.yaml` | Automation include | All active and disabled EV automations | `configuration.yaml` |
| `scripts.yaml` | Script include | `EV Restore Default Values` and unrelated lighting scripts | `configuration.yaml`; restore automation; EV Energy Settings button |
| `scenes.yaml` | Scene include | No EV reference found | `configuration.yaml` |
| `.storage/input_boolean` | Input-boolean helper storage | Smart charging enabled | Start/stop/dashboard |
| `.storage/input_datetime` | Input-datetime helper storage | Windows, legacy start/stop, plan timestamp | Plan, cost templates, dashboard |
| `.storage/input_number` | Input-number helper storage | Targets, currents, defaults, legacy values | Planner, start/stop, restore script, cost templates |
| `.storage/input_text` | Input-text helper storage | Selected day/night blocks | Planner, recommendation templates, dashboard |
| `.storage/core.config_entries` | Active config-entry store | Template helpers, Nord Pool entries, OCPP entry | HA integration/template setup |
| `.storage/core.config_entries.backup` | Backup copy of config entries | Previous storage snapshot | Not loaded as current config by evidence in export |
| `.storage/core.entity_registry` | Entity registration and final IDs | EV, OCPP, Nord Pool, template, and automation entities | HA entity platform |
| `.storage/core.device_registry` | Device registration | OCPP/Nord Pool device associations | HA device model |
| `.storage/core.restore_state` | Restored runtime snapshot | Helper states, automation enabled states, last-triggered times | HA restore-state mechanism |
| `.storage/lovelace.ev_energy` | Dashboard body | EV status/settings cards, restore button, and chart | EV Energy dashboard |
| `.storage/lovelace_dashboards` | Dashboard registry | EV Energy registration | Lovelace dashboard loader |
| `.storage/lovelace_resources` | Lovelace resources | ApexCharts Card module URL | EV Energy chart |
| `custom_components/ocpp/manifest.json` | OCPP dependency metadata | Version and Python requirements | OCPP integration loading |
| `custom_components/ocpp/services.yaml` | OCPP service schema | `set_charge_rate` and other available OCPP services | HA service registry; current project calls only `set_charge_rate` |
| `custom_components/nordpool/manifest.json` | Nord Pool dependency metadata | Version and integration metadata | Nord Pool integration loading |
| `www/community/apexcharts-card/apexcharts-card.js` | Installed frontend card | ApexCharts Card implementation | Lovelace resource URL |

`secrets.yaml` exists in the export. Its contents are intentionally not reproduced, and no `!secret` reference occurs in the EV YAML shown above.

## 11. Historical Components

### 11.1 Disabled historical automations

The Phase 2 report identifies older lock and planning components as historical artifacts. The restore snapshot confirms these automations are off:

- `automation.ev_day_lock_selected_charge_block`
- `automation.ev_night_lock_selected_charge_block`
- `automation.ev_planned_charging_test`
- `automation.nattladdning_pa`
- `automation.nattladdning_av`

Their complete YAML is retained in Section 5.7 because they remain in `automations.yaml`.

### 11.2 Historical general planner

The following enabled template entries and their inputs remain configured but are outside the current `EV Build Charge Plan` chain:

- `input_number.ev_required_energy`
- `input_number.ev_charger_power`
- `input_number.ev_charge_power_kw`
- `input_datetime.ev_charge_start`
- `input_datetime.ev_charge_stop`
- `sensor.ev_requiered_charging_hours`
- `sensor.ev_cheapest_charge_slots`
- `sensor.ev_cheapest_charge_slots_2`
- `sensor.ev_estimated_charge_cost`
- `sensor.ev_cheapest_continuous_charge_block`

Their exact templates are in Section 4.4.

### 11.3 Orphaned entity-registry entries

The following YAML-template entities remain registered in `.storage/core.entity_registry`, but no corresponding definition exists in the current `configuration.yaml` or active template config entries:

- `sensor.ev_day_top_up_plan` (`unique_id: ev_day_topup_plan`)
- `sensor.ev_night_main_charge_plan` (`unique_id: ev_night_main_charge_plan`)
- `binary_sensor.ev_day_charging_active` (`unique_id: ev_day_charging_active`)
- `binary_sensor.ev_night_charging_active` (`unique_id: ev_night_charging_active`)
- `binary_sensor.ev_charging_planned_active` (`unique_id: ev_charging_planned_active`)
- `binary_sensor.ev_tomorrow_prices_ready` (`unique_id: ev_tomorrow_prices_ready`)

`EV Planned Charging Test` still references `binary_sensor.ev_charging_planned_active`, but the automation is restored off.

### 11.4 Missing entities referenced by historical lock automations

These references occur in the disabled lock automation YAML, but neither entity exists in the exported entity registry or template config entries:

- `sensor.ev_day_cheapest_continuous_charge_block`
- `sensor.ev_night_cheapest_continuous_charge_block`

### 11.5 Additional configured but unused artifacts

- The second Nord Pool config entry `01KS06ZB4TP1XETVMMJFM3C7SQ` is enabled but has no EV planner entity association in the registry.
- `sensor.ev_day_charge_visual`, `sensor.ev_night_charge_visual`, `sensor.ev_day_visual_block`, and `sensor.ev_night_visual_block` remain enabled, but the current EV Energy graph implements block highlighting directly in its JavaScript generator and does not reference these sensors.
- `binary_sensor.ev_is_charging` and `sensor.ev_charger_status` remain enabled but no current automation or EV Energy dashboard reference was found.
- `input_number.halo_charge_current` remains defined in YAML but no consumer was found.
- The top-level `EV Charger Fail Safe On Startup` block remains in `configuration.yaml`, but no automation entity registration was found. Its runtime status is **Not verified from implementation.**
