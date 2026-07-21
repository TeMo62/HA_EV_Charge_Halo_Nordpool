# EV Charging Planner Developer Guide

## 1. Purpose and Scope

This guide describes how to maintain and change the current EV Charging Planner without moving responsibility between its established architectural layers or changing its verified charging behavior.

It covers:

- Home Assistant planning logic
- configuration and runtime helpers
- template and derived entities
- charging automations
- the OCPP boundary
- the EV Energy dashboard
- operational verification and safe-change practices

It does not document charger firmware, electrical installation work, Nordpool integration internals, unrelated Home Assistant configuration, or functionality that has not been implemented and verified.

Readers are expected to understand Home Assistant entities, YAML configuration, automations, template syntax, and safe operation of an EV charger.

This guide is organized by architectural responsibility first. File and entity locations are mapped to those responsibilities, but the file layout does not define the architecture.

## 2. Architectural Overview

The system separates user configuration, planning, runtime decisions, physical execution, and presentation.

```text
                         EXTERNAL INPUTS
                +-----------------------------+
                | Nordpool price attributes   |
                | OCPP charger observations   |
                +--------------+--------------+
                               |
                               v
  +----------------+    +----------------------+    +----------------------+
  | User settings  |--->| Planning layer       |--->| Selected day/night   |
  | and defaults   |    | EV Build Charge Plan |    | blocks + timestamp   |
  +----------------+    +----------------------+    +----------+-----------+
                                                               |
                                                               v
                                                    +----------------------+
                                                    | Derived runtime      |
                                                    | recommendation state |
                                                    +----------+-----------+
                                                               |
                                                               v
                                                    +----------------------+
                                                    | Execution layer      |
                                                    | Start / Stop / Rate  |
                                                    +----------+-----------+
                                                               |
                                                               v
                                                    +----------------------+
                                                    | OCPP boundary        |
                                                    | Halo charger         |
                                                    +----------------------+

  Dashboard -------------------------------------------------------------->
  Displays state and exposes approved helper/manual controls; it does not
  own planning or execution logic.
```

### 2.1 Architectural layers

| Layer | Primary responsibility | Main implementation |
|---|---|---|
| Configuration | Store user-operated targets, currents, windows, defaults, and smart-charging enablement. | Storage-backed `input_number`, `input_datetime`, and `input_boolean` helpers. |
| Planning | Calculate one lowest-cost continuous day block and one lowest-cost continuous night block. | `EV Build Charge Plan` in `automations.yaml`. |
| Plan state | Store the calculated blocks and plan-build timestamp. | `input_text.ev_day_selected_charge_block`, `input_text.ev_night_selected_charge_block`, `input_datetime.ev_plan_last_updated`. |
| Derived runtime decision | Determine whether the current time is inside the selected day or night block and combine those decisions. | Template binary sensors. |
| Execution | Apply the selected current, start or stop charging, restore configured values, and provide startup safety behavior. | `EV Charge Start`, `EV Charge Stop`, `Set Halo charge current`, and `EV Charger Fail Safe On Startup`. |
| Charger communication | Translate Home Assistant execution into OCPP commands and expose charger observations. | OCPP integration, OCPP services, and OCPP-backed entities. |
| Presentation and operation | Show settings, plans, prices, state, and charger health; expose approved controls. | `.storage/lovelace.ev_energy` and ApexCharts Card. |

## 3. Architectural Invariants

The following rules define the current design and must not be broken without an explicit, documented design change.

1. **The planner decides; execution automations execute.**  
   `EV Build Charge Plan` owns block selection. Start and stop automations must not recalculate prices or choose alternate blocks.

2. **Day and night planning remain independent.**  
   They have separate targets, currents, windows, selected blocks, recommendation states, and execution-current selection.

3. **The active plan consists of one continuous 15-minute block per configured window.**  
   The current planner does not select a disconnected set of individually cheap slots.

4. **Selected blocks are planner outputs, not execution state.**  
   Execution automations read them indirectly through recommendation sensors and must not overwrite them.

