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

## Files

```text
automations/ev_charging.yaml        active automations
config/ev_helpers.yaml              helpers and v1 initial values
config/ev_templates.yaml            OCPP wrappers and active charging window
config/ev_cost_templates.yaml       estimated cost
scripts/ev_restore_default_values.yaml
dashboards/ev_energy.json           dashboard export
docs/installation.md
docs/operations.md
docs/v1-baseline.md
```

The older Charge Amps blueprint in `blueprints/` is an early historical
attempt and is not part of the v1 flow.

## Important

V1 intentionally has no automatic restart fail-safe. A connected vehicle may
start charging when Home Assistant restarts. See [operations and
restart](docs/operations.md).

No secrets, tokens, databases, or raw `.storage` files are included.
