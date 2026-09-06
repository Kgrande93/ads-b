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
