# Hva `install.sh` gjør

`install.sh` er en automatisert versjon av installasjonsstegene i
`README.md`. Den setter opp en komplett ADS-B-mottakerstasjon
(readsb + tar1090 + fr24feed) på en frisk Debian 13-VM, uten at du må
kjøre kommandoene manuelt én etter én.

## Forutsetninger

- Må kjøres som root (`sudo bash install.sh`) — scriptet avbryter selv om
  ikke.
- Du må ha en FR24 delingsnøkkel klar (fra
  flightradar24.com/share-your-data) — scriptet spør om den først og
  avbryter hvis feltet er tomt.

## Steg for steg

**1/6 — Installerer avhengigheter**
Kjører `apt update && apt upgrade`, installerer
`qemu-guest-agent rtl-sdr git curl`, starter guest-agenten og laster inn
udev-reglene på nytt (for at RTL2832U-dongelen skal fungere som en
ikke-root-enhet).

**2/6 — Installerer readsb + tar1090**
Kloner `wiedehopf/adsb-scripts` til `/root/adsb-scripts` (hvis den ikke
allerede finnes) og kjører `readsb-install.sh`. Dette installerer selve
ADS-B-dekoderen (readsb) og webgrensesnittet (tar1090) i samme steg.

**3/6 — Fjerner motstridende dekoder-repos**
Sletter eventuelle gamle PiAware/FlightAware apt-repo-filer
(`piaware.list` og `flightaware.gpg`) som kan kollidere med readsb, og
oppdaterer apt-cachen.

**4/6 — Installerer fr24feed**
Kjører FR24s offisielle installer (`fr24.com/install.sh`), men sender
svarene på installerens interaktive spørsmål automatisk via `printf`
pipet inn i den (`1`, tom, `no`, `no`, `yes` — mottakertype 1, tomme
dekoderargumenter, RAW nei, Basestation nei, MLAT ja). Disse svarene
spiller egentlig ingen rolle i det lange løp, fordi neste steg overskriver
konfigurasjonsfilen uansett.

**5/6 — Skriver riktig `fr24feed.ini`**
FR24-installeren setter opp sin egen (ikke-fungerende) dump1090-dekoder på
Debian 13. Dette steget overskriver `/etc/fr24feed.ini` med en versjon som
i stedet peker på readsb sin Beast-strøm på `127.0.0.1:30005`, med nøkkelen
du oppga i starten satt inn. Deretter aktiveres og restartes
`fr24feed`-tjenesten.

**6/6 — Verifiserer**
Venter 5 sekunder, kjører `fr24feed-status` for å vise feed-status, og
sjekker at tar1090 svarer på `aircraft.json`. Hvis tar1090 ikke svarer,
kjøres tar1090-installasjonssteget på nytt automatisk som et forsøk på
selvreparasjon. Til slutt skrives VM-ens IP-adresse ut sammen med lenken
til webgrensesnittet (`http://<ip>/tar1090`).

## Hva scriptet ikke gjør

- Setter ikke opp OpenSky-feederen (det er fortsatt et manuelt steg i
  README.md, siden det innebærer interaktive spørsmål om branch/port/
  brukernavn).
- Installerer ikke graphs1090 (signalgrafer) eller flydatabasen
  (`aircraft.csv.gz`) — begge er valgfrie tilleggssteg i README.md.
- Setter ikke posisjon (lat/lon/høyde) for readsb — det må gjøres separat
  med `readsb-set-location` før OpenSky-feederen tas i bruk.