5. **Templates derive state; they do not cause charger actions.**  
   Sensors and binary sensors may calculate recommendations and observations but must not invoke OCPP or operate switches.

6. **OCPP communicates with the charger; it does not plan.**  
   Price selection, day/night policy, and time-window decisions remain in Home Assistant's planner layer.

7. **The dashboard presents and exposes approved controls; it does not own business logic.**  
   A dashboard card must not become an alternative planner, recommendation engine, or charger-control sequence.

8. **Automated start and stop remain gated by `input_boolean.ev_smart_charging_enabled`.**  
   Plan calculation may continue while automatic execution is disabled.

9. **Start sequencing remains current first, availability second.**  
   The selected day/night current is written, the system waits five seconds for the existing current-setting automation and OCPP path to act, and only then is charger availability enabled.

10. **Stop sequencing remains charge control first, availability second.**  
    Active charging is stopped through charge control, the system waits one minute, and only then is availability disabled.

11. **`EV Charger Fail Safe On Startup` remains an active safety rule.**  
    It must not be removed or bypassed as an incidental part of another change.

12. **Historical planner and lock entities are not active design dependencies.**  
    Their presence must not be used as justification for duplicating or reviving the former live/replanning flow.

The broader policy that configuration helpers are the mandatory authoritative source for every project setting is not fully verified as a project-wide design decision. See [assumptions.md](assumptions.md).

## 4. Core Data Flow

### 4.1 Planning flow

```text
Nordpool raw_tomorrow + tomorrow_valid
                     |
Day/night targets ---+
Day/night currents --+--> EV Build Charge Plan (23:00)
Day/night windows ----+             |
                                    +--> selected day block
                                    +--> selected night block
                                    +--> plan last-updated timestamp
```

The planner uses next-day price data at its normal 23:00 execution time. It calculates the number of required 15-minute slots from target energy and effective charging power, searches for the lowest-cost continuous candidate inside each configured window, and stores each result as `HH:MM–HH:MM` or `none`.

### 4.2 Runtime recommendation and execution flow

```text
selected day block ---> day recommendation ----+
                                                |
selected night block -> night recommendation --+--> combined recommendation
                                                         |
                             +---------------------------+------------------+
                             |                                              |
                          turns ON                                       turns OFF
                             |                                              |
                             v                                              v
                    EV Charge Start                                EV Charge Stop
                    1. choose day/night current                     1. charge control OFF
                    2. set max-current helper                       2. wait 1 minute
                    3. wait 5 seconds                               3. availability OFF
                    4. availability ON                              4. applicable resets
                             |                                              |
                             +---------------- OCPP -------------------------+
```

### 4.3 Current-setting flow

```text
EV Charge Start / Stop
          |
          v
input_number.ev_max_charge_current
          |
          v
Set Halo charge current
          |
          v
ocpp.set_charge_rate(devid: charger, limit_amps: ...)
          |
          v
Halo charger
```

This indirection is intentional: execution selects the desired current through the helper, while the existing OCPP automation owns transmission of that value to the charger.

## 5. Development Principles and Rationale

- Keep planning and time-based decisions in Home Assistant; keep charger communication at the OCPP boundary. This isolates price and policy logic from charger-specific protocol behavior.
- Keep day and night planning independent because the two passes have different energy targets, currents, and allowed windows.
- Keep the planner's output as one lowest-cost continuous 15-minute block per configured window. Execution can then use one stable start/end interval rather than coordinate fragmented slots.
- Do not duplicate planner or recommendation logic in execution automations. Duplication would allow the calculated plan and the executed plan to diverge.
- Do not move business logic into dashboard cards. Dashboard refresh and rendering behavior must not determine whether charging occurs.
- Keep `switch.ev_charger_available` as the gate for starting a new charging session. With the vehicle connected, enabling availability permits the charger to begin; the active start path does not explicitly enable charge control.
- Keep `switch.ev_charge_control` as the mechanism used first to stop an active transaction. Charger availability is not treated as the reliable first step for terminating an already active charge.
- Preserve the start order: set current limit, wait five seconds, enable availability. The delay gives the helper-triggered OCPP current-setting path time to apply the intended limit before charging is allowed to begin.
- Preserve the stop order: disable charge control, wait one minute, disable availability. This reflects verified charger behavior: availability must not be relied upon to stop an active transaction, and the delay gives the charger time to leave the active charging state before availability is withdrawn.
- Keep automatic start and stop behind `input_boolean.ev_smart_charging_enabled`. Planning may still run while automatic execution is disabled, allowing plan inspection and experimentation without charger actuation.
- Preserve the startup failsafe so Home Assistant does not leave charging permitted merely because runtime recommendation state has not yet settled after restart.
- Treat selected-block lock automations and older planner entities as historical artifacts outside the active plan/start/stop chain.

