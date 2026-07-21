# Home Assistant package

`packages/ev_charging_planner.yaml` is the deployable Phase 2 implementation. It creates the helpers, templates, script and automations used by the planner.

## Important migration note

Do not install the complete package into a Home Assistant instance that already contains entities with the same IDs. The current development instance has several helpers and template entities created through the UI, so those must be removed or migrated first. Until that migration is completed, treat this package as the canonical source and update the existing UI entities individually.

## Prerequisites

- Home Assistant with package support enabled
- Nord Pool sensor `sensor.nordpool_kwh_se3_sek_3_10_025` with 15-minute `raw_tomorrow` data
- HACS OCPP integration exposing the charger entities listed in the package header
- A single-phase charging installation limited to 10 A

Enable packages in `configuration.yaml` if required:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Copy the package into the configured `packages` directory, validate the Home Assistant configuration, and restart Home Assistant. Leave smart charging disabled until all prerequisite entity IDs have been verified.

The package is intended for vehicles that do not expose state of charge to Home Assistant. Energy targets are therefore manual estimates in kWh.
