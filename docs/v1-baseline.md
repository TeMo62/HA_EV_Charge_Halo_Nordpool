# V1 baseline – 2026-07-22

The baseline was created from a ZIP export of the active Home Assistant
configuration and separately exported GUI template entries.

## Included

- `EV Build Charge Plan`
- `EV Charge Start`
- `EV Charge Stop`
- `Set Halo charge current`
- `EV Restore Defaults After Day Charging Window`
- the `EV Restore Default Values` script
- active OCPP wrapper entities
- day and night indicators and cost sensors
- relevant helpers and the EV dashboard

The exported default values were 10 kWh and 10 A for both day and night. The
time windows were 10:00–21:00 and 00:15–08:00 respectively. The different
active values at the time of export resulted from testing the restore-defaults
button and are not used as installation defaults.

## Intentionally omitted

- the disabled fixed night automations at 00:30 and 06:00
- both old `Lock Selected Charge Block` automations
- the test automation's logbook entry
- the misplaced and unregistered restart fail-safe configuration
- old calculation and visualization templates
- databases, logs, tokens, integration config entries, and other Home Assistant
  data

The following old templates are not included:

- `ev_requiered_charging_hours`
- `ev_cheapest_charging_hours`
- `ev_cheapest_charge_slots`
- `ev_estimated_charge_cost`
- `EV_Cheapest_Continuous_Charge_Block`
- the four old `*_visual*` sensors

## Known v1 limitations

- A plan is not recalculated automatically when a target or current is changed.
- The plan builder normally runs only at 23:00.
- A restart requires a manual charger check.
- `sensor.sensor_ev_session_energy` has a historical doubled `sensor_` in
  its entity ID and is retained for compatibility with the active automation.