## 6. Responsibility Boundaries

| Layer | May decide | May perform | Owns | Must not do |
|---|---|---|---|---|
| Configuration helpers | No charging decision by themselves. | Accept approved user and automation writes. | Stored settings and defaults. | Calculate plans or communicate with OCPP. |
| Planner (`EV Build Charge Plan`) | Select the lowest-cost continuous block for each configured window. | Write selected blocks and plan timestamp. | Plan calculation and plan outputs. | Start or stop the charger directly. |
| Templates and derived sensors | Determine whether a selected block is active; derive display/runtime observations. | Publish computed entity state. | Recommendation and derived-state calculation. | Issue charger commands or replace the planner. |
| Execution automations | Choose the applicable current and react to recommendation/day-target events. | Set current, operate application switches, and restore configured values. | Start, stop, reset, and sequencing behavior. | Recalculate selected blocks. |
| OCPP integration | No price or schedule decisions. | Send charger commands and expose charger state/telemetry. | Protocol boundary and OCPP entities. | Replace Home Assistant's day/night policy. |
| Charger | Apply accepted OCPP controls and report status. | Start charging when an attached vehicle is made available; maintain physical transaction state. | Physical charging behavior and telemetry. | Be assumed to provide vehicle energy need to the planner. |
| EV Energy dashboard | No charging decision. | Display entities and invoke existing manual plan-build/helper controls. | Presentation and user interaction. | Become a business-logic or safety layer. |

## 7. Implementation Map

The architecture above is implemented in the following locations. These locations are implementation details, not responsibility definitions.

| Location or component | Architectural role | Important dependencies | Changes that belong here |
|---|---|---|---|
| `configuration.yaml` | Root loading, inline derived entities, and startup safety. | Home Assistant configuration loader; OCPP and Nordpool entities. | Root includes, inline templates, and `EV Charger Fail Safe On Startup`. |
| `automations.yaml` | Planning and execution. | Helpers, recommendation sensors, OCPP service, and template switches. | `EV Build Charge Plan`, `EV Charge Start`, `EV Charge Stop`, and `Set Halo charge current`. |
| `.storage/input_boolean` | Stored execution enablement. | Home Assistant helper registry. | Smart-charging enablement helper definition. |
| `.storage/input_datetime` | Stored windows and plan metadata. | Planner and dashboard. | Day/night windows and plan timestamp. |
| `.storage/input_number` | Stored targets, currents, and defaults. | Planner, start/stop automation, current-setting automation, dashboard. | Targets, currents, defaults, and maximum current. |
| `.storage/input_text` | Stored planner outputs. | Planner and recommendation templates. | Selected day and night block values. |
| `.storage/core.config_entries` | Integration and template configuration. | OCPP, Nordpool, and template entities. | Existing integration/config-entry data. Direct editing is not an ordinary development workflow. |
| `.storage/lovelace.ev_energy` | Presentation and approved controls. | Existing helpers and sensors; ApexCharts Card. | EV Energy dashboard presentation. |
| `scripts.yaml` | Script configuration included by the root config. | Home Assistant script configuration. | No active EV planner script is documented in the current system model. |

The active EV automation aliases are:

- `EV Build Charge Plan`
- `EV Charge Start`
- `EV Charge Stop`
- `Set Halo charge current`
- `EV Charger Fail Safe On Startup`

The startup failsafe is defined in `configuration.yaml`; the other listed active EV automations are in `automations.yaml`.

