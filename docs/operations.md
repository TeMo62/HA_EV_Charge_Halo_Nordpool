# Drift och omstart

## Normal drift

Planen byggs klockan 23:00 när morgondagens 96 kvartpriser finns. Ett
sammanhängande block väljs separat inom dag- och nattfönstret. Vid blockets
start sätts rätt strömgräns via OCPP och laddaren görs tillgänglig.

Vid stopp stängs först `switch.ev_charge_control`. Efter 60 sekunder stängs
`switch.ev_charger_available`. Ordningen är viktig eftersom Halo normalt inte
accepterar att availability stängs av medan laddning pågår.

## Omstart av Home Assistant

V1 har avsiktligt ingen automatisk fail-safe vid omstart. OCPP-integrationen
och laddarens entiteter blir tillgängliga först en stund efter Home Assistant,
vilket gör en fast tidsfördröjning opålitlig.

Om bilen är ansluten kan laddningen därför starta vid en omstart. Kontrollera
laddaren manuellt när OCPP åter är tillgängligt. Om laddningen ska stoppas:

1. Stäng av `switch.ev_charge_control`.
2. Vänta tills laddningen har upphört.
3. Stäng av `switch.ev_charger_available`.

## Ändringar efter byggd plan

Ändring av energimål, ström eller tidsfönster räknar inte automatiskt om en
redan byggd plan. Kör `automation.ev_build_charge_plan` manuellt efter en sådan
ändring om den ska gälla nästa pass.
