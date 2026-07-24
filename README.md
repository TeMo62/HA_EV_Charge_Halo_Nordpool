# HA EV Charge – Halo & Nord Pool

V1 baseline of the Home Assistant solution that was in active operation on
2026-07-22. It plans separate day and night charging sessions using Nord Pool
15-minute prices and controls a Charge Amps Halo locally through OCPP.

## V1 features

- cheapest continuous charging block within separate day and night windows
- energy target and charging current for each session
- OCPP current limit
- start through `availability`
- stop in the Halo-safe order: `charge_control`, 60 seconds, `availability`
- restore default values after the day session
- estimated cost and EV dashboard
- manual 90-minute evening top-up
- manual stop using the Halo-safe shutdown order

See [dashboard screenshots and descriptions](docs/dashboard.md) for the status,
price chart, and settings views.

## Files

```text
automations/ev_charging.yaml        active automations
config/ev_helpers.yaml              helpers and v1 initial values
config/ev_templates.yaml            OCPP wrappers and active charging window
config/ev_cost_templates.yaml       estimated cost
scripts/ev_restore_default_values.yaml
scripts/ev_manual_charging.yaml     evening top-up and manual stop
dashboards/ev_energy.yaml           dashboard export
docs/dashboard.md
docs/installation.md
docs/operations.md
docs/v1-baseline.md
```

## Important

V1 intentionally has no automatic restart fail-safe. A connected vehicle may
start charging when Home Assistant restarts. See [operations and
restart](docs/operations.md).

No secrets, tokens, databases, or raw `.storage` files are included.

## Feedback

Questions, suggestions, and reports from similar installations are welcome
through [GitHub Issues](https://github.com/TeMo62/HA_EV_Charge_Halo_Nordpool/issues).

## License

This project is licensed under the [MIT License](LICENSE).