## 8. Source of Truth and Write Ownership

Phase 2 is authoritative when documentation, Phase 1, and an interpretation of configuration conflict. The table distinguishes stored configuration, calculated outputs, runtime decisions, and charger observations. It does not establish the unverified policy that every helper is the project's permanent source of truth.

| Information | Current storage or producer | Classification | Writers |
|---|---|---|---|
| Smart-charging enablement | `input_boolean.ev_smart_charging_enabled` | Stored configuration state | User/dashboard. |
| Day/night targets, currents, defaults, and windows | Relevant `input_number` and `input_datetime` helpers | Stored configuration state | User/dashboard; `EV Charge Stop` restores targets and currents under its applicable condition. |
| Selected day/night period | `input_text.ev_day_selected_charge_block`, `input_text.ev_night_selected_charge_block` | Calculated plan output | `EV Build Charge Plan`; historical lock automations remain configured but are outside the active chain. |
| Plan update time | `input_datetime.ev_plan_last_updated` | Calculated plan metadata | `EV Build Charge Plan`. |
| Applied current limit | `input_number.ev_max_charge_current` | Execution state and OCPP command input | `EV Charge Start` and `EV Charge Stop`; observed by `Set Halo charge current`. |
| Recommendation now | `binary_sensor.ev_day_charge_recommended_now`, `binary_sensor.ev_night_charge_recommended_now`, `binary_sensor.ev_charge_recommended_now` | Calculated runtime decision | Template definitions only. |
| Session energy, connector status, voltage, imported current | OCPP entities | Charger observation/runtime status | OCPP integration. |
| Actual EV current and charging state | `sensor.ev_actual_current`, `binary_sensor.ev_is_charging` | Derived runtime observation | Template definitions only. |
| Price data | Nordpool sensor attributes, including `raw_tomorrow` and `tomorrow_valid` | External observation | Nordpool integration. |
| Availability and charge control | Underlying OCPP switches, exposed through the two `ev_` template switches | Charger command/state boundary | Execution automations, startup failsafe, and manual operator actions. |

Do not treat OCPP telemetry, recommendation states, or dashboard rendering as persistent configuration. Do not write selected-block values from execution automations.

## 9. Naming Conventions

The active EV entity IDs use lowercase snake case with the `ev_` prefix. Day and night variants use `ev_day_` and `ev_night_` respectively.

| Item | Established examples |
|---|---|
| Helpers | `input_number.ev_day_target_kwh`, `input_datetime.ev_night_window_end`, `input_text.ev_day_selected_charge_block` |
| Sensors | `sensor.ev_session_energy`, `sensor.ev_actual_current` |
| Binary sensors | `binary_sensor.ev_day_charge_recommended_now`, `binary_sensor.ev_charge_recommended_now` |
| Application switches | `switch.ev_charger_available`, `switch.ev_charge_control` |
| Active automation aliases | `EV Build Charge Plan`, `EV Charge Start`, `EV Charge Stop` |
| Template variables | `energy`, `amps`, `power`, `needed_slots`, `window_start`, `window_end`, `slots`, `best`, `np` |
| OCPP target | Device ID `charger` |

The active planner uses English aliases and `EV` prefixing. The wider configuration contains older and unrelated names, including Swedish aliases; no universal naming convention is verified beyond the active EV naming pattern.

## 10. Adding or Changing a Helper

Follow this process for a helper required by an explicitly approved change:

1. Classify the value as stored user setting, planner output, or execution state. Do not create a helper for a value already produced by a sensor or OCPP entity.
2. Use the corresponding existing family: `input_number` for targets/currents/defaults, `input_datetime` for windows or timestamps, `input_text` for selected blocks, and `input_boolean` for enablement.
3. Add or change the helper through the existing Home Assistant storage-backed helper configuration. Use the active `ev_`, `ev_day_`, or `ev_night_` naming pattern where applicable.
4. Update every active planner, template, automation, and dashboard reference that must use the value. Reference the entity ID; do not duplicate the setting as a literal elsewhere.
5. If the helper affects plan duration, change the calculation only in `EV Build Charge Plan`. If it affects execution, change only the relevant execution automation after confirming the responsibility boundary.
6. Expose the helper in `.storage/lovelace.ev_energy` only when it is intended to be user-operated or displayed there.
7. Validate the entity ID, range, unit, default, and all template references before use.

