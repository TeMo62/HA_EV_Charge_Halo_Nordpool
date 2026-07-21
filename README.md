# HA EV Charge – Halo & Nord Pool

Home Assistant project for planning and controlling a Charge Amps Halo over
OCPP using Nord Pool electricity prices.

## Scope

The current design is intended for vehicles that do **not** expose state of
charge (SoC) to Home Assistant. The operator supplies estimated day and night
energy requirements in kWh. The planner converts each target into a required
duration and selects the lowest-cost continuous 15-minute block inside the
configured charging window.

This is deliberately a best-estimate model. Vehicle-derived energy demand is
listed as a future enhancement in the [roadmap](ROADMAP.md).

## Current architecture

Phase 2 uses:

- separate day and night plans;
- continuous 15-minute Nord Pool price intervals;
- user-defined energy targets, currents, and time windows;
- OCPP availability to permit a new charging session;
- OCPP charge control to stop an active transaction;
- a current-first start sequence and a charge-control-first stop sequence;
- an active Home Assistant startup fail-safe.

The deployable package is in
[`home-assistant/packages/ev_charging_planner.yaml`](home-assistant/packages/ev_charging_planner.yaml).
The complete architecture and implementation reference are available under
[`docs/design-reference`](docs/design-reference/AI_CONTEXT.md).

The repository also contains an early price-threshold blueprint. It is a
prototype and is not part of the active Phase 2 production architecture.

## Goals

- Plan charging against Nord Pool SE3 quarter-hour prices.
- Complete the requested energy delivery inside configured day and night
  windows.
- Keep charging current within the installation's configured limit.
- Preserve predictable manual control and safe failure behaviour.
- Document installation, operation, verification, and future development.

## Repository layout

```text
blueprints/             Early prototype automation
home-assistant/         Deployable Phase 2 package and migration notes
docs/installation.md    OCPP and Nord Pool setup overview
docs/design-reference/  Phase 2 architecture and implementation source
ROADMAP.md               Planned improvements and wishlist
```

## Safety

The charger, vehicle, cable, circuit breaker, and electrical installation must
independently enforce their permitted limits. This project does not replace
certified electrical protection or load balancing. Never commit passwords,
tokens, API keys, certificates, exact private-network details, or
`secrets.yaml`.

## License

No license has been selected yet. Until a license file is added, normal
copyright rules apply.
