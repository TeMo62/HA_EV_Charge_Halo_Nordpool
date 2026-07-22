# HA EV Charge – Halo & Nord Pool

V1-baslinje för den Home Assistant-lösning som var i aktiv drift
2026-07-22. Den planerar separata dag- och nattpass från Nord Pools
15-minuterspriser och styr en Charge Amps Halo lokalt via OCPP.

## V1 omfattar

- billigaste sammanhängande laddblock inom separata dag- och nattfönster
- energimål och laddström per pass
- OCPP-strömgräns
- start via `availability`
- stopp i Halo-säker ordning: `charge_control`, 60 sekunder, `availability`
- återställning av standardvärden efter dagspasset
- uppskattad kostnad och EV-dashboard

## Filer

```text
automations/ev_charging.yaml        aktiva automationer
config/ev_helpers.yaml              helpers och v1-startvärden
config/ev_templates.yaml            OCPP-wrappers och aktivt laddfönster
config/ev_cost_templates.yaml       uppskattad kostnad
scripts/ev_restore_default_values.yaml
dashboards/ev_energy.json           dashboardexport
docs/installation.md
docs/operations.md
docs/v1-baseline.md
```

Den äldre Charge Amps-blueprinten i `blueprints/` är ett historiskt första
försök och ingår inte i v1-flödet.

## Viktigt

V1 har avsiktligt ingen automatisk fail-safe vid omstart. En ansluten bil kan
börja ladda när Home Assistant startas om. Se [drift och
omstart](docs/operations.md).

Inga hemligheter, tokens, databaser eller råa `.storage`-filer ingår.