For an `input_number`, verify range, step, unit, and whether it is a day, night, default, or applied-current value. For an `input_datetime`, verify `has_time` and whether a date is required; active windows are time-only while `ev_plan_last_updated` has date and time.

The implementation does not establish a general rule for defaults of future helpers. Do not infer a default from runtime telemetry or a dashboard value.

## 11. Adding or Changing a Sensor

Use templates for derived decisions and runtime presentation. Keep each sensor's responsibility narrow.

| Sensor type | Current pattern | Allowed dependencies |
|---|---|---|
| Recommendation binary sensor | Tests current time against a selected block. | Selected-block helper and current time. |
| Combined recommendation | Logical OR of day and night recommendations. | Existing recommendation binary sensors. |
| Runtime sensor | Derives a value from OCPP status or attributes. | OCPP sensor/entity state and attributes. |
| Price-availability binary sensor | Validates Nordpool next-day data. | `tomorrow_valid` and `raw_tomorrow`. |

When changing a template:

1. Identify producer entities and the sensor's single responsibility.
2. Verify entity IDs and attributes against the current implementation.
3. Preserve day/night separation.
4. Keep time-block parsing in the recommendation layer, not in a dashboard card or execution automation.
5. Use only defensive behavior established by the relevant pattern. Existing templates use constructs such as `float(0)`, `or []`, a `none` plan output, and an explicit `–` check before selected-block parsing.
6. Verify valid values, `unknown`, `unavailable`, empty values, and malformed selected-block text before connecting the sensor to execution.

The project has no general state-versus-attribute policy. Existing use is concrete: Nordpool raw price lists are attributes; selected blocks, derived current, and recommendation values are entity states. Do not invent attributes where consumers expect state.

## 12. Changing Planner Logic

`EV Build Charge Plan` is the sole active planner. It runs at 23:00 and uses:

- `sensor.nordpool_kwh_se3_sek_3_10_025`
- `raw_tomorrow`
- `tomorrow_valid`
- day/night energy targets
- day/night charge currents
- day/night time-window helpers

Effective charging power is calculated as:

```text
(current - 1) × 230 / 1000 kW
```

with a minimum of 0.1 kW. Required 15-minute slots are calculated from target energy and effective power. Candidate slots are sorted by timestamp, and the continuous candidate with the lowest summed price is selected. The planner writes `HH:MM–HH:MM` or `none` to the selected-block helpers and updates `ev_plan_last_updated`.

### Why the planner is built once from next-day data

The active planner uses next-day data at 23:00. Its source set therefore represents the next calendar day, avoiding the former live-planner behavior in which a future-only candidate window moved forward as time passed. The current implementation has no separate current-time filter for passed slots because its normal planning input is already the next day's list.

Use this change process:

1. Identify the authoritative helper or Nordpool attribute for the calculation.
2. Change the calculation in `EV Build Charge Plan` only.
3. Keep distinct day and night paths and outputs.
4. Check dependent recommendation templates, `EV Charge Start`, `EV Charge Stop`, and dashboard consumers.
5. Preserve the `HH:MM–HH:MM` format expected by consumers.
6. Trigger or wait for a controlled plan build and inspect blocks before testing execution.

Do not add a current-time passed-slot filter or describe one as present without an explicit approved change. Historical `EV Day Lock Selected Charge Block` and `EV Night Lock Selected Charge Block` automations are not part of the active flow.

## 13. Changing Execution Automations

