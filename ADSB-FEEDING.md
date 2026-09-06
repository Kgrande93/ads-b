# Mate ADS-B-antennen til adsb.lol, adsb.fi og airplanes.live

## Hvorfor

Flyskjermen skal snart få en "lås fly"-modus der man søker opp et
flightnummer og følger det spesifikke flyet - også når det er utenfor
antennens egen rekkevidde. Det krever et globalt datakilde, og valget
falt på **adsb.lol** (samme åpne, gratis, nøkkelfrie community-nettverk
som `readsb`/`tar1090` - verktøyene den lokale antennen allerede
bruker - kommer fra).

adsb.lol sier selv: *"In the future, you will require an API key which
you can obtain by feeding adsb.lol."* Siden vi uansett skal bruke
APIet deres til søkefunksjonen, er det naturlig å mate data tilbake -
det er slik disse nettverkene holdes oppe, og du får trolig bedre/
prioritert tilgang som takk. adsb.fi og airplanes.live er to
søsterprosjekter i samme miljø; å mate alle tre koster ekstra 2 minutter
og null i drift, så denne guiden dekker alle tre.

**Viktig**: dette gjelder antennen/mottakeren (readsb-oppsettet), IKKE
selve Flyskjermen-firmwaret i dette repoet. Ingen kodeendring her.

## Forutsetning

Du har allerede en fungerende ADS-B-mottaker som kjører `readsb`,
`dump1090-fa` eller tilsvarende (det er dette som mater
`aircraft.json`-endepunktet Flyskjermen bruker som "lokal antenne"-
datakilde). Alle tre skriptene under legger seg oppå denne installasjonen
uten å forstyrre det som allerede kjører.

## Kjent fallgruve: `netstat` mangler på Debian 13

Installasjons-/oppdateringsskriptene til alle tre nettverkene (og
`install-or-update-interface.sh` som adsb.lol bruker for å oppdatere
feed-oppsettet) bruker internt `netstat` for å sjekke lyttende porter.
`net-tools`-pakken (som gir `netstat`) er ikke installert som standard på
Debian 13 - samme type manglende avhengighet som `curl` var for
tar1090-installasjonen (se hovedoppskriften i README.md).

Unngå feilen ved å installere pakken *før* du kjører feed-skriptene:

```bash
sudo apt install net-tools -y
```

> ⚠️ Hvis du allerede har kjørt et feed-/oppdateringsskript og fikk en
> `netstat: command not found`-feil underveis: installer `net-tools` som
> over og kjør skriptet på nytt (`sudo bash /usr/local/share/adsblol/git/install-or-update-interface.sh`
> for adsb.lol). Feilen stopper vanligvis ikke selve feedingen - sjekk med
> `sudo systemctl status adsblol-feed adsblol-mlat` om tjenestene likevel
> kjører før du antar noe er ødelagt.

## Løst: `re-api.adsb.lol` gir 403 selv med riktig IP ("no healthy mlat")

Hvis `re-api.adsb.lol` gir `403 Forbidden`/`Access denied` selv når du
har verifisert at forespørselen kommer fra samme offentlige IP som
feederen (bruk `curl -v` direkte fra VM-en, ikke bare nettleseren, for å
utelukke NAT/VLAN/IPv6-forskjeller mellom klient og feeder), er den
vanligste årsaken **at MLAT-tilkoblingen ikke regnes som "healthy" enda**
- ikke et IP-problem.

Sjekk feed-statusen din på https://adsb.lol (samme IP-baserte
gjenkjenning som re-api bruker) og se på `mlat`-objektet i JSON-svaret:

```json
"mlat": [{
  "peer_count": 0,
  "outlier_percent": 0.0,
  "bad_sync_timeout": 0
}]
```

`peer_count: 0` betyr at MLAT-klienten ikke har synkronisert med andre
nærliggende feedere enda - multilaterasjon krever at minst to andre
stasjoner ser samme fly samtidig for å beregne posisjon via
tidsforskjeller. Uten peers regnes MLAT-en som "ikke healthy", og
`re-api` avviser deg selv om beast-feeden (port 30004) fungerer helt fint.

Dette er **normal oppførsel rett etter en fersk installasjon eller
restart**, ikke en feil. Sammenlign med `journalctl -u adsblol-mlat`:
`Results: X positions/minute` bygger seg typisk opp fra 0 til et stabilt
tall over 15-90 minutter etter oppstart, etter hvert som synk med
nærliggende stasjoner etableres. La tjenesten kjøre uforstyrret (ikke
restart den for å "fikse" det) og sjekk `peer_count`/
`positions_per_second` periodisk til den er over 0.

## 1. adsb.lol (prioritert - dette er kilden vi bruker for flysøket)

```bash
curl -L -o /tmp/lol-feed.sh https://adsb.lol/feed.sh
sudo bash /tmp/lol-feed.sh
```

Skriptet spør om koordinater og høyde over havet for antennen (bruk f.eks.
https://www.freemaptools.com/elevation-finder.htm hvis du ikke har
høyden fra før).

**Verifiser**: gå til https://adsb.lol og sjekk at stasjonen din vises
som aktivt matende.

**Krav den API-nøkkelen**: gå til https://my.adsb.lol og "claim" stasjonen
din (knyttes til UUID-en feed-skriptet genererte, ligger i
`/usr/local/share/adsblol/adsblol-uuid`) - det er dette som gir tilgang
til API-nøkkel og feeder-only-funksjoner (re-api, MLAT-kart, osv.) når/hvis
adsb.lol krever nøkkel for søke-endepunktet vi bruker.

