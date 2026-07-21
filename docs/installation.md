# Installation

## Förutsättningar

- Home Assistant 2024.12 eller senare.
- Den officiella Nord Pool-integrationen konfigurerad för rätt elområde och SEK.
- HACS installerat.
- Chargeamps-integrationen installerad och ansluten till din Halo.
- En `input_boolean` som används som huvudbrytare för automatisk laddning.

Spara aldrig e-postadress, lösenord, API-nyckel eller innehållet i
`secrets.yaml` i detta repository.

## 1. Installera Nord Pool

I Home Assistant går du till **Inställningar → Enheter och tjänster → Lägg
till integration** och väljer **Nord Pool**. Välj ditt elområde och SEK.

Kontrollera att du får en sensor för aktuellt pris. Priset ska anges i
SEK/kWh. Nord Pools rena marknadspris saknar moms och övriga påslag.

## 2. Installera Chargeamps

I HACS söker du efter **Chargeamps**, installerar integrationen och startar
om Home Assistant. Lägg därefter till Chargeamps under **Inställningar →
Enheter och tjänster**.

Integrationen kräver dina Charge Amps-uppgifter och en API-nyckel från
Charge Amps support. Dessa uppgifter ska endast anges i Home Assistant.

Kontrollera att Halo-enheten har en switch som kan aktivera och stoppa
laddning. Testa switchen manuellt innan automatiken skapas.

## 3. Skapa huvudbrytaren

Gå till **Inställningar → Enheter och tjänster → Hjälpare → Skapa hjälpare**
och välj **Växla**. Döp den exempelvis till `Automatisk elbilsladdning`.

När hjälparen är avstängd gör blueprinten inga ändringar. Den stoppar inte
automatiskt en redan pågående laddning när du stänger av hjälparen.

## 4. Kopiera blueprinten

Kopiera filen
`blueprints/automation/halo_nordpool_price_control.yaml` till samma sökväg
under Home Assistants konfigurationsmapp.

Gå sedan till **Inställningar → Automationer och scener → Blueprints**,
ladda om blueprints och skapa en automation från blueprinten.

Välj:

- Nord Pools sensor för aktuellt pris.
- Halos laddningsswitch.
- Huvudbrytaren du skapade.
- En startgräns och en något högre stoppgräns.

Exempel: start vid 0,75 SEK/kWh och stopp vid 0,90 SEK/kWh. Skillnaden
förhindrar att laddningen slår av och på vid små prisvariationer.

## Säker provkörning

1. Börja med bilen urkopplad och verifiera hur switchen reagerar.
2. Sätt tillfälligt gränser som gör att start respektive stopp utlöses.
3. Kontrollera automationsspåren i Home Assistant.
4. Återställ önskade gränser innan bilen ansluts.
5. Kontrollera alltid laddboxens och elanläggningens strömgränser separat.

## Begränsning i första versionen

Denna version reagerar på det aktuella priset. Den väljer ännu inte ett
optimalt antal framtida 15-minutersperioder eller tar hänsyn till en
avresetid. Det läggs till efter att de faktiska Halo- och Nord Pool-entiteterna
har verifierats i din Home Assistant-installation.
