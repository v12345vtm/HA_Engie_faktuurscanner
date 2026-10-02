# ENGIE-facturen verwerken in Home Assistant

Deze code leest je ENGIE-facturen (PDF) uit een map op je Home Assistant-server, berekent per maand de energieprijs en de all-in prijs voor elektriciteit en gas, en maakt daar sensoren en een overzichtspagina van. Alles draait op de HA-server zelf, in [pyscript](https://github.com/custom-components/pyscript).

De logica komt uit je twee PyCharm-scripts (`engiefactuurscanner.py` en `energie_prijzen.py`) en is behouden. Het verschil: de zware PDF-verwerking draait nu in een aparte thread (`task.executor`), en het resultaat komt in sensoren, het HA-log en een HTML-pagina terecht.

## Inhoud

1. [Wat het doet](#wat-het-doet)
2. [Waar plaats je wat](#waar-plaats-je-wat)
3. [Installatie](#installatie)
4. [Gebruik](#gebruik)
5. [Sensoren](#sensoren)
6. [De overzichtspagina](#de-overzichtspagina)
7. [Hoe de code werkt](#hoe-de-code-werkt)
8. [Hoe de prijzen berekend worden](#hoe-de-prijzen-berekend-worden)
9. [Instellingen aanpassen](#instellingen-aanpassen)
10. [Logs en status](#logs-en-status)
11. [De JSON-bestanden](#de-json-bestanden)
12. [Problemen oplossen](#problemen-oplossen)
13. [Beperkingen](#beperkingen)

## Wat het doet

```
/media/engie/*.pdf                                   jouw ENGIE-facturen
        |  stap 1: ha_engie_pdf2json      (pdfplumber, in een aparte thread)
        v
/media/engie/extracteddata/energie_historiek.json    ruwe factuurdata, 1 item per PDF
        |  stap 2: ha_engie_nacalculatie  (in een aparte thread)
        v
/media/engie/extracteddata/energie_prijzen.json      prijzen per maand + controle
        |
        |-- kopie --> /config/www/engie/energie_prijzen.json --> ha_engie_resume.html
        |
        |  ha_engie_facturen              (pyscript: sensoren + log)
        v
sensor.incl_engie_*   sensor.excl_engie_*   sensor.engie_facturen_status
```

Eén aanroep van de service `pyscript.verwerk_engie_facturen` laat deze keten één keer van boven naar onder lopen. Al verwerkte PDF's worden overgeslagen, dus een nieuwe run met één extra factuur is snel klaar.

## Waar plaats je wat

```
/config/
├── pyscript/
│   ├── requirements.txt              bestaat al: voeg pdfplumber toe
│   ├── ha_engie_facturen.py          service, sensoren, log, opstart
│   └── modules/
│       ├── ha_engie_pdf2json.py      stap 1: PDF's -> energie_historiek.json
│       └── ha_engie_nacalculatie.py  stap 2: prijsberekening
└── www/
    └── engie/
        ├── ha_engie_resume.html      overzichtspagina
        └── energie_prijzen.json      kopie voor de pagina (wordt automatisch gemaakt)

/media/
└── engie/                            hier zet je de PDF-facturen
    └── extracteddata/                wordt automatisch gemaakt
        ├── energie_historiek.json
        ├── energie_prijzen.json
        └── debug/                    enkel als je met debug: true draait
```

Let op:

- De twee modules moeten in `/config/pyscript/modules/` staan. Pyscript kan alleen uit die map importeren, en `ha_engie_facturen.py` importeert ze.
- De bestandsnamen zijn in kleine letters, want modulenamen zijn hoofdlettergevoelig op Linux.
- Bestaat `/config/pyscript/modules/` of `/config/www/` nog niet, maak ze dan aan.

## Installatie

1. **Requirements.** Open `/config/pyscript/requirements.txt` en voeg `pdfplumber` toe, onder `qrcode`:

   ```
   qrcode
   pdfplumber
   ```

2. **Bestanden kopiëren** naar de plaatsen uit het schema hierboven.
3. **Home Assistant herstarten.** Pyscript installeert `pdfplumber` en laadt de scripts. De eerste keer kan dat even duren.
4. **PDF's plaatsen** in `/media/engie`. Bestaat de map niet, dan maakt de code ze zelf aan en meldt dat in het log.
5. **Service uitvoeren** (zie [Gebruik](#gebruik)). Doe de eerste keer best een run met `debug: true`, dan zie je per PDF welke tekst is uitgelezen.
6. **Logniveau instellen** als je de info-regels in het HA-log wil zien (zie [Logs en status](#logs-en-status)).

Na een wijziging in een van de bestanden herlaadt pyscript automatisch. Door de opstart-trigger draait de verwerking dan ook één keer.

## Gebruik

### De service aanroepen

Via Ontwikkelaarstools → Acties, of in een script of automatisering:

```yaml
action: pyscript.verwerk_engie_facturen
data:
  herstart: false   # true = historiek wissen en alle PDF's opnieuw scannen
  debug: false      # true = uitgelezen tekst per PDF bewaren
```

| Veld | Standaard | Wat het doet |
|---|---|---|
| `herstart` | `false` | Wist `energie_historiek.json` en scant alle PDF's opnieuw. Handig na een aanpassing aan de uitlezing. |
| `debug` | `false` | Bewaart per PDF de uitgelezen tekst als `.txt` in `/media/engie/extracteddata/debug/`. |

Beide velden zijn optioneel. Zonder velden worden alleen nieuwe PDF's gescand.

### Automatisch

- **Bij het opstarten van HA** (en bij het herladen van pyscript) draait de verwerking vanzelf. Dat is nodig omdat sensoren die met `state.set()` worden gemaakt na een herstart niet meer bestaan, tot ze opnieuw gezet worden.
- **Dagelijks nieuwe PDF's oppikken** kan met een automatisering:

  ```yaml
  automation:
    - alias: ENGIE-facturen verwerken
      triggers:
        - trigger: time
          at: "06:15:00"
      actions:
        - action: pyscript.verwerk_engie_facturen
  ```

  Of door onderaan `ha_engie_facturen.py` de decorator van `engie_bij_opstart` te vervangen door `@time_trigger("startup", "cron(15 6 * * *)")`. Die regel staat daar al als commentaar.

### Pagina in een dashboard tonen

Met een webpagina-kaart (iframe):

```yaml
type: iframe
url: /local/engie/ha_engie_resume.html
aspect_ratio: 100%
```

## Sensoren

| Entiteit | Eenheid | Betekenis |
|---|---|---|
| `sensor.incl_engie_elektriciteit_kwh` | EUR/kWh | All-in prijs elektriciteit: afname + netwerk + toeslagen + btw |
| `sensor.incl_engie_gas_kwh` | EUR/kWh | All-in prijs gas per kWh |
| `sensor.incl_engie_gas_m3` | EUR/m³ | All-in prijs gas per m³ |
| `sensor.excl_engie_elektriciteit_kwh` | c€/kWh | Energieprijs elektriciteit: enkel afname, excl. btw, netwerk en toeslagen |
| `sensor.excl_engie_gas_kwh` | c€/kWh | Energieprijs gas, zelfde opbouw |
| `sensor.engie_facturen_status` | tekst | Samenvatting van de laatste verwerking |

De prijssensoren tonen de **meest recente** periode die elektriciteit, respectievelijk gas bevat. Had je laatste factuur enkel gas, dan blijft de elektriciteitssensor op de maand ervoor staan. Het attribuut `periode` zegt altijd welke maand het is.

Attributen van de prijssensoren: `periode` (bv. `09/2026`), `periode_van`, `periode_tot`, `factuurnummer` en `controle_ok` (of de factuur klopte met de gescande totalen).

Attributen van de statussensor: `status` (`ok`, `waarschuwing` of `fout`), `laatste_run`, `pdf_gevonden`, `nieuw_gescand`, `overgeslagen`, `json`, `periodes`, `controle_fouten` en `fouten` (maximaal 5).

Is er voor een soort nog geen gegevens, dan staat de sensor op `unknown`.

Goed om te weten: deze sensoren worden door pyscript in de toestandsmachine gezet en horen niet bij een integratie. Ze hebben geen unieke ID, dus je kan ze niet via de UI hernoemen. Naam en eenheid pas je aan in de lijst `SENSOREN` bovenaan `ha_engie_facturen.py`.

Tip (niet getest): de eenheden `EUR/kWh` en `EUR/m³` zijn gekozen zodat de `incl_`-sensoren in het Energiedashboard als prijs-entiteit te kiezen zouden moeten zijn, als je HA-valuta EUR is.

## De overzichtspagina

`ha_engie_resume.html` toont de historiek in dezelfde opbouw als de console van je PyCharm-script:

- **Recentste prijs**: dezelfde waarden als de sensoren, plus gas per m³.
- **Energieprijs per maand**: enkel afname, in c€/kWh (elektriciteit en gas) en EUR/m³ (gas).
- **All-in prijs per maand**: met eindsom van de factuur en een controle-kolom (`OK` of `Controleren`).
- **Meldingen**: de opmerkingen uit de controle, enkel zichtbaar als er zijn.

De nieuwste maand staat bovenaan en wordt gemarkeerd. De pagina volgt het licht/donker-thema van je toestel.

Open ze op `http://<ip-van-ha>:8123/local/engie/ha_engie_resume.html`.

De pagina leest `energie_prijzen.json` uit dezelfde map (`/config/www/engie/`) en haalt die bij elk bezoek vers op, zodat je nooit oude gegevens uit de browsercache ziet. De JSON in `/media/engie/extracteddata/` kan een statische pagina niet lezen, want `/media` vraagt een login. Daarom schrijft stap 2 een kopie naar `www`.

Alles in `/config/www` is via `/local` zonder login bereikbaar, ook voor andere toestellen op je netwerk. De kopie bevat factuurnummers, contractnaam en verbruik. Stel je HA bloot aan het internet, bedenk dan of je dat wil.

## Hoe de code werkt

### Drie bestanden

| Bestand | Rol |
|---|---|
| `ha_engie_facturen.py` | Het script dat pyscript laadt. Bevat de service `verwerk_engie_facturen`, de opstart-trigger, het zetten van de sensoren, het wegschrijven naar het HA-log en de statussensor. |
| `modules/ha_engie_pdf2json.py` | Stap 1. Je scanner uit `engiefactuurscanner.py`. |
| `modules/ha_engie_nacalculatie.py` | Stap 2. Je berekening uit `energie_prijzen.py`. |

De twee stappen staan in `modules/` zodat het hoofdscript ze kan importeren. Alle instellingen van de berekening staan bovenaan `ha_engie_nacalculatie.py`, de paden bovenaan `ha_engie_facturen.py`.

### Native functies en `task.executor`

Pyscript draait in de event loop van Home Assistant. Een PDF uitlezen met `pdfplumber` blokkeert even, en dat hoort daar niet. Daarom staat het eigenlijke werk in functies met de decorator `@pyscript_compile`: dat zijn gewone (native) Python-functies. Pyscript start ze met `task.executor(...)` in een aparte thread, zodat HA gewoon doorloopt.

```python
@pyscript_compile
def _scan_native(...):          # gewone Python, draait in een thread
    ...

def engie_pdf2json(...):        # pyscript-functie, draait in de event loop
    return task.executor(_scan_native, ...)
```

Twee ontwerpkeuzes:

- **De native functies zijn zelfstandig.** De imports (`json`, `re`, `pathlib`, `pdfplumber`, ...) en alle hulpfuncties (`parse_factuur_tekst`, `verwerk_factuur`, ...) staan binnenin de functie. Ze zijn daardoor onafhankelijk van pyscript-instellingen. Krijg je toch een fout als `import of ... not allowed`, zie [Problemen oplossen](#problemen-oplossen).
- **Native functies loggen en zetten geen sensoren zelf.** `log` en `state` bestaan alleen in pyscript-code. De native functies geven daarom een dict terug met de resultaten en een lijst `meldingen` van `(niveau, tekst)`. Het hoofdscript zet die in het HA-log en gebruikt de resultaten voor de sensoren.

### Een run stap voor stap

Wat `_voer_uit()` in `ha_engie_facturen.py` doet:

1. **Controle of er al een run bezig is.** Zo ja, dan wordt de nieuwe aanvraag overgeslagen en gelogd. Zo kunnen de opstart-run en een handmatige aanroep elkaars JSON niet overschrijven.
2. **Stap 1, PDF's naar JSON.**
   - Maakt `/media/engie` en `extracteddata/` aan als ze ontbreken.
   - Wist bij `herstart: true` eerst `energie_historiek.json`.
   - Leest de bestaande JSON en onthoudt de `bronbestand`-namen die al verwerkt zijn.
   - Overloopt alle `*.pdf` (hoofdletterongevoelig, op naam gesorteerd). Nieuwe PDF's gaan door `pdfplumber` en `parse_factuur_tekst`, al verwerkte worden overgeslagen.
   - Schrijft de JSON alleen weg als er nieuwe facturen zijn.
3. **Stap 2, nacalculatie.** Leest `energie_historiek.json`, berekent de prijzen per factuur, schrijft `energie_prijzen.json` en de kopie voor de pagina. Is de historiek-JSON er niet, dan is er niets te berekenen en volgt een waarschuwing.
4. **Sensoren zetten** uit het blok `meest_recente`. Alleen als er prijzen zijn: bij een mislukte run behouden de sensoren hun vorige waarde.
5. **Status bepalen en loggen.** `fout` als er fouten waren (PDF onleesbaar, `pdfplumber` ontbreekt), `waarschuwing` als een periode de controle niet haalt of er niets te verwerken is, anders `ok`.

### Incrementeel en veilig

- **Incrementeel.** Een PDF telt als verwerkt zodra zijn bestandsnaam als `bronbestand` in de JSON staat. Vervang je een PDF door een gecorrigeerde met dezelfde naam, gebruik dan `herstart: true`.
- **Een mislukte PDF blokkeert niets.** Een fout bij één bestand wordt gelogd, de rest gaat door. Het bestand staat niet in de JSON en wordt de volgende run opnieuw geprobeerd.
- **Atomisch schrijven.** JSON gaat eerst naar een `.tmp`-bestand en wordt dan vervangen. Stopt HA midden in een run, dan blijft er geen half bestand achter.
- **Dubbele periodes.** Staan twee facturen op dezelfde maand, dan wint de factuur die het laatst in `energie_historiek.json` staat. Dat komt als opmerking in de controle.
- **Enkel-gasfacturen.** De scanner kent de eerste netwerkkost en toeslag altijd aan elektriciteit toe. Bij een factuur zonder elektriciteit verplaatst de nacalculatie die bedragen naar gas, met een opmerking. Dat is je eigen opvangregel uit PyCharm.

## Hoe de prijzen berekend worden

Per factuur (= 1 maand), apart voor elektriciteit en gas:

| Grootheid | Formule |
|---|---|
| Subtotaal excl. btw | afname + netwerk + toeslagen |
| Btw | subtotaal x `BTW_PCT` / 100 |
| Totaal incl. btw | subtotaal + btw |
| **Energieprijs** (excl.) | afname / kWh, op de sensor vermenigvuldigd met 100 voor c€/kWh |
| **All-in prijs** (incl.) | totaal incl. btw / kWh |
| Gas per m³ | prijs per kWh x `KWH_PER_M3` |

De kWh voor elektriciteit komt uit `totaal_afname_kwh`, of anders uit piek + dal. Voor gas uit `verbruik_kwh`. De afname is `afname_engie` (elektriciteit) of `verbruik_engie` (gas), de bedragen van "dit betaal je aan ENGIE".

De all-in prijs is de totale kost gedeeld door het verbruik. Zitten er vaste bedragen in de rubrieken, dan schommelt hij dus mee met je verbruik. De energieprijs (enkel afname) is daar het zuiverste cijfer om maanden mee te vergelijken.

**Controle tegen de factuur.** De som van de rubrieken moet overeenkomen met `totaal_excl_btw`, en som x (1 + btw) met `totaal_incl_btw`, binnen `TOLERANTIE` (0,02 EUR). Anders krijgt de periode `controle.ok = false`, en toont de pagina "Controleren". Typische oorzaken: een veld dat de scanner niet vond, of een `BTW_PCT` die niet klopt voor die periode.

## Instellingen aanpassen

| Wat | Waar | Standaard |
|---|---|---|
| Map met PDF's | `PDF_MAP` in `ha_engie_facturen.py` | `/media/engie` |
| Map voor de JSON's | `DATA_MAP` in `ha_engie_facturen.py` | `/media/engie/extracteddata` |
| Kopie voor de pagina | `WEB_JSON` in `ha_engie_facturen.py` | `/config/www/engie/energie_prijzen.json` |
| Sensoren (naam, eenheid, icoon) | `SENSOREN` in `ha_engie_facturen.py` | 5 sensoren |
| Btw-percentage | `BTW_PCT` in `ha_engie_nacalculatie.py` | `6.0` |
| kWh per m³ | `KWH_PER_M3` in `ha_engie_nacalculatie.py` | `11.4` |
| Toegelaten verschil bij de controle | `TOLERANTIE` in `ha_engie_nacalculatie.py` | `0.02` |
| Decimalen | `DECIMALEN_BEDRAG` en `DECIMALEN_PRIJS` | `2` en `4` |

Verplaats je `WEB_JSON`, zet dan `ha_engie_resume.html` in dezelfde map, want de pagina zoekt `energie_prijzen.json` naast zichzelf.

**De uitlezing aanpassen** (bv. als ENGIE de factuur anders opmaakt): de reguliere expressies staan in `parse_factuur_tekst`, binnen `_scan_native` in `ha_engie_pdf2json.py`. Dat is dezelfde functie als in je PyCharm-script. Werkwijze: draai met `debug: true`, open de `.txt` in `extracteddata/debug/`, pas de patronen aan, bewaar het bestand (pyscript herlaadt) en draai met `herstart: true`.

## Logs en status

Voorbeelden van regels die je in het log ziet:

```
Engie pdf2json: 14 pdf gevonden in /media/engie: 3 gescand, 11 overgeslagen (reeds verwerkt), 0 fout(en)
Engie pdf2json: energie_historiek.json bijgewerkt: 3 nieuwe factuur/facturen, 14 in totaal
Engie nacalculatie: energie_prijzen.json bijgewerkt: 14 periodes
Engie nacalculatie: meest recente prijs elektriciteit (09/2026): 8.80 c/kWh excl., 0.1781 EUR/kWh all-in
Engie: 5 prijssensoren bijgewerkt
Engie: 14 pdf gevonden, 3 nieuw gescand, json bijgewerkt, 14 periodes berekend
```

Zonder PDF's: `Engie pdf2json: 0 pdf gevonden in map /media/engie` (waarschuwing). Een nieuw aangemaakte JSON meldt `aangemaakt`, een gewijzigde `bijgewerkt`, een ongewijzigde `geen nieuwe facturen om toe te voegen`.

| Niveau | Gebruikt voor |
|---|---|
| `debug` | per PDF "verwerken" of "overslaan" |
| `info` | samenvattingen, JSON aangemaakt of bijgewerkt, sensoren bijgewerkt |
| `warning` | 0 pdf gevonden, map aangemaakt, periode die de controle niet haalt |
| `error` | PDF kon niet verwerkt worden, `pdfplumber` ontbreekt, onverwachte fout |

Standaard toont HA geen `info`-regels van custom components. Wil je ze zien, zet dan in `configuration.yaml`:

```yaml
logger:
  logs:
    custom_components.pyscript: info
```

Dit geldt voor alle pyscript-scripts samen. Je leest ze in Instellingen → Systeem → Logboeken, met de optie voor ruwe logs. Zonder die instelling zie je het resultaat altijd nog in `sensor.engie_facturen_status`.

## De JSON-bestanden

Voorbeelden met verzonnen cijfers.

**`energie_historiek.json`**: ruwe uitlezing, 1 item per PDF.

```json
[
    {
        "bronbestand": "factuur_2026-08.pdf",
        "contract": "Engie Basic - vaste prijs - 1 jaar",
        "factuurnummer": "123456789012",
        "factuurdatum": "05.09.2026",
        "facturatieperiode": "01.08.2026 - 31.08.2026",
        "elektriciteit": {
            "ean": "541234567890123456",
            "afname_engie": 20.0,
            "netwerkkosten": 15.0,
            "toeslagen_en_heffingen": 5.0,
            "piek_kwh": 100,
            "dal_kwh": 150,
            "totaal_afname_kwh": 250
        },
        "aardgas": {
            "ean": "541234567890654321",
            "verbruik_engie": 30.0,
            "netwerkkosten": 10.0,
            "toeslagen_en_heffingen": 2.0,
            "verbruik_kwh": 500,
            "verbruik_m3": 44
        },
        "totaalbedragen": {
            "totaal_excl_btw": 82.0,
            "btw": 4.92,
            "totaal_incl_btw": 86.92,
            "totaal_te_betalen": 86.92
        }
    }
]
```

**`energie_prijzen.json`**: berekende prijzen. Dit is ook wat de pagina leest.

```json
{
    "gegenereerd": "2026-10-02T12:00:00+02:00",
    "instellingen": { "btw_pct": 6.0, "kwh_per_m3": 11.4, "decimalen_bedrag": 2, "decimalen_prijs": 4 },
    "meest_recente": {
        "elektriciteit": {
            "periode": "09/2026",
            "periode_van": "2026-09-01",
            "periode_tot": "2026-09-30",
            "factuurnummer": "123456789013",
            "controle_ok": true,
            "energie_eur_per_kwh_excl_btw": 0.088,
            "all_in_eur_per_kwh_incl_btw": 0.1781
        },
        "gas": {
            "periode": "09/2026",
            "...": "zelfde velden als hierboven, plus:",
            "energie_eur_per_m3_excl_btw": 0.6464,
            "all_in_eur_per_m3_incl_btw": 1.0825
        }
    },
    "periodes": [
        {
            "periode": "08/2026",
            "periode_van": "2026-08-01",
            "periode_tot": "2026-08-31",
            "factuurnummer": "123456789012",
            "contract": "Engie Basic - vaste prijs - 1 jaar",
            "btw_pct": 6.0,
            "elektriciteit": { "kwh": 250, "afname": 20.0, "netwerk": 15.0, "toeslag": 5.0, "...": "..." },
            "gas": { "kwh": 500, "m3_factuur": 44, "kwh_per_m3_factuur": 11.36, "...": "..." },
            "totalen_factuur": { "subtotaal_excl_btw": 82.0, "btw": 4.92, "eindsom_incl_btw": 86.92 },
            "controle": { "ok": true, "verschil_excl_btw": 0.0, "verschil_incl_btw": 0.0, "opmerkingen": [] }
        }
    ]
}
```

## Problemen oplossen

| Wat je ziet | Oorzaak en oplossing |
|---|---|
| Log: `pdfplumber is niet geinstalleerd` | `pdfplumber` staat niet in `requirements.txt`, of de installatie is mislukt. Voeg het toe, herstart HA en zoek in het log naar fouten bij het installeren. |
| Fout `import of ... not allowed` | Pyscript blokkeert imports die niet op zijn lijst staan. Zet "Allow All Imports" aan in de opties van de pyscript-integratie (Instellingen → Apparaten en diensten → Pyscript → Configureren). |
| De service `pyscript.verwerk_engie_facturen` bestaat niet | Het script is niet geladen. Zoek in het log naar fouten van `custom_components.pyscript`, controleer dat de twee modules echt in `/config/pyscript/modules/` staan en herstart HA of voer `pyscript.reload` uit. |
| Status zegt `0 pdf gevonden` | De PDF's staan niet in `PDF_MAP`, of hun extensie is geen `.pdf`. |
| Veel lege velden, of "Controleren" in de pagina | De factuur is anders opgemaakt dan de patronen verwachten, of `BTW_PCT` klopt niet voor die periode. Draai met `debug: true`, vergelijk de `.txt` met de patronen in `parse_factuur_tekst`, en scan opnieuw met `herstart: true`. |
| Een gecorrigeerde PDF verschijnt niet in de historiek | Een PDF met een bestandsnaam die al verwerkt is, wordt overgeslagen. Geef hem een andere naam, of draai met `herstart: true`. |
| Sensoren staan op `unknown` na een herstart | De opstart-run vond geen gegevens. Controleer of `/media/engie/extracteddata/energie_historiek.json` nog bestaat en of `/media` al beschikbaar was. Een handmatige run herstelt dit. |
| Pagina zegt "Nog geen gegevens gevonden" | De service is nog niet gedraaid, of `/config/www/engie/energie_prijzen.json` ontbreekt. |
| 404 op `/local/engie/ha_engie_resume.html` | Werd `/config/www` pas aangemaakt nadat HA gestart was, dan serveert HA die map pas na een herstart. |
| Log: `er loopt al een verwerking` | Er is al een run bezig, bv. de opstart-run. Wacht tot die klaar is. |

## Beperkingen

- **Niet getest op echte facturen.** De uitleeslogica is 1-op-1 je PyCharm-code. De rest is getest met nagemaakte factuurtekst en nagebootste pyscript-functies, niet met echte PDF's of op een draaiende Home Assistant.
- **Eén btw-percentage** voor elektriciteit en gas samen (`BTW_PCT`). Gelden er voor jouw periode verschillende tarieven, dan klopt de controle niet.
- **Sensoren zijn "virtueel"** (geen unieke ID, niet te beheren via de UI) en verdwijnen na een herstart tot de opstart-run ze terugzet.
- **De uitlezing hangt af van de factuuropmaak.** Verandert ENGIE de lay-out, dan moeten de patronen in `parse_factuur_tekst` mee.
