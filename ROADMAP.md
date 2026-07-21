# Roadmap

This roadmap records planned improvements and design wishes. Items are not
committed release dates, and safety-related changes require verification on the
actual charger and electrical installation.

## Current baseline

Phase 2 supports vehicles that do not report state of charge to Home Assistant.
The user supplies estimated day and night energy targets. The planner selects
the lowest-cost continuous quarter-hour block that can deliver each estimate
inside its configured window.

## Near-term priorities

- Migrate the existing UI-managed entities to the packaged configuration
  without creating duplicate entity IDs.
- Test the packaged startup fail-safe after a Home Assistant restart on the
  actual charger.
- Add configuration validation for missing entities, invalid time windows,
  unavailable next-day prices, and current settings above the approved limit.
- Add deterministic planner tests using captured 96-slot Nord Pool datasets.
- Document a controlled commissioning and rollback procedure.

## Dynamic household load management

Monitor total and per-phase household consumption and reduce EV charging
current when the installation approaches its safe limit.

The design should include:

- configurable main-fuse and per-phase limits;
- a safety margin and hysteresis to avoid rapid current changes;
- gradual current ramp-up after household load falls;
- an explicit minimum usable EV current;
- fail-safe reduction or suspension when house telemetry is unavailable;
- OCPP current updates through the existing current-control boundary;
- tests for large household loads, sensor loss, phase imbalance, and recovery.

This feature must complement, not replace, certified electrical load balancing.

## Vehicle SoC and vehicle profiles

When a future vehicle exposes SoC to Home Assistant, derive energy demand from
the vehicle instead of relying only on a best estimate.

A proposed calculation is:

```text
required_kWh = battery_capacity_kWh
               × (target_SoC - current_SoC)
               ÷ charging_efficiency
```

The implementation should support:

- per-vehicle battery capacity and charging-efficiency profiles;
- target SoC and departure time;
- validation of stale or unavailable SoC data;
- a configurable reserve or uncertainty margin;
- automatic fallback to the existing manual kWh estimate;
- clear dashboard indication of whether demand came from vehicle SoC or manual
  input;
- safe behaviour when changing vehicles.

## Additional candidates

- Include grid fees, taxes, and other configurable price components.
- Record planned versus delivered energy and use the difference to improve
  future estimates.
- Add notifications for missing prices, failed OCPP commands, missed targets,
  and fallback operation.
- Display house load, EV limit, selected blocks, delivered energy, and plan
  provenance on the dashboard.
- Support reusable vehicle profiles without coupling the planner to one vehicle
  integration.
