# Vremena vožnje između stanica javnog prevoza u Kantonu Sarajevo

Četiri tabele sa ukupno **2.810.749 zabilježenih vožnji između dvije uzastopne stanice**, izvedene iz sirovih GPS zapisa autobusa, tramvaja i trolejbusa u periodu od 16. novembra 2025. do 12. aprila 2026. godine. Jedan red opisuje jednu takvu vožnju, sa poznatim trajanjem, udaljenošću, vremenom polaska i dolaska, stanicama, vozilom i linijom.

Kanton Sarajevo nema javno dostupan red vožnje u GTFS formatu, pa ovakav skup podataka do sada nije postojao. Podaci su nastali kao dio završnog rada prvog ciklusa na Elektrotehničkom fakultetu Univerziteta u Sarajevu, ali su upotrebljivi i izvan predikcije vremena dolaska, za analizu brzina, pouzdanosti, zagušenja po dijelovima grada ili poređenje načina prevoza.

**Kod kojim su podaci nastali:** [eta-public-transport-sarajevo](https://github.com/akanlic2/eta-public-transport-sarajevo)

## Sadržaj

| Folder | Tabela | Redova | Kolona | Polazaka | Stanica | Vozila | Ruta | Period | Datoteka |
|---|---|---:|---:|---:|---:|---:|---:|---|---:|
| `gras-tramvaji/` | `gras_tramvaji_segmenti` | 1.054.267 | 27 | 68.929 | 50 | 45 | 21 | 16.11.2025 – 12.04.2026 | 4 |
| `gras-trolejbusi/` | `gras_trolejbusi_segmenti` | 962.145 | 25 | 61.618 | 108 | 33 | 29 | 16.11.2025 – 12.04.2026 | 4 |
| `gras-busevi/` | `gras_autobusi_segmenti` | 135.822 | 24 | 13.817 | 179 | 21 | 31 | 10.12.2025 – 12.04.2026 | 1 |
| `centrotrans-busevi/` | `centrotrans_autobusi_segmenti` | 658.515 | 23 | 61.609 | 605 | 88 | — | 10.12.2025 – 12.04.2026 | 2 |

Podskupovi su odvojeni jer se razlikuju po operateru, tipu vozila i raspoloživim kolonama. GRAS je gradsko preduzeće koje vozi tramvaje, trolejbuse i autobuse, a Centrotrans je privatni prevoznik sa prigradskim i gradskim autobuskim linijama. Oznake stanica i polazaka nezavisne su po tabeli, pa `stop_id` 45 kod tramvaja i kod Centrotransa nisu ista stanica.

Tabele su u **CSV** formatu, sa zaglavljem u prvom redu i u UTF-8 kodiranju. Pošto GitHub ne prima datoteku veću od 100 MB, tri veće tabele podijeljene su hronološki na dijelove, od `gras_tramvaji_segmenti_dio1.csv` do `_dio4.csv` i tako redom. Rez uvijek pada između dva polaska, pa je svaka vožnja u cijelosti u jednoj datoteci i nijedan red se ne ponavlja. Dijelovi se spajaju u jednu tabelu jednom naredbom, kako pokazuje sekcija Učitavanje.

**Nijedna tabela nema nedostajućih vrijednosti.**

## Odakle podaci dolaze

Sirovi zapisi preuzeti su sa platforme [NAVSA](https://navsa.spatias.com/), koja objedinjuje podatke o kretanju vozila javnog prevoza u Kantonu Sarajevo. Polazište je 46.861.059 GPS pozicija bez dokumentacije o strukturi, bez oznake polaska i bez liste stanica.

Do ovih tabela se od njih dolazi u nekoliko koraka. Neprekidni trag svakog vozila razlomljen je na pojedinačne polaske, aktivne stanice detektovane su iz samih pozicija uz stajališta iz OpenStreetMap-a kao referentne tačke, GPS tačke su dodijeljene stanicama, a zatim je za svaki par uzastopnih stanica izračunato vrijeme vožnje. Postupak je u cijelosti opisan u završnom radu i izveden u sveskama u repozitoriju sa kodom.

## Struktura tabele

Jedan red je jedan segment, dakle vožnja od prethodne do trenutne stanice unutar jednog polaska. Polazak se rekonstruiše grupisanjem po `trip_id` i sortiranjem po `stop_sequence`.

Ciljna varijabla `travel_time_from_prev_sec` je razlika između `arrival_time` i `prev_departure_time`, dakle **čisto vozno vrijeme koje ne uključuje zadržavanje na stanici**. Zadržavanje je zasebna kolona `prev_dwell_time_sec`. Ko želi vrijeme od dolaska do dolaska, kakvo putnik zapravo osjeti, sabere te dvije kolone.

### Zajedničke kolone

| Kolona | Tip | Opis |
|---|---|---|
| `trip_id` | string | Oznaka polaska. Sufiks poput `_a_ca` nastaje pri razdvajanju pogrešno spojenih polazaka |
| `orig_trip_id` | int | Oznaka polaska prije razdvajanja. Kod Centrotransa se kolona zove `old_trip_id` |
| `stop_sequence` | int | Redni broj segmenta unutar polaska, počinje od 1 |
| `prev_stop_id` | int | Oznaka polazne stanice segmenta |
| `prev_stop_name` | string | Naziv polazne stanice |
| `stop_id` | int | Oznaka odredišne stanice segmenta |
| `stop_name` | string | Naziv odredišne stanice |
| `prev_departure_time` | datetime | Vrijeme polaska sa prethodne stanice |
| `arrival_time` | datetime | Vrijeme dolaska na odredišnu stanicu |
| `travel_time_from_prev_sec` | float | **Ciljna varijabla.** Trajanje vožnje u sekundama, bez zadržavanja |
| `distance_from_prev_m` | float | Haversine udaljenost između dvije stanice, u metrima |
| `prev_stop_lat`, `prev_stop_lon` | float | Koordinate polazne stanice |
| `stop_lat`, `stop_lon` | float | Koordinate odredišne stanice |
| `route_progress` | float | Položaj segmenta u polasku, od 0 do 1 |
| `hour` | int | Sat dolaska, 0 do 23 |
| `day_of_week` | int | Dan u sedmici, 0 je ponedjeljak a 6 nedjelja |
| `month` | int | Mjesec, 1 do 12 |
| `prev_dwell_time_sec` | float | Zadržavanje vozila na polaznoj stanici segmenta, u sekundama |
| `vehicle_id` | string | Garažni broj vozila |

### Kolone o liniji

| Kolona | Tabele | Opis |
|---|---|---|
| `route` | tramvaji, trolejbusi, GRAS autobusi | Ruta u obliku „terminus → terminus". Kod tramvaja i autobusa izvedena iz oznake polaska, kod trolejbusa iz stvarno posjećenih terminusa |
| `direction_route` | trolejbusi | Ruta prema oznaci polaska, zadržana radi poređenja. Poklapa se sa `route` u 85,4 % redova, a razlika pokazuje koliko je oznaka nepouzdana |
| `direction_prefix` | Centrotrans | Dispečerska oznaka polaska, 10.381 jedinstvena vrijednost. Sirova oznaka ima oblik `67630-3`, gdje prefiks identifikuje pojedinačni polazak a sufiks od 1 do 7 označava dan u sedmici. Objavljen je samo prefiks, jer sufiks u 99,9 % zapisa ponavlja kolonu `day_of_week`. Oznaka ne govori kojom rutom polazak ide, pa je ostavljena onome ko je želi dalje istražiti |

Centrotrans nema kolonu `route`, jer njegova sirova oznaka smjera nije tekstualni opis relacije nego numerički kod, pa se iz nje ruta nije mogla pouzdano izvesti.

### Indikatori posebnih vožnji

Ove kolone nastale su iz sirove oznake polaska i **ne mogu se rekonstruisati iz ostalih kolona**. Služe za izdvajanje redovnog saobraćaja, jer vožnja u depo ili primopredaja vozila ne slijede istu logiku kao linija u špici.

| Kolona | Tabele | Opis |
|---|---|---|
| `is_nocni` | tramvaji, trolejbusi | Noćni polazak |
| `is_depo` | trolejbusi | Vožnja iz depoa ili u depo |
| `is_vanredni` | tramvaji | Vanredna vožnja |
| `is_primopredaja` | tramvaji | Primopredaja vozila |
| `is_probni` | tramvaji | Probna vožnja |
| `is_vozno_osoblje` | GRAS autobusi | Službena vožnja voznog osoblja |
| `is_suburban` | GRAS autobusi, Centrotrans | Polazak izlazi iz urbanog jezgra |
| `is_mixed_traffic_segment` | tramvaji | Obje stanice segmenta leže u zoni u centru grada gdje tramvaj dijeli put sa automobilima |

## Šta je namjerno izostavljeno

Tabele ne sadrže kolone koje se u jednom redu koda dobiju iz postojećih, jer bi samo povećale fajl. Radi se o ovome:

```python
df["is_weekend"] = df["day_of_week"] >= 5
df["hour_sin"] = np.sin(2 * np.pi * df["hour"] / 24)      # i ostalo ciklično kodiranje
df["terminus_a"] = df["route"].str.split(" → ").str[0]     # i terminus_b
df = df.sort_values(["trip_id", "stop_sequence"])
df["prev_segment_travel_time"] = df.groupby("trip_id")["travel_time_from_prev_sec"].shift(1)
df["prev_segment_distance"] = df.groupby("trip_id")["distance_from_prev_m"].shift(1)
```

Izostavljene su i oznake jutarnje i popodnevne špice. Njihove granice bile su postavljene zasebno za svaki podskup, prema izmjerenom vrhuncu opterećenja, pa ista oznaka nije značila isto u svim tabelama.

## Učitavanje

```python
import glob
import pandas as pd

VREMENSKE = ["prev_departure_time", "arrival_time"]

# podijeljena tabela, dijelovi se učitaju i spoje u jednu
df = pd.concat(
    [pd.read_csv(f, parse_dates=VREMENSKE)
     for f in sorted(glob.glob("gras-tramvaji/gras_tramvaji_segmenti_dio*.csv"))],
    ignore_index=True,
)

# tabela u jednoj datoteci
autobusi = pd.read_csv("gras-busevi/gras_autobusi_segmenti.csv", parse_dates=VREMENSKE)

# prosječna brzina po segmentu, u km/h
df["brzina_kmh"] = df["distance_from_prev_m"] / df["travel_time_from_prev_sec"] * 3.6

# samo redovne dnevne vožnje
redovne = df[~df["is_nocni"] & ~df["is_vanredni"] & ~df["is_primopredaja"] & ~df["is_probni"]]

# tipično trajanje po paru stanica i satu
redovne.groupby(["prev_stop_name", "stop_name", "hour"])["travel_time_from_prev_sec"].median()
```

CSV ne nosi tipove kolona, pa se dvije vremenske kolone čitaju sa `parse_dates`, dok brojeve i logičke vrijednosti pandas prepozna sam. Tipovi ostalih kolona navedeni su u tabelama iznad.

U R-u se datoteka čita sa `read.csv()`, a dijelovi spajaju sa `do.call(rbind, lapply(list.files("gras-tramvaji", full.names = TRUE), read.csv))`.

## Ograničenja

Vremena dolaska i polaska nisu mjerena nego procijenjena iz GPS pozicija, koje stižu u nepravilnim razmacima, pa svaka vrijednost nosi grešku procjene. Stanice su detektovane iz samih podataka, a ne preuzete iz službenog registra, pa se nazivi mogu razlikovati od zvaničnih i moguće je da neka rijetko korištena stanica nedostaje.

Period nije neprekidan. Kod sva tri GRAS podskupa podaci nedostaju od 12. do 21. marta, a Centrotrans se u većem obimu pojavljuje tek od februara, uz dva izolovana dana u decembru. Autobuski podskupovi počinju skoro mjesec dana kasnije od tramvaja i trolejbusa.

Segmenti sa nemogućim vrijednostima, poput impliciranih brzina iznad realnog praga ili trajanja koja odgovaraju stajanju a ne vožnji, uklonjeni su tokom obrade. Skup zato ne opisuje sve što se stvarno dogodilo, nego vožnje za koje je trajanje pouzdano utvrđeno.

Podaci ne sadrže broj putnika, vremenske uslove, ni službeni red vožnje, jer ništa od toga nije bilo dostupno. Rute kod tramvaja i GRAS autobusa izvedene su iz oznake polaska, koja je kod trolejbusa bila nepouzdana, pa je tamo ruta određena iz stvarno posjećenih terminusa.

## Licenca

Podaci su objavljeni pod licencom [Open Database License (ODbL) v1.0](https://opendatacommons.org/licenses/odbl/1-0/). Slobodno ih koristite, mijenjajte i dijelite, uz navođenje izvora i uz uslov da izvedene baze podataka dijelite pod istom licencom.

Položaji stanica izvedeni su uz korištenje stajališta iz OpenStreetMap-a, pa uz svako korištenje ide i napomena **© OpenStreetMap doprinosioci**, čiji podaci su također pod ODbL licencom.

## Citiranje

```
A. Kanlić, "Vremena vožnje između stanica javnog prevoza u Kantonu Sarajevo",
skup podataka, Elektrotehnički fakultet, Univerzitet u Sarajevu, 2026.
```

Rad u kojem su podaci nastali:

```
A. Kanlić, "Predikcija vremena dolaska javnog prevoza na osnovu analize podataka",
završni rad prvog ciklusa, Elektrotehnički fakultet, Univerzitet u Sarajevu,
Sarajevo, 2026.
```

## Kontakt

Za pitanja o podacima, uočene greške ili prijedloge, otvorite issue u ovom repozitoriju.
