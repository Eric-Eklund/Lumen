# Lumen + Cullimingo: sammanslagningsplan

Utredning och plan för att flytta Lumens backup-garantier in i Cullimingo
(Flutter-baserad culling-app, AGPL-3.0), så att ett program täcker hela flödet
kort → SSD → NAS → culling → export.

Daterad 2026-09-18. Underlaget är läst från Cullimingo på commit `e47cd02`
(PR #1-huvudet) och Lumen på `1318910`.

---

## 1. Utgångsläge

| | Lumen | Cullimingo |
|---|---|---|
| Språk | Go | Dart / Flutter |
| Licens | MIT | AGPL-3.0-or-later + CLA |
| Form | CLI + TUI, körbar från cron | GUI, ett fönster, ingen CLI |
| Storlek | 18 585 rader kod | 38 365 rader (exkl. genererat) + 20 051 test |
| Styrka | Ofelbar, obevakad backup | Culling, förhandsvisning, XMP, export |

Cullimingos ingest gör redan kort → en eller två destinationer i en läsning,
med SHA-256-verifiering, namnmallar, sidecar-hantering och dagsfiltrering.
Den skriver aldrig över en befintlig fil och raderar sin egen kopia vid
misslyckad verifiering. Etiken matchar Lumens.

---

## 2. Vad Cullimingo saknar

Granskningen hittade sex luckor. De tre första är de allvarliga.

### 2.1 Tyst filbortfall vid skanning

`folder_scanner.dart` hoppar över oläsbara poster med `handleError` och
loggar bara en varning. Dessutom bryts hela listningen av en tidsgräns:

```dart
const _scanStallTimeout = Duration(seconds: 8);
```

Ett kort som hänger mitt i listningen ger en ingest-plan med färre filer än
kortet har. Sammanfattningen visar kopierade, överhoppade, konflikter och
misslyckade. Ingen av dem räknar det som aldrig kom med i planen.
"Copied & verified: 812" kan vara sant medan bild 813 aldrig var påtänkt.

Lumens motsvarighet returnerar `[]Unreadable` och avslutar med
`incompleteSourceError`, alltså exitkod skild från noll.

### 2.2 Ingen destinationsvakt

Det finns ingen kontroll av att destinationsroten fortfarande finns, varken
före eller under en körning. Roten kommer från mappväljaren och skickas rakt
in. Tre utfall när NAS försvinner:

1. **Monteringen blir en tom lokal katalog.** `createSync(recursive: true)`
   återskapar trädet på systemdisken. Skrivningen lyckas, SHA-256 lyckas,
   dialogen säger "Copied & verified". Systemdisken fylls, NAS är tom, helt
   tyst. **Farligast.**
2. **Hård NFS-montering som hänger.** Skrivningarna blockerar i kärnan utan
   tidsgräns. Avbryt-knappen sätter bara en flagga som hindrar *nya* kopior;
   pågående returnerar aldrig. Dödar man appen ligger halva filer kvar.
3. **SMB/gvfs som returnerar fel.** Undantaget fångas, halvfilen raderas,
   filen räknas som misslyckad. Detta fall är korrekt.

### 2.3 Ingen stall-detektor på skrivsidan

`verified_copy.dart` använder `openWrite()` utan tidsgräns, fyra kopior
parallellt. Lumens `copyStream` löser det så här, och designen är det som
ska porteras:

- Klockan och förloppsmätaren är **samma observation**: varje chunk
  destinationen tar emot knuffar tidsgränsen framåt. "Baren rörde sig" och
  "monteringen lever" är ett och samma faktum.
- Pollning med ticker, inte en timer som armeras per chunk.
- Vid stall: transfern **överges, dödas inte** (en blockerad syscall går inte
  att avbryta i vare sig Go eller Dart). `onAbandoned` städar bort den halva
  filen när anropet till slut returnerar.
- Tidsgränsen mäter **tystnad, inte filstorlek**. Därför är ett lågt värde
  säkert även för en 2 GB-video.

### 2.4 Ingen verifiering i efterhand

Cullimingo hashar en gång, vid skrivningen, och aldrig mer. Bevisat lokalt
2026-09-18 med Lumen: en ändrad byte i en NAS-fil ger

```
status : Missing from NAS : 0          ← ser ingenting
verify : DSC_0003.NEF — [NAS hash mismatch]
         Error: 1 of 7 file(s) failed verification
         exitkod 1
```

Storleken är oförändrad, så en jämförelse på namn och storlek är blind för
det. Bara en omhashning hittar det.

### 2.5 Ingen fri-utrymmeskontroll

Lumen har `copyop.CheckSpace`, som grupperar rötter per filsystem så en delad
disk inte dubbelräknas. Cullimingo har ingen motsvarighet.

### 2.6 Ingen tvärdatumsökning

Cullimingo jämför bara exakt destinationssökväg. Ändrar man namnmall kopieras
allt om som dubbletter. Lumens `MissingFromDest` söker på namn och storlek i
hela destinationsträdet, och bekräftar träffen mot capture-time innan den
hoppar över filen.

---

## 3. Resescenariot

Målet: kort → SSD på resan, synk till NAS när uppkoppling finns.

**Cullimingo klarar det inte idag.** Ingest stöder två destinationer i
*samma* läsning, men det finns inget uppskjutet andra steg. Det är precis
Lumens fas 2, och det är det största enskilda nya arbetet.

Lumens beteende, verifierat lokalt 2026-09-18:

```
NAS borta  → copy  → 7 filer till SSD, fas 2 skjuts upp,
                     "Connect to VPN ... then run: lumen sync"   exit 0
NAS åter   → sync  → 7 filer kopierade                           exit 0
             verify→ All 7 files verified OK                     exit 0
             copy  → "already up to date", inget kopieras        exit 0
```

Viktig designregel att bevara: fas 2 läser **alltid om från SSD**, aldrig
från kamerapaths. Det är därför man kan koppla ur kortet mellan stegen.

---

## 4. Kostnad

Det mesta av Lumen behöver aldrig portas. Terminalgränssnittet,
förhandsvisningen och konfigurationen har Cullimingo egna svar på.

| Lumen-paket | Kodrader | Portas? |
|---|---|---|
| `scan` | 1091 | Ja, jämförelsemotorn är hjärtat |
| `copyop` | 616 | Ja, stall-detektor och utrymmeskoll |
| `status` | 470 | Ja, bantad |
| `verify` | 387 | Ja |
| `progress` | 280 | Bara för huvudlöst läge |
| `checksum` | 42 | Nej, `crypto` finns |
| `tui` `ui` `preview` `devices` `config` | 6261 | Nej |

Netto cirka 2 800 rader logik, minus det Cullimingo redan har (EXIF-läsning,
enhetsdetektering, hashning). Realistiskt **1 800 till 2 500 rader ny Dart
plus ungefär lika mycket test**. Ett par veckors fokuserat arbete.

### Verify i cron går att lösa rent

`verified_copy.dart` importerar bara `dart:io` och `crypto`, alltså inget
Flutter. Det enda som binder skannern till Flutter är loggningen via talker.
Abstraheras den kan man bygga en andra frontend med `dart compile exe` under
`bin/`, som delar kärna med gränssnittet. Samma form som Lumen har idag med
`cmd/lumen` ovanpå `internal/`.

---

## 5. Planen

### Fas 0 — Säkerhetsnätet (~300 rader)

Måste komma först. Utan den kan verktyget skriva din backup till systemdisken
utan att säga något.

- Destinationsvakt: roten finns, ligger på förväntad enhet, kontrolleras om
  mellan filerna, inte bara vid planering.
- Skannern returnerar antalet oläsbara poster och om listningen bröts av
  tidsgränsen.
- Båda talen syns i ingest-sammanfattningen.

*Kan gå uppströms.* Matchar Niels egen etik: i ingest rapporterar varje fil
redan sitt sämsta utfall, just för att en bild som tappade sin sidecar inte
ska räknas som helt lyckad.

### Fas 1 — Stall-detektor (~400 rader)

Portera designen från `copyop.copyStream` enligt 2.3. Byt strömskrivningen
mot `RandomAccessFile` så varje chunk ger ett await att mäta på.
Vakthundsmönstret finns redan i `preview_pool.dart`.

*Kan gå uppströms, men svårare att argumentera för.*

### Fas 2 — Jämförelsemotorn (~900 rader)

Portera `MissingFromDest` med tvådatumsökning och capture-time-bekräftelse,
plus `SplitStable` och `CheckSpace`. Grunden som både uppskjuten synk och
verify står på.

*Här lämnar du uppströms och gafflar.*

### Fas 3 — Resescenariot (~500 rader)

SSD blir en uttalad mellanlagring. En synkvy visar vad som saknas på NAS.
En monteringsvakt kör synken när NAS dyker upp. Kopieringen till NAS läser
alltid om från SSD.

### Fas 4 — Verify i två frontend (~600 rader)

Bryt ut loggningen bakom ett gränssnitt. Lägg verify-passet i kärnan med
reservsökningen via namn och storlek (`findCopy`). Bygg både en gränssnittsvy
och en `dart compile exe`-binär för cron. Binären ärver exitkoderna, vilket
är hela poängen.

### Fas 5 — Avveckling

Kör båda verktygen parallellt mot samma kort under några shoots och jämför
utfallen fil för fil. Lumens `TestCopyAndVerifyAgree` är förlagan: bevisa att
de fattar samma beslut innan du slutar köra det gamla.

---

## 6. Varför inte tvärtom

Att lägga ett Cullimingo-liknande gränssnitt ovanpå Lumen är ungefär tolv
gånger mer arbete, i den svårare riktningen.

| Riktning | Ny kod |
|---|---|
| Lumens backup-logik in i Cullimingo | ~2 500 rader Dart |
| Cullimingos gränssnitt ovanpå Lumen | ~30 000 rader Go |

Det som skulle behöva byggas om i Go: culling-vyn (11 579 rader),
XMP-motorn (8 081), export (1 345), inspector (1 034) och hela
förhandsvisningskedjan med tvånivåcache, isolate-pool och LibRaw/libvips
(cirka 2 100). Därtill saknar Go ett moget GUI-ramverk för det här, och
LibRaw och libvips skulle behöva cgo-bindningar från grunden.

Lumens TUI på 4 418 rader har redan datumträd, rutnät och Kitty-grafik. Den
är inte vägen till Cullimingos gränssnitt, den är en annan produkt.

---

## 7. Att bygga och verifiera lokalt

### Lumen (verifierat 2026-09-18, allt grönt)

```bash
go build -o lumen ./cmd/lumen
go test ./...                                   # 11 paket, alla OK

rm -rf testdata/camera testdata/ssd testdata/nas
go run testdata/make_testdata.go

# Resescenariot: NAS borta
mkdir -p testdata/ssd
printf 'y\n' | ./lumen --config testdata/config.toml copy    # 7 → SSD, fas 2 uppskjuten

# NAS tillbaka
mkdir -p testdata/nas
./lumen --config testdata/config.toml sync                   # 7 → NAS
./lumen --config testdata/config.toml verify                 # All 7 OK, exit 0
printf 'y\n' | ./lumen --config testdata/config.toml copy    # idempotent

# Bitröta: verify fångar det, status gör det inte
printf 'X' | dd of=testdata/nas/photos/2026/2026-03/2026-03-25/DSC_0003.NEF \
             bs=1 seek=100 conv=notrunc
./lumen --config testdata/config.toml verify                 # exit 1, hash mismatch
./lumen --config testdata/config.toml status                 # Missing: 0  ← blind
```

### Cullimingo (kräver din egen maskin: display + native libs)

```bash
# Native beroenden
sudo apt install libraw-dev libvips-dev libheif-dev libheif-plugin-aomenc \
                 clang cmake ninja-build pkg-config libgtk-3-dev \
                 libsecret-1-dev libjsoncpp-dev patchelf exiftool
# eller: brew install libraw vips

git clone https://github.com/nielsfranke/Cullimingo && cd Cullimingo
git fetch origin refs/pull/1/head:raw-fix && git checkout raw-fix
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter test                                    # 988 tester
flutter analyze && dart format --output=none --set-exit-if-changed .
flutter run -d linux
```

AppImage lokalt:

```bash
flutter build linux --release
tool/bundle_linux.sh
tool/build_appimage.sh            # → build/linux/Cullimingo-x86_64.AppImage
```

### Sluttest för hela flödet (efter fas 3)

Kör mot en riktig kortläsare och en NAS-montering du kan koppla ned:

1. Sätt i kort, ingest till SSD med NAS nedmonterad. Räkna filerna.
2. Montera NAS, kör synken. Räkna filerna på NAS.
3. Kör verify. Ska vara grön.
4. Ändra en byte på NAS. Kör verify igen. Ska bli röd med exitkod skild
   från noll.
5. Koppla ned NAS **mitt under** en synk. Inget får hamna på systemdisken,
   och sammanfattningen måste säga vad som inte kom fram.
6. Gör en fil oläsbar på kortet (`chmod 000`). Ingest måste rapportera det i
   sammanfattningen, inte bara i loggen.

Punkt 5 och 6 är de som faller idag.
