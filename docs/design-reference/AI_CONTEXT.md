# EV Charging Planner: AI Context

## Project Purpose

EV Charging Planner is a Home Assistant-based system for planning and controlling an OCPP-connected EV charger. It selects the lowest-cost continuous charging periods from Nordpool SE3 electricity prices while respecting configured day and night energy targets, current limits, and time windows.

## System Model

```text
Nordpool next-day quarter-hour prices
  -> Home Assistant plan builder
  -> selected day and night charge blocks
  -> recommendation sensors
  -> start/stop and charge-rate automations
  -> OCPP integration
  -> EV charger
```

Home Assistant owns planning, decisions, automation, and the operator interface. OCPP is the charger communication and execution layer.

At 23:00, `EV Build Charge Plan` calculates separate day and night blocks from `raw_tomorrow` prices. A block is a continuous `HH:MM–HH:MM` sequence of 15-minute slots. It must fit inside the respective configured window and have the lowest total price for the duration calculated from the target energy and charge current.

The combined recommendation is on when either the day or night block is currently active. Automated execution requires smart charging to be enabled.

## Important Design Principles

- Day and night plans are independent; their energy targets are not coordinated.
- Planning uses continuous quarter-hour intervals, not independent cheapest intervals.
- The configured time window determines the deadline for charging.
- Charger availability permits or prevents a new charging session.
- Charge control ends an active charging transaction.
- Normal start does not explicitly enable charge control.
- The start sequence is: set charge-rate limit, wait five seconds, enable availability.
- The stop sequence is: disable charge control, wait one minute, disable availability.
- The Home Assistant startup failsafe is an active safety rule.

## Source of Truth

For documentation and implementation conflicts, Phase 2 is authoritative.

For active runtime behavior, use the following sources:

| Domain | Source of truth |
|---|---|
| Current implementation | Home Assistant configuration and storage configuration |
| Price data | Nordpool sensor, including `raw_tomorrow` and `tomorrow_valid` |
| Planner inputs | Relevant Home Assistant input helpers |
| Selected charge blocks | `input_text.ev_day_selected_charge_block` and `input_text.ev_night_selected_charge_block` |
| Time-based decision | Day, night, and combined recommendation binary sensors |
| Charger status and telemetry | OCPP entities |
| Charger control boundary | `switch.ev_charger_available` and `switch.ev_charge_control` |

## Important Entities

### Planning Inputs and Outputs

| Entity | Role |
|---|---|
| `input_boolean.ev_smart_charging_enabled` | Enables automated start and stop actions. |
| `input_number.ev_day_target_kwh` | Day energy target. |
| `input_number.ev_night_target_kwh` | Night energy target. |
| `input_number.ev_day_charge_current` | Day charge current. |
| `input_number.ev_night_charge_current` | Night charge current. |
| `input_number.ev_max_charge_current` | Current limit sent through OCPP. |
| `input_datetime.ev_day_window_start` / `input_datetime.ev_day_window_end` | Day charging window. |
| `input_datetime.ev_night_window_start` / `input_datetime.ev_night_window_end` | Night charging window. |
| `input_text.ev_day_selected_charge_block` | Selected day block. |
| `input_text.ev_night_selected_charge_block` | Selected night block. |
| `input_datetime.ev_plan_last_updated` | Timestamp of the latest plan build. |

### Decisions and Monitoring

| Entity | Role |
|---|---|
| `binary_sensor.ev_day_charge_recommended_now` | Day block is active now. |
| `binary_sensor.ev_night_charge_recommended_now` | Night block is active now. |
| `binary_sensor.ev_charge_recommended_now` | Day OR night recommendation. |
| `binary_sensor.tomorrow_prices_available` | Next-day price data is valid and has at least 96 entries. |
| `sensor.ev_session_energy` | OCPP session energy for connector 1. |
| `sensor.ev_actual_current` | OCPP L1 current while connector 1 is charging; otherwise zero. |
| `binary_sensor.ev_is_charging` | Derived charging state based on actual EV current. |

### Charger Control

| Entity | Role |
|---|---|
| `switch.ev_charger_available` | Application-level availability control; abstracts `switch.charger_availability`. |
| `switch.ev_charge_control` | Application-level active-transaction control; abstracts `switch.charger_connector_1_charge_control`. |

## Important Automations

| Automation | Role |
|---|---|
| `EV Build Charge Plan` | Builds next-day day and night selected blocks at 23:00. |
| `EV Charge Start` | Applies the active current and enables charger availability. |
| `EV Charge Stop` | Disables charge control, then availability, and performs configured resets. |
| `Set Halo charge current` | Sends `input_number.ev_max_charge_current` through `ocpp.set_charge_rate` for device ID `charger`. |
| `EV Charger Fail Safe On Startup` | Turns off charge control and availability after Home Assistant startup. |

## Important Integrations

| Integration | Role |
|---|---|
| Home Assistant | Hosts helpers, templates, automations, and the EV Energy dashboard. |
| Nordpool | Provides SE3 spot prices in SEK/kWh and next-day quarter-hour prices. |
| OCPP | Provides charger control, state, telemetry, and `ocpp.set_charge_rate`. |
| ApexCharts Card | Displays price data and selected charging blocks in the EV Energy dashboard. |

## Rules That Must Always Be Followed

- Treat Phase 2 as the authoritative project model when it conflicts with Phase 1 or an interpretation of the implementation.
- Document or change only verified behavior; keep hypotheses separate from verified facts.
- Do not alter the day/night planning model without explicit project direction.
- Do not replace the availability-based start gate or the charge-control-based stop mechanism without explicit project direction.
- Preserve the ordering and delays of the implemented start and stop sequences.
- Preserve the startup failsafe as an active safety rule.
- Treat historical planning entities and the two selected-block lock automations as outside the active planning and start/stop chain.

## Terminology

| Term | Meaning |
|---|---|
| Day block | The selected continuous charging interval calculated from the day settings. |
| Night block | The selected continuous charging interval calculated from the night settings; it can cross midnight. |
| Selected block | A stored `HH:MM–HH:MM` interval chosen by the plan builder. |
| Recommendation | A binary decision that the current time lies inside a selected block. |
| Availability | The charger state that permits a connected vehicle to begin a new charge. |
| Charge control | The control used to stop an active charging transaction. |
| Smart charging | The enablement condition for automated start and stop actions. |
| OCPP | The protocol integration used for charger commands, state, and telemetry. |