| Automation | Trigger and responsibility | Safety-critical behavior |
|---|---|---|
| `EV Charge Start` | Combined recommendation changes to `on`; smart charging must be enabled. | Choose active day/night current, write `input_number.ev_max_charge_current`, wait 5 seconds, then turn on availability. |
| `Set Halo charge current` | `input_number.ev_max_charge_current` changes. | Call `ocpp.set_charge_rate` with `devid: charger` and `limit_amps` from the helper. |
| `EV Charge Stop` | Combined recommendation changes to `off`, or active day recommendation reaches day target. | Smart charging and availability must be on; turn charge control off, wait 1 minute, then turn availability off. |
| `EV Charger Fail Safe On Startup` | Home Assistant start. | After 2 minutes 30 seconds, turn charge control off, wait up to 30 seconds for non-`Charging` status, then turn availability off. |

`EV Charge Stop` restores day and night target/current helpers to defaults under its implemented day-related condition and sets `input_number.ev_max_charge_current` to the default night current after every stop execution.

### Why start uses current before availability

`input_number.ev_max_charge_current` is not the charger command itself. It triggers `Set Halo charge current`, which sends the OCPP charge-rate command. The five-second wait intentionally separates current selection from enabling availability so a connected vehicle does not begin charging before the intended day/night limit has had time to reach the charger.

### Why stop uses charge control before availability

The verified charger behavior distinguishes starting permission from stopping an active transaction:

- availability is the gate used to permit a new charging session;
- charge control is used to stop an active transaction;
- availability must not be treated as the first or sole active-charge stop mechanism.

Therefore the established stop sequence is:

```text
charge control OFF
        |
        v
wait one minute
        |
        v
availability OFF
```

The delay allows the charger and vehicle transaction state to settle before availability is withdrawn. This sequence also applies at block end when the charger reports `SuspendedEV` or another active-transaction status.

Do not reverse these operations. Do not remove, shorten, or bypass the delays without an explicit design decision supported by controlled testing.

## 14. OCPP Integration Rules

The configured OCPP Central System has charge-point ID `charger`, one connector, listens on `<HOME_ASSISTANT_IP>:9000`, and has TLS disabled. The active charge-rate call is:

```yaml
service: ocpp.set_charge_rate
data:
  devid: charger
  limit_amps: "{{ states('input_number.ev_max_charge_current') | int }}"
```

| Control or observation | Meaning in this project |
|---|---|
| `switch.ev_charger_available` | Application abstraction of `switch.charger_availability`; enables or prevents a new charge. |
| `switch.ev_charge_control` | Application abstraction of `switch.charger_connector_1_charge_control`; stops an active transaction. |
| `input_number.ev_max_charge_current` | Input to the charge-rate automation. |
| `sensor.ev_session_energy` | Session-energy observation for connector 1. |
| Connector status and current entities | OCPP observations used by runtime templates. |

Normal automated start is availability-based. With a connected vehicle, making the charger available starts charging; the start automation does not explicitly enable charge control. At block end, charge control is disabled before availability.

No verified application-level retry, command timeout, reconnect, or rejected-command handling exists in the execution automations. Do not represent such behavior as present. Intended behavior after lost OCPP connectivity or a rejected command remains an open question.

## 15. Dashboard Changes

The storage-based dashboard is `.storage/lovelace.ev_energy`. It displays planner status, smart-charging state, targets, selected blocks, estimated-cost entities, recommendation state, charger controls, price availability, settings, and an ApexCharts price graph.

The existing manual action triggers `automation.ev_build_charge_plan`. Settings and status cards show existing helpers and sensors. The chart reads Nordpool data and selected-block states to highlight planned periods.

When changing a dashboard field:

1. Confirm that the entity already has an owner outside the dashboard.
2. Reference the entity ID directly.
3. Use selected-block entities for plan display; do not reimplement time or price logic in the card.
4. Preserve the relationship between Nordpool raw price data and selected day/night block values.
5. Verify that the visual value agrees with the rendered entity state.

### Why logic stays out of the dashboard

Dashboard cards are a presentation and interaction surface. Their rendering, refresh behavior, or availability must not determine whether the charger starts or stops. Keeping decisions in planner/templates and actions in automations makes charging behavior independent of whether any dashboard is open.

## 16. Error Handling and Defensive Logic

