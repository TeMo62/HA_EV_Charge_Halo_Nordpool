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

## Verified additions after the baseline

The following additions were verified in the active installation on
2026-07-24:

- live current, session energy, and charger status on the dashboard
- a manual 90-minute evening top-up
- a manual Halo-safe stop button
- a YAML dashboard export for easier reuse and maintenance

## Known v1 limitations

- A plan is not recalculated automatically when a target or current is changed.
- The plan builder normally runs only at 23:00.
- A restart requires a manual charger check.
- A restart during an evening top-up cancels its delayed automatic stop.
- `sensor.sensor_ev_session_energy` has a historical doubled `sensor_` in
  its entity ID and is retained for compatibility with the active automation.