> ⚠️ **Kjent, ufarlig oppstartsfeil i MLAT-loggen**: rett etter
> installasjon/omstart kan `journalctl -u adsblol-mlat` vise
> `Beast-format results connection with 127.0.0.1:31421: [Errno 111]
> Connection refused`. Dette er samme type race condition som mellom
> readsb og fr24feed etter reboot (se README.md): mlat-klienten prøver å
> koble seg til `adsblol-feed`-tjenestens lokale resultatport (31421) før
> den tjenesten har rukket å åpne porten. Den kobler seg selv til innen
> 15-30 sekunder (`connection established`), og MLAT begynner å levere
> posisjoner normalt (`Results: X positions/minute` øker gradvis over de
> første 10-15 minuttene). Ingen handling nødvendig - sjekk bare at
> `Receiver: connected` og økende `positions/minute` dukker opp i
> `journalctl -u adsblol-mlat -f` noen minutter etter oppstart.

## 2. adsb.fi

```bash
curl -L -o /tmp/fi-feed.sh https://adsb.fi/feed.sh
sudo bash /tmp/fi-feed.sh
```

**Verifiser**: gå til https://adsb.fi fra samme nettverk som mottakeren -
nederst til venstre skal det stå "You are feeding data".

## 3. airplanes.live

```bash
curl -L -o /tmp/al-feed.sh https://raw.githubusercontent.com/airplanes-live/feed/main/install.sh
sudo bash /tmp/al-feed.sh
```

**Verifiser**: gå til https://airplanes.live/myfeed fra samme nettverk
som mottakeren (siden viser status basert på IP-adressen til
nettleseren din, ikke pålogging/konto).

## Alternativ: Docker/Ultrafeeder

Hvis mottakeren allerede kjører i Docker (eller du foretrekker det), kan
ett og samme `ultrafeeder`-image (sdr-enthusiasts/docker-adsb-ultrafeeder)
mate alle tre samtidig via miljøvariabelen `ULTRAFEEDER_CONFIG`:

```
adsb,in.adsb.lol,30004,beast_reduce_plus_out;
adsb,feed.adsb.fi,30004,beast_reduce_plus_out;
adsb,feed.airplanes.live,30004,beast_reduce_plus_out;
mlat,in.adsb.lol,31090;
mlat,feed.adsb.fi,31090;
mlat,feed.airplanes.live,31090;
```

(semikolon påkrevd mellom hver linje, bortsett fra den siste). Se
full oppskrift: https://github.com/sdr-enthusiasts/docker-adsb-ultrafeeder

## Oppsummert

| Nettverk | Skript | Status-side |
|---|---|---|
| adsb.lol | `curl -L -o /tmp/lol-feed.sh https://adsb.lol/feed.sh && sudo bash /tmp/lol-feed.sh` | adsb.lol |
| adsb.fi | `curl -L -o /tmp/fi-feed.sh https://adsb.fi/feed.sh && sudo bash /tmp/fi-feed.sh` | adsb.fi |
| airplanes.live | `curl -L -o /tmp/al-feed.sh https://raw.githubusercontent.com/airplanes-live/feed/main/install.sh && sudo bash /tmp/al-feed.sh` | airplanes.live/myfeed |

## Verifisert

Innholdet over er sjekket direkte mot offisiell dokumentasjon og
kildekode (ikke bare antatt korrekt):

- **adsb.lol**: `feed.sh`-kommandoen, spørsmål om koordinater/høyde, og
  at API-nøkkel foreløpig *ikke* er påkrevd for å mate ("in the future
  you will require an API key") er bekreftet mot adsb.lol sine egne
  docs. Claim-prosessen via `my.adsb.lol` med stasjonens UUID er
  bekreftet via community-kilder (samme UUID kan gjenbrukes på tvers av
  ADSBExchange/adsb.lol/airplanes.live hvis du allerede mater et av dem).
- **adsb.fi**: `feed.sh`-kommandoen og portene 30004 (ADS-B)/31090
  (MLAT) er bekreftet mot `adsbfi/adsb-fi-scripts` på GitHub.
- **airplanes.live**: installasjonsskriptet er bekreftet mot
  `airplanes-live/feed` på GitHub. **Rettet fra opprinnelig utkast**:
  `myfeed`-siden krever ikke innlogging — den viser status basert på
  IP-adressen til nettleseren, og du må se den fra samme nettverk som
  mottakeren.

## Kilder

- [ADSB.lol - Bare Metal](https://www.adsb.lol/docs/get-started/bare-metal/)
- [ADSB.lol - re-api (feeders only)](https://www.adsb.lol/docs/feeders-only/re-api/)
- [adsbfi/adsb-fi-scripts](https://github.com/adsbfi/adsb-fi-scripts)
- [Getting Started With adsb.fi - ADS-B One Wiki](https://wiki.adsb.one/index.php/Getting_Started_With_adsb.fi)
- [airplanes-live/feed](https://github.com/airplanes-live/feed)
- [How to Feed ADS-B Data to Airplanes.live](https://airplanes.live/how-to-feed/)
- [docker-adsb-ultrafeeder](https://github.com/sdr-enthusiasts/docker-adsb-ultrafeeder)