| Situation | Verified current behavior | Status |
|---|---|---|
| Missing or invalid next-day prices | Planner creates a persistent notification and stops. | Implemented. |
| Fewer than 96 next-day entries | Next-day price availability is false; plan build is aborted. | Implemented. |
| Empty raw price list | Planner uses `raw_tomorrow or []`; if no candidate block is found, output is `none`. | Implemented. |
| Missing numeric helper state | Planner uses numeric defaults and enforces a 0.1 kW minimum power. | Implemented in planner templates. |
| Invalid or empty selected-block format | Recommendation logic checks for `–` before parsing. | Implemented. |
| Block crossing midnight | Night recommendation logic handles a crossing block. | Implemented. |
| Home Assistant restart | Startup failsafe turns charge control and availability off after configured waits. | Implemented. |
| Active or `SuspendedEV` transaction at block end | Charge control is turned off before availability. | Implemented design behavior. |
| Invalid window or insufficient contiguous slots | No explicit user-facing validation beyond no matching candidate and `none` output is documented. | Limited/unclear. |
| OCPP disconnected, command rejected, or charger does not accept command | No verified handling in active execution automations. | Open. |

## 17. Testing and Verification

Run tests in a controlled charging context. Observe entity state, automation traces, logbook entries where applicable, OCPP state, and the EV Energy dashboard.

| Test | Preconditions | Procedure | Expected result | Observe |
|---|---|---|---|---|
| Configuration control | Relevant configuration is loaded. | Validate changed YAML/template configuration before reload. | No syntax or entity-reference error. | Configuration validation result. |
| Template rendering | Source helpers and OCPP entities have known states. | Render changed templates with normal values. | State matches documented calculation or decision. | Template result and entity state. |
| Next-day plan build | `tomorrow_valid` is true and `raw_tomorrow` has at least 96 entries. | Trigger or wait for `EV Build Charge Plan`. | Day/night blocks are written as `HH:MM–HH:MM` when candidates exist; timestamp updates. | Trace, selected blocks, timestamp. |
| Missing next-day prices | Tomorrow data is invalid or incomplete. | Trigger plan builder. | Persistent notification is created and no new plan is built. | Notification and trace. |
| Day planning | Valid next-day data and configured day inputs. | Build a plan. | Continuous block inside day window using day inputs. | Day block and recommendation. |
| Night planning | Valid next-day data and configured night inputs. | Build a plan. | Continuous block inside night window using night inputs. | Night block and recommendation. |
| Passed-slot scope | Normal 23:00 planning time with next-day data. | Inspect planner inputs and output. | Candidate input is `raw_tomorrow`, representing the next calendar day. | Trace and price attribute. |
| Start at selected time | Smart charging on; recommendation changes to on. | Allow selected block to begin. | Current selected; charge-rate automation runs; availability on after 5 seconds. | Traces, max current, availability, OCPP state. |
| Stop at selected end | Smart charging and availability on; recommendation changes to off. | Allow block to end. | Charge control off; availability off after 1 minute. | Trace and switch states. |
| Day-target stop | Smart charging and availability on; day recommendation on. | In a controlled test, reach day target. | Stop automation follows documented sequence. | Trace, session energy, switches, resets. |
| Charge-current change | Controlled test permits change. | Change `ev_max_charge_current` through its normal source. | OCPP call uses device `charger` and helper value. | Trace and OCPP state. |
| Home Assistant restart | Controlled maintenance window. | Restart Home Assistant. | Failsafe applies documented delays and disables controls. | Failsafe trace and states. |
| Manual charging | Controlled charger/vehicle state. | Exercise manual controls without changing planner logic. | Actual behavior appears through OCPP and derived entities. | Availability, charge control, status, current, energy. |
| Empty/unknown/unavailable input | Controlled non-production values available. | Render templates with invalid/empty states. | Explicit fallbacks behave as configured; uncovered behavior is recorded as unclear. | Template results and trace. |
| OCPP loss or rejected command | Only where safely reproducible. | Observe active automation and entities. | No application-level retry/recovery is expected because none is documented. | OCPP state, trace, logs. |

## 18. Safe Change Procedure

