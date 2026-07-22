# Installation and recovery

This repository is primarily a reproducible baseline of the working
installation. Do not add the YAML entities on top of the same GUI-created
helpers and templates in the existing installation, as this will create
duplicate entity IDs.

## Dependencies

- Home Assistant with the OCPP integration connected to the Halo
- Nord Pool sensor with `raw_today`, `raw_tomorrow`, and `tomorrow_valid`
- 15-minute prices, normally 96 entries per day
- ApexCharts Card through HACS for the exported dashboard chart

The baseline uses the following installation-specific sources:

- `sensor.nordpool_kwh_se3_sek_3_10_025`
- `switch.charger_availability`
- `switch.charger_connector_1_charge_control`
- `sensor.charger_connector_1_status_connector`
- `sensor.charger_connector_1_current_import`
- `sensor.charger_connector_1_energy_session`
- OCPP device ID `charger`

Replace these names in the files if the integrations create different entity
IDs.

## Existing installation

The current Home Assistant installation already uses GUI-created helpers and
templates. Use the repository files as a version-controlled reference until a
planned migration is performed. The automations can be compared with or
restored to `automations.yaml`, and the script to `scripts.yaml`.

Remove the misplaced `EV Charger Fail Safe On Startup` block from
`configuration.yaml` during the next planned cleanup. It is not registered as
an automation and should not be moved to `automations.yaml`.

## Clean installation

1. Create the helpers from `config/ev_helpers.yaml` or their GUI equivalents.
2. Add the template entities from both template files in `config/`.
3. Add the script to `scripts.yaml` and the automations to
   `automations.yaml`.
4. Validate the configuration before restarting.
5. Import `dashboards/ev_energy.json` or recreate the cards manually.
6. Verify all entity IDs while smart charging is disabled.
7. Test start and stop first without a vehicle connected.
8. Enable `input_boolean.ev_smart_charging_enabled` only after testing is
   complete.

After changing a target, current, or time window, rebuild the plan manually if
the change should affect an already selected block.
