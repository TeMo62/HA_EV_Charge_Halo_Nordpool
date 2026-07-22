# V1-baslinje 2026-07-22

Baslinjen är framtagen ur en ZIP-export av den aktiva Home Assistant-
konfigurationen och de separat exporterade GUI-template-posterna.

## Medtaget

- `EV Build Charge Plan`
- `EV Charge Start`
- `EV Charge Stop`
- `Set Halo charge current`
- `EV Restore Defaults After Day Charging Window`
- skriptet `EV Restore Default Values`
- aktiva OCPP-wrapperentiteter
- dag-/nattindikering och kostnadssensorer
- relevanta helpers och EV-dashboarden

Standardvärdena i exporten var 10 kWh och 10 A för både dag och natt.
Tidsfönstren var 10:00–21:00 respektive 00:15–08:00. De avvikande aktiva
värdena i exportögonblicket kom från prov av återställningsknappen och används
inte som installationsstandard.

## Avsiktligt utelämnat

- de avstängda fasta nattautomationerna 00:30/06:00
- båda gamla `Lock Selected Charge Block`-automationerna
- testautomationens loggbokspost
- den felplacerade och icke registrerade omstarts-fail-safe-konfigurationen
- gamla beräknings- och visualiseringstemplates
- databaser, loggar, tokens, integrationernas config entries och övrig HA-data

Följande gamla templates ingår inte:

- `ev_requiered_charging_hours`
- `ev_cheapest_charging_hours`
- `ev_cheapest_charge_slots`
- `ev_estimated_charge_cost`
- `EV_Cheapest_Continuous_Charge_Block`
- de fyra gamla `*_visual*`-sensorerna

## Kända v1-begränsningar

- En plan räknas inte om automatiskt när mål eller ampere ändras.
- Planbyggaren körs normalt endast 23:00.
- Omstart kräver manuell kontroll av laddaren.
- `sensor.sensor_ev_session_energy` har ett historiskt dubbelt `sensor_` i
  entity-ID:t och behålls för kompatibilitet med den aktiva automationen.