1. Back up current Home Assistant configuration and storage-backed configuration.
2. Identify affected entities, templates, automations, dashboard cards, and OCPP dependencies.
3. Read the relevant design decision and active architectural flow.
4. Change the smallest possible component area.
5. Validate YAML and render affected templates.
6. Reload only the relevant configuration where supported; otherwise use the controlled restart required by the change.
7. Verify entity state and automation traces.
8. Perform the controlled functional test matching the changed responsibility.
9. Verify startup failsafe and manual charger function when charger control may be affected.
10. Update applicable documentation.

## 19. Change Documentation

| Change type | Documentation update |
|---|---|
| Component, flow, integration, or boundary change | Update `Architecture.md`. |
| Verified design decision or changed rationale/consequence | Update `Design-Decisions.md`. |
| Change to concise project model, entities, rules, or terminology used by AI sessions | Update `AI_CONTEXT.md`. |
| Change to maintenance procedures, structure, tests, or safety process | Update `Developer-Guide.md`. |
| New uncertainty, unresolved behavior, or no-longer-verified assumption | Update `assumptions.md`. |
| Assumption confirmed by owner or implementation evidence | Reclassify/remove it in `assumptions.md` and update the document recording the verified rule. |
| Configuration-only change | Update documentation only if behavior, ownership, safety, or operations change. |
| Defect correction | Update architecture/design only if documented behavior changes or uncertainty is resolved. |
| `CHANGELOG.md` | No current file is documented. If explicitly introduced, record user-visible behavior and implementation changes. |

## 20. Common Mistakes to Avoid

- Do not duplicate continuous-block selection in execution automation, dashboard code, or historical lock automation.
- Do not hard-code a target, current, or window value that already has an active helper.
- Do not use the dashboard as a business-logic layer.
- Do not turn availability off before charge control has stopped an active transaction.
- Do not assume availability-off alone stops active charging.
- Do not treat the day target as coordinated with the night target.
- Do not mix day/night selected blocks or recommendation entities.
- Do not overwrite selected-block helpers from execution logic.
- Do not assume Nordpool or OCPP entities are always available.
- Do not replace next-day planning input with current-day data without an approved planner change.
- Do not change entity IDs without updating every template, automation, dashboard, and service reference.
- Do not remove or bypass the startup failsafe.
- Do not treat historical planners or lock automations as active requirements.

## 21. AI Development Instructions

Before proposing or applying a change, an AI assistant must:

1. Read `AI_CONTEXT.md`, `Architecture.md`, `Design-Decisions.md`, and `assumptions.md`.
2. Treat Phase 2 as authoritative if sources conflict; report unresolved conflicts.
3. Verify every entity ID, automation alias, service, attribute, and file location against the current implementation.
4. Separate verified facts, implementation observations, and assumptions.
5. Never invent entities, helpers, integrations, services, scripts, error handling, or charger behavior.
6. Do not restructure the system or move responsibility across layers without explicit direction.
7. Keep changes small and reviewable; show complete YAML for each changed component.
8. Check Home Assistant indentation and template syntax.
9. State affected files, entities, dependencies, and expected runtime behavior.
10. Provide matching verification steps and update documentation when behavior or design changes.

## 22. Open Questions

| Question | Why the answer is needed | Affected area | How to verify |
|---|---|---|---|
| Are configuration helpers the mandatory persistent source of truth for user settings, with runtime values treated only as temporary state? | Determines how future changes classify and write settings versus runtime data. | Helpers, automations, templates, documentation. | Confirm with project owner. |
| Are simple YAML, avoidance of Python, and avoidance of Node-RED mandatory constraints or historical advice? | Determines whether implementation technology choices are constrained. | Future implementation approach. | Confirm with project owner. |
| What is the intended application response to lost OCPP connectivity, rejected commands, and charger-command timeouts? | No verified retry or recovery exists in active execution automations. | OCPP boundary and execution safety. | Review intended control policy and validate in a controlled test. |
| What behavior is required for invalid windows or insufficient continuous price slots beyond writing `none`? | Current planner can output `none`, but broader operator handling is undocumented. | Planner output, dashboard, operator workflow. | Confirm intended behavior and observe controlled planner runs. |
