# EV Charging Planner Design Decisions

## Scope

This document records verified design decisions in the current EV Charging Planner implementation. It describes the implemented system only.

## Decision: Home Assistant Owns Planning and Decisions; OCPP Owns Charger Communication

### Decision

Home Assistant performs price-based planning, time-based charging decisions, and automation orchestration. The OCPP integration is the communication and execution layer for the charger.

### Motivation

The planner requires Nordpool price data, user-configured targets, currents, and time windows. These inputs and the resulting plan are implemented as Home Assistant helpers, templates, and automations. Charger commands and telemetry are available through OCPP entities and services.

### Consequences

- Charge-plan state and charging decisions are visible and controllable in Home Assistant.
- OCPP exposes the charger boundary for availability, charge control, charge-rate commands, status, and telemetry.
- The EV Energy dashboard can present both plan state and charger state from Home Assistant entities.

## Decision: Calculate Separate Day and Night Plans

### Decision

The system calculates independent continuous charging blocks for day and night.

### Motivation

Each period has its own target energy, charge current, and allowed time window. The active implementation builds both plans from next-day price data at 23:00.

### Consequences

- Day and night selected blocks are stored separately.
- Each plan is evaluated by its own recommendation sensor.
- Day and night energy targets are not coordinated with each other while the vehicle cannot report its energy requirement.

## Decision: Select the Lowest-Cost Continuous Quarter-Hour Block

### Decision

For each day and night plan, the planner selects the lowest-cost continuous sequence of 15-minute price intervals that fits inside the configured time window.

### Motivation

The required duration is calculated from the configured energy target and charge current. The planner evaluates only sequences that contain the required number of contiguous quarter-hour slots.

### Consequences

- The selected plan is a single `HH:MM–HH:MM` block for each period.
- The plan does not consist of independent lowest-price quarter-hour intervals.
- A plan is built only when valid next-day Nordpool data contains at least 96 quarter-hour entries.

## Decision: Use Availability to Permit New Charging

### Decision

`switch.ev_charger_available` is used as the gate that permits or prevents new charging.

### Motivation

A connected vehicle can start charging when the charger becomes available. Availability is therefore used as the control outside permitted charging periods.

### Consequences

- The normal start sequence ends by enabling charger availability.
- The system prevents a vehicle from beginning a new charge when availability is off.
- A template switch abstracts the underlying OCPP availability entity.

## Decision: Use Charge Control to End an Active Transaction

### Decision

`switch.ev_charge_control` is turned off to end an active charging transaction at the end of a permitted charging period.

### Motivation

Charge control is the intended mechanism for stopping an active transaction. Normal charging start is performed by making the charger available rather than by explicitly turning on charge control.

### Consequences

- The stop sequence turns off charge control before it turns off availability.
- Availability is turned off after a fixed one-minute delay.
- The same sequence applies when the charger reports `SuspendedEV` or another active transaction status.
- A template switch abstracts the underlying OCPP connector charge-control entity.

## Decision: Apply the Current Limit Before Enabling Availability

### Decision

The system writes the selected current to `input_number.ev_max_charge_current`, sends it through `ocpp.set_charge_rate`, waits five seconds, and then enables charger availability.

### Motivation

This is the verified implemented start chain for automated charging.

### Consequences

- Day charging applies the configured day current when the day recommendation is active.
- Night charging applies the configured night current when the night recommendation is active.
- The charger is made available only after the charge-rate update path has been initiated and the five-second delay has elapsed.

## Decision: Gate Automated Execution with Smart-Charging Enablement

### Decision

Automated start and stop actions require `input_boolean.ev_smart_charging_enabled` to be on.

### Motivation

The enablement helper is an explicit condition in both `EV Charge Start` and `EV Charge Stop`.

### Consequences

- A recommendation alone does not execute automated charging actions when smart charging is disabled.
- Plan output and recommendation entities remain distinct from the enablement condition used for execution.

## Decision: Treat the Time Window as the Charging Deadline

### Decision

The configured charging window determines when charging must end.

### Motivation

If the vehicle reaches its target within the window, charging can end; otherwise charging ends when the selected block and allowed window end. The active stop automation additionally triggers when the day target is reached during an active day recommendation.

### Consequences

- A vehicle may remain below its target when the permitted window ends.
- Reaching the configured day target can invoke the stop sequence before the selected day block ends.

## Decision: Restore Defaults After the Day-Related Stop Condition

### Decision

The stop automation restores configured day and night target and current helpers to their default values when its day-related condition is true, and then applies the default night current as the maximum charge current.

### Motivation

This behavior is implemented in `EV Charge Stop` and is verified as the current behavior.

### Consequences

- After the applicable stop path, day and night targets and currents return to their configured defaults.
- `input_number.ev_max_charge_current` is set to the default night current after every stop execution.

## Decision: Keep the Startup Failsafe Active

### Decision

Home Assistant startup runs an EV charger failsafe that turns off charge control and then charger availability.

### Motivation

The failsafe is a verified active safety rule. It is required to ensure availability can be turned off after startup.

### Consequences

- The failsafe waits two and a half minutes after Home Assistant starts.
- It turns off charge control, waits up to 30 seconds for the charger status to stop reporting `Charging`, and turns off availability.
- Charger control is returned to the off state through the same application-level abstractions used by the runtime flow.

## Historical Artifacts

The older selected-block lock automations and older planning entities are historical artifacts. They are not part of the active day/night planning or start/stop chain documented above.
