# Installation overview

This document describes the integrations and runtime assumptions used by the
Phase 2 system. The repository does not yet provide a complete one-click
installation package; exporting the verified runtime configuration is the
first item in the [roadmap](../ROADMAP.md).

## Prerequisites

- Home Assistant with HACS installed.
- A Charge Amps Halo configured for OCPP operation.
- The HACS OCPP integration.
- A Nord Pool integration that exposes SE3 quarter-hour prices in SEK,
  including `raw_today`, `raw_tomorrow`, and `tomorrow_valid`.
- Network connectivity between the Halo and the Home Assistant OCPP endpoint.

Never store credentials, tokens, certificates, exact private-network details,
or `secrets.yaml` in this repository.

## 1. Install OCPP through HACS

1. Open **HACS** in Home Assistant.
2. Select **Integrations**.
3. Search for **OCPP** and install the integration.
4. Restart Home Assistant when HACS requests it.
5. Go to **Settings → Devices & services → Add integration**.
6. Search for **OCPP** and create the central-system configuration.

The verified Phase 2 model uses charge-point ID `charger`, one connector, port
`9000`, and no TLS on the trusted local network. Keep the Home Assistant host
address private and configure it locally rather than committing it.

Configure the Halo, using the applicable Charge Amps instructions, to connect
to the Home Assistant OCPP WebSocket endpoint. Confirm that Home Assistant
creates connector status, current, energy, availability, and charge-control
entities before enabling automation.

Expected underlying control entities include:

- `switch.charger_availability`
- `switch.charger_connector_1_charge_control`
- `number.charger_connector_1_maximum_current`
- `sensor.charger_connector_1_energy_session`
- `sensor.charger_connector_1_status_connector`

Entity IDs can vary. Verify every ID in **Developer Tools → States**.

## 2. Configure Nord Pool

Configure Nord Pool for region **SE3**, currency **SEK**, kWh pricing, and
quarter-hour data. The current installation uses:

`sensor.nordpool_kwh_se3_sek_3_10_025`

Before building a plan, verify that:

- `tomorrow_valid` is `true`;
- `raw_tomorrow` contains at least 96 entries;
- each entry has `start`, `end`, and `value` fields;
- the unit is `SEK/kWh`.

The planner runs at 23:00 and creates one continuous lowest-cost block for the
day window and one for the night window.

## 3. Verify the Phase 2 control boundary

Phase 2 uses template facades in front of the OCPP entities:

- `switch.ev_charger_available` controls whether a new charge may start.
- `switch.ev_charge_control` stops an active connector transaction.

Normal start order:

1. Apply the selected current limit.
2. Wait five seconds.
3. Enable charger availability.

Normal stop order:

1. Disable connector charge control.
2. Wait one minute.
3. Disable charger availability.

Do not reverse these sequences without a verified charger test.

## 4. Current limit

The current electrical installation is limited to **10 A** even though other
parts of the charging equipment may support 16 A. Home Assistant and OCPP must
not request more than the currently approved installation limit.

## 5. Initial verification

Perform tests with the vehicle disconnected first:

1. Confirm all referenced entities are available.
2. Build a plan manually and inspect the selected day and night blocks.
3. Verify the current-setting automation without exceeding 10 A.
4. Verify availability on/off behaviour.
5. Verify charge-control off behaviour during a controlled session.
6. Restart Home Assistant and confirm the startup fail-safe executes.
7. Review automation traces before unattended use.

The Phase 2 implementation details, helpers, templates, automations, and known
historical artifacts are documented in
[`design-reference/Implementation-Reference.md`](design-reference/Implementation-Reference.md).
