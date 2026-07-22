# Installation och återställning

Detta är i första hand en reproducerbar baslinje av den fungerande
installationen. Lägg inte YAML-entiteterna ovanpå samma GUI-skapade helpers och
templates i den befintliga installationen; då uppstår dubbla entity-ID:n.

## Beroenden

- Home Assistant med OCPP-integrationen ansluten till Halo
- Nord Pool-sensor med `raw_today`, `raw_tomorrow` och `tomorrow_valid`
- 15-minuterspriser, normalt 96 poster per dygn
- ApexCharts Card via HACS för den exporterade dashboardgrafen

Baslinjen använder följande installationsspecifika källor:

- `sensor.nordpool_kwh_se3_sek_3_10_025`
- `switch.charger_availability`
- `switch.charger_connector_1_charge_control`
- `sensor.charger_connector_1_status_connector`
- `sensor.charger_connector_1_current_import`
- `sensor.charger_connector_1_energy_session`
- OCPP-enhets-ID `charger`

Byt dessa namn i filerna om integrationerna skapar andra entity-ID:n.

## Befintlig installation

Den nuvarande HA-installationen kör redan GUI-skapade helpers och templates.
Använd därför repo-filerna som versionshanterad referens tills en planerad
migrering görs. Automationerna kan jämföras med eller återställas till
`automations.yaml`, och skriptet med `scripts.yaml`.

Ta bort det felplacerade blocket `EV Charger Fail Safe On Startup` ur
`configuration.yaml` vid nästa planerade städning. Det registreras inte som en
automation och ska inte flyttas till `automations.yaml`.

## Ren installation

1. Skapa helpers från `config/ev_helpers.yaml` eller motsvarande i GUI.
2. Lägg till template-entiteterna från båda filerna i `config/`.
3. Lägg skriptet i `scripts.yaml` och automationerna i `automations.yaml`.
4. Kontrollera konfigurationen innan omstart.
5. Importera `dashboards/ev_energy.json` eller återskapa korten manuellt.
6. Kontrollera alla entity-ID:n med smart charging avstängd.
7. Testa först start och stopp utan ansluten bil.
8. Aktivera `input_boolean.ev_smart_charging_enabled` först när testet är klart.

Efter ändring av mål, ampere eller tidsfönster måste planen byggas om manuellt
om ändringen ska påverka ett redan valt block.
