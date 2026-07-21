# EV Charging Planner Architecture

## Purpose

EV Charging Planner is a Home Assistant-based controller for an OCPP-connected EV charger. It plans charging against Nordpool spot prices and controls charging within user-defined day and night windows.

The system calculates one continuous 15-minute charging block for each window. It uses the configured energy target and charge current to determine the required duration, then selects the lowest-cost continuous block that fits inside the corresponding window.

## Overview

Home Assistant contains the planning, decision, execution, monitoring, and user-interface layers. The OCPP integration is the boundary to the charger: it provides charger state and telemetry, receives charge-rate commands, and exposes the availability and charge-control switches.

```text
Nordpool price data
        |
        v
Home Assistant charge-plan automation
        |
        v
Selected day and night blocks
        |
        v
Template decision sensors
        |
        v
Start/stop automations and charge-rate automation
        |
        v
OCPP Central System -> EV charger
```

## Architecture

The architecture has the following functional layers:

| Layer | Responsibility |
|---|---|
| Configuration and settings | Stores smart-charging state, energy targets, charge currents, time windows, defaults, and selected charge blocks. |
| Price acquisition | Provides Nordpool SE3 price data, including raw 15-minute prices for the following day. |
| Planning | Builds separate day and night charge plans at 23:00. |
| Decision | Determines whether the current time belongs to either selected block. |
| Execution | Applies the charge-rate limit and controls the charger's availability and active charge control. |
| Charger integration | Connects Home Assistant to the charger through OCPP. |
| Monitoring and UI | Exposes derived runtime values and presents status, settings, plans, and prices in the EV Energy dashboard. |

Planning and decisions are implemented in Home Assistant. OCPP is the communication and execution layer for charger commands and charger data.

## Main Components

### Nordpool price source

The configured price source is Nordpool SE3 in SEK/kWh, including VAT. The planner uses the `raw_tomorrow` price data. A plan can be built only when tomorrow's prices are valid and contain at least 96 quarter-hour entries.

### Configuration helpers

Home Assistant helpers hold the planner configuration and output:

| Helper category | Active data |
|---|---|
| Enablement | `input_boolean.ev_smart_charging_enabled` |
| Day settings | Target energy, charge current, default target/current, and start/end window |
| Night settings | Target energy, charge current, default target/current, and start/end window |
| Plan output | `input_text.ev_day_selected_charge_block` and `input_text.ev_night_selected_charge_block` |
| Plan metadata | `input_datetime.ev_plan_last_updated` |
| Applied limit | `input_number.ev_max_charge_current` |

Selected blocks are stored as `HH:MM–HH:MM` text values.

### EV Build Charge Plan

`EV Build Charge Plan` runs at 23:00. For each of the day and night windows, it:

1. Reads the relevant target energy, charge current, window boundaries, and next-day Nordpool prices.
2. Calculates charge power as `(current - 1) x 230 / 1000` kW, with a minimum of 0.1 kW.
3. Calculates the number of required 15-minute slots.
4. Evaluates continuous sequences of the required length that are fully contained in the configured time window.
5. Selects the sequence with the lowest sum of price values.
6. Stores the resulting start and end time in the selected-block helper.

If next-day prices are unavailable, the automation creates a persistent notification and stops without building a plan. It updates `ev_plan_last_updated` after a plan is built.

Day and night plans are independent. Their energy targets are not coordinated with each other.

### Recommendation sensors

The template binary sensors convert selected block values into runtime decisions:

| Sensor | Responsibility |
|---|---|
| `binary_sensor.ev_day_charge_recommended_now` | On while the current time is inside the selected day block. |
| `binary_sensor.ev_night_charge_recommended_now` | On while the current time is inside the selected night block, including a block that crosses midnight. |
| `binary_sensor.ev_charge_recommended_now` | Logical OR of the day and night recommendation sensors. |

### Execution automations

`EV Charge Start` runs when the combined recommendation becomes on and smart charging is enabled. It selects the active day or night current, writes it to `input_number.ev_max_charge_current`, waits five seconds, and turns on `switch.ev_charger_available`.

`Set Halo charge current` reacts to changes in `input_number.ev_max_charge_current` and calls `ocpp.set_charge_rate` for device ID `charger`.

`EV Charge Stop` runs when the combined recommendation becomes off, or when the day recommendation is on and session energy reaches the day target. When smart charging is enabled and the charger is available, it turns off `switch.ev_charge_control`, waits one minute, and turns off `switch.ev_charger_available`. It restores the configured day and night targets and currents to their default values under its day-related condition, then sets the maximum charge current to the default night current.

The current configured OCPP maximum current is 10 A.

### OCPP integration and charger abstraction

The OCPP Central System listens on `<HOME_ASSISTANT_IP>:9000` without TLS. The configured OCPP charge-point ID is `charger` and the configuration has one connector.

Two template switches provide the application's charger-control boundary:

| Application switch | Underlying OCPP entity | Role |
|---|---|---|
| `switch.ev_charger_available` | `switch.charger_availability` | Allows or prevents new charging. |
| `switch.ev_charge_control` | `switch.charger_connector_1_charge_control` | Stops an active charging transaction. |

Availability is the gate for starting a new charge. Normal start does not explicitly turn on charge control: an attached vehicle starts charging when the charger becomes available. At block end, charge control is turned off before availability is turned off. This sequence also applies when the charger reports `SuspendedEV` or another active transaction status.

### Monitoring and EV Energy dashboard

OCPP provides charger state and telemetry, including connector status, imported current, session energy, transaction information, and voltage.

Derived Home Assistant entities include:

| Entity | Derivation |
|---|---|
| `sensor.ev_session_energy` | OCPP session energy for connector 1. |
| `sensor.ev_actual_current` | OCPP L1 current when connector 1 status is `Charging`; otherwise zero. |
| `binary_sensor.ev_is_charging` | On when the actual EV current is greater than zero. |
| `binary_sensor.tomorrow_prices_available` | On when next-day Nordpool price data is valid and has at least 96 entries. |

The storage-based EV Energy Lovelace dashboard provides a manual plan-build action, smart-charging state, day and night targets, selected blocks, estimated costs, recommendation state, charger switches, price availability, configuration settings, and an ApexCharts view of current and next-day prices with selected blocks highlighted.

## Information Flows

### Plan input to plan output

```text
User settings + Nordpool raw_tomorrow
        -> EV Build Charge Plan
        -> selected day block and selected night block
        -> plan last-updated timestamp
```

The planner reads targets, currents, and windows separately for day and night. Each selected block represents the lowest-cost continuous quarter-hour sequence that satisfies the calculated required duration within that window.

### Plan output to charging decision

```text
Selected day block  -> day recommendation sensor --+
                                                  +-> combined recommendation
Selected night block -> night recommendation sensor -+
```

The combined recommendation is evaluated together with the smart-charging enablement helper before automated start or stop actions are executed.

### Charger data to monitoring

```text
EV charger -> OCPP integration -> OCPP entities -> derived sensors -> EV Energy dashboard
```

## Decision Flows

### Plan construction decision

At 23:00, the planner verifies that tomorrow's Nordpool data is valid. It then searches each configured window for continuous slot sequences of the duration required by that window's target energy and charge current. The sequence with the lowest total price is selected.

### Start decision

Charging is started when all of the following are true:

- The combined recommendation changes to on.
- `input_boolean.ev_smart_charging_enabled` is on.

The automation applies the day current if the day recommendation is active; otherwise it applies the night current when the night recommendation is active. It then enables charger availability after the five-second delay.

### Stop decision

Charging is stopped when either of the following triggers occurs:

- The combined recommendation changes to off.
- The day recommendation is on and session energy reaches the configured day target.

The stop sequence is conditional on smart charging being enabled and the charger being available. It first disables active charge control, then disables charger availability after one minute.

## Runtime Flows

### Normal charging window

```text
Selected block becomes active
  -> combined recommendation becomes on
  -> selected current is written
  -> OCPP charge-rate limit is sent
  -> 5-second delay
  -> charger availability is enabled
  -> connected vehicle charges
```

### End of charging window or day-target completion

```text
Stop trigger
  -> charge control is disabled
  -> 1-minute delay
  -> charger availability is disabled
  -> applicable configured values are restored to defaults
  -> maximum current is set to default night current
```

### Home Assistant startup failsafe

`EV Charger Fail Safe On Startup` runs at Home Assistant startup. After a two-and-a-half-minute delay, it turns off charge control, waits up to 30 seconds for charger status to no longer be `Charging`, and turns off charger availability. This failsafe is an active safety rule.

## Integrations

| Integration | System role |
|---|---|
| Home Assistant | Hosts configuration helpers, automations, templates, dashboard, planning, and decisions. |
| Nordpool | Supplies SE3 spot prices in SEK/kWh, including raw next-day quarter-hour data. |
| OCPP | Provides the Central System connection, charger entities, telemetry, and control services. |
| ApexCharts Card | Renders the price and selected-block visualization in the EV Energy dashboard. |

## Design Principles

- Planning and charger execution are separated: Home Assistant calculates plans and makes decisions; OCPP communicates those commands to the charger.
- The day and night charge plans are separate and independently calculated.
- Plans use continuous 15-minute intervals, not independent lowest-price intervals.
- Charger availability prevents new charging outside permitted periods.
- Charge control terminates an active charging transaction at the end of a permitted period.
- The start sequence is charge-rate limit, five-second delay, then charger availability.
- The stop sequence is charge-control off, one-minute delay, then charger availability off.
- Smart charging enablement gates the automated start and stop automations.
- The startup failsafe actively returns charger control and availability to the off state.

## Historical Artifacts Outside the Active Architecture

The configuration retains historical planning entities and the `EV Day Lock Selected Charge Block` and `EV Night Lock Selected Charge Block` automations. They are not part of the active day/night planning and start/stop chain described in this document.
