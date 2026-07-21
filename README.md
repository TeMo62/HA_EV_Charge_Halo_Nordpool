# HA EV Charge – Halo & Nord Pool

Home Assistant-projekt för att styra elbilsladdning med en Halo-laddbox utifrån elpriser från Nord Pool.

## Status

Projektet är under utveckling. Dokumentation, konfiguration och automationer läggs till och verifieras stegvis.

## Mål

- Hämta och använda aktuella elpriser från Nord Pool i Home Assistant.
- Planera laddning till prisvärda timmar.
- Styra Halo-laddboxen på ett säkert och förutsägbart sätt.
- Ge tydliga inställningar, status och möjlighet till manuell överstyrning.
- Dokumentera installation, konfiguration och felsökning.

## Planerad struktur

```text
config/       Home Assistant-konfiguration och hjälpare
automations/  Automationer för laddstyrning
scripts/      Återanvändbara Home Assistant-skript
docs/         Installation, konfiguration och felsökning
tests/        Testfall och exempeldata där det är möjligt
```

## Säkerhet

Laddning ska alltid begränsas av laddboxens, bilens och elanläggningens tillåtna värden. Lägg aldrig lösenord, API-nycklar, tokens eller andra hemligheter i repot.

## Licens

Ingen licens har valts ännu. Tills en licensfil läggs till gäller vanliga upphovsrättsregler.
