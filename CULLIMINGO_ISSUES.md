# Cullimingo: åtgärdslista för ingest

Sex konkreta problem i Cullimingos ingest-kedja, hittade vid granskningen
2026-09-17/18. Avsedd att arbeta ur lokalt, en post i taget.

Referenser pekar på Cullimingo commit `e47cd02` (huvudet på PR #1). RAW-
förhandsvisningsbuggen som PR #1 löser är **inte** med här, den är åtgärdad.

Prioritetsordning: G1, G2, G5 gör ingest säker mot lokal disk. G3 krävs
innan du pekar den mot NAS. G4 och G6 är större och kan vänta.

| ID | Problem | Allvar | Omfattning |
|---|---|---|---|
| G1 | Tyst filbortfall vid skanning | Kritisk | ~150 rader |
| G2 | Ingen destinationsvakt | Kritisk | ~150 rader |
| G3 | Ingen stall-detektor på skrivsidan | Hög | ~400 rader |
| G4 | Ingen verifiering i efterhand | Hög | ~600 rader |
| G5 | Ingen fri-utrymmeskontroll | Medel | ~120 rader |
| G6 | Ingen tvärdatumsökning | Medel | ~500 rader |

---

## G1 — Tyst filbortfall vid skanning

**Allvar: kritisk.** Detta är tillståndet där någon formaterar ett kort.

### Symptom

Ingest-sammanfattningen säger att allt importerats medan filer ligger kvar
på kortet, omnämnda bara i en loggfil.

### Rotorsak

Två oberoende mekanismer i `lib/features/library/data/folder_scanner.dart`:

1. **Rad 142–146.** `handleError` sväljer `FileSystemException` per post och
   loggar en varning. Posten försvinner ur resultatet.
2. **Rad 147–155.** Hela listningen bryts av en tidsgräns och returnerar det
   som hunnit hittas:

   ```dart
   const _scanStallTimeout = Duration(seconds: 8);   // rad 20
   ```

   Samma tidsgräns används även per `stat()` på rad 176 och i EXIF-läsningen
   på rad 231 och 260.

`scanFolderFast` (rad 109) returnerar `List<ScannedFile>` och har ingen kanal
för "detta gick inte att läsa". `IngestSummary` (rad 238 i
`ingest_service.dart`) räknar kopierade, överhoppade, konflikter och
misslyckade. Ingen av dem täcker det som aldrig kom med i planen.

### Reproduktion

```bash
mkdir -p /tmp/kort/DCIM && cp nagon.NEF /tmp/kort/DCIM/
cp nagon.NEF /tmp/kort/DCIM/hemlig.NEF && chmod 000 /tmp/kort/DCIM/hemlig.NEF
# Kör ingest mot /tmp/kort. Sammanfattningen nämner inte hemlig.NEF.
```

### Åtgärd

Låt `scanFolderFast` returnera en post i stället för en lista:

```dart
typedef ScanResult = ({
  List<ScannedFile> files,
  List<String> unreadable,     // sökvägar handleError svalde
  bool truncated,              // tidsgränsen bröt listningen
});
```

Tråda båda fälten genom `scanSources` (rad 188 i `ingest_service.dart`) och
in i `IngestSummary`. Visa dem i `_summaryView()` (rad 812 i
`ingest_dialog.dart`) bredvid de befintliga raderna på rad 835–838. En
trunkerad skanning ska vara en varning i röd text, inte en sifferrad, för den
betyder att antalet filer är okänt.

`allOk` (rad 263) måste bli falskt när något är oläsbart eller trunkerat.
Det är hela poängen: sammanfattningen får aldrig påstå mer än den vet.

### Test

- Katalog med en fil på `chmod 000` ger `unreadable.length == 1` och
  `allOk == false`.
- Konstruerad ström som aldrig avslutas ger `truncated == true`.

### Lumen-referens

`scan.Walk` returnerar `[]Unreadable` vid sidan av filerna, och
`runCopy`/`runDirect` returnerar `incompleteSourceError` så kommandot
avslutar med kod skild från noll.

### Uppströms?

Ja. Passar Niels egen etik: i `runIngest` (rad 279) rapporterar varje fil
redan sitt *sämsta* utfall, just för att en bild som tappade sin sidecar inte
ska räknas som helt lyckad.

---

## G2 — Ingen destinationsvakt

**Allvar: kritisk.** Tyst dataförlust när en nätverksresurs försvinner.

### Symptom

NAS försvinner, mountpointen blir en tom lokal katalog, och hela backupen
skrivs till systemdisken. Verifieringen godkänner den, eftersom det är en
riktig lokal fil. Dialogen säger "Copied & verified".

### Rotorsak

Ingen kontroll av destinationsroten finns någonstans. Roten kommer från
mappväljaren (`_dest`, rad 49 i `ingest_dialog.dart`) och skickas rakt in på
rad 342. Därefter, i `lib/core/files/verified_copy.dart` rad 145:

```dart
File(dest).parent.createSync(recursive: true);
```

Den återskapar glatt hela trädet var som helst.

Tre utfall när NAS går offline:

| Läge | Utfall |
|---|---|
| Mountpoint blir tom lokal katalog | Allt till systemdisken, tyst. **Värst.** |
| Hård NFS som hänger | Se G3 |
| SMB/gvfs som felar | Korrekt: halvfil raderas, räknas som misslyckad |

### Reproduktion

```bash
mkdir -p /tmp/nas && sudo mount -t tmpfs none /tmp/nas
# ingest till /tmp/nas/foton, avbryt, sudo umount /tmp/nas
# kör ingest igen → filerna hamnar i den tomma lokala /tmp/nas
```

### Åtgärd

En `DestinationGuard` som tas innan körningen och kontrolleras om mellan
filerna, inte bara vid planering:

1. Vid start: spara rotens `stat().dev` (enhets-id) via en FFI-`stat`, eller
   som fattigmansvariant en dold sentinel-fil i roten som måste finnas kvar.
2. Före varje fil: roten finns och har samma `dev`. Annars avbryt hela
   körningen med ett eget utfall, `CopyOutcome.destinationLost`.
3. Aldrig `createSync(recursive: true)` ovanför den bekräftade roten. Skapa
   bara underkataloger *inuti* den.

Sentinel-varianten är enklast och plattformsoberoende: en `.cullimingo-root`
som skrivs vid första körningen och läses före varje fil. Försvinner den har
monteringen bytts ut.

### Test

Tillfällig katalog som destination, radera den mitt i en körning, kräv att
resterande filer rapporteras som förlorad destination och att inget skrevs
utanför den ursprungliga roten.

### Lumen-referens

`config.RootAvailable` (`internal/config/config.go` rad 229). Notera att
Lumen har en besläktad exponering här: även Lumen skapar roten vid första
kopieringen. Skillnaden är att `status` visar fri disk per rot, så ett fel
syns. Bygg hellre vakten starkare än Lumens.

### Uppströms?

Ja, och det är den mest övertygande av alla sex.

---

## G3 — Ingen stall-detektor på skrivsidan

**Allvar: hög.** Krävs innan ingest pekas mot NAS.

### Symptom

Hård NFS-montering försvinner. Skrivningarna blockerar i kärnan utan
tidsgräns. Fyra kopior hänger parallellt. Avbryt-knappen gör ingenting.
Dödar man appen ligger halvskrivna filer kvar, eftersom uppstädningen aldrig
körs.

### Rotorsak

`lib/core/files/verified_copy.dart` rad 146:

```dart
sinks.add(File(dest).openWrite());
```

`IOSink.add()` buffrar utan mottryck, och det finns ingen tidsgräns någonstans
i `_streamCopy` (rad 138). Samtidigheten är fyra (`concurrency`, rad 283 i
`ingest_service.dart`).

Avbrytningen är dessutom skenbar: `_cancel()` (rad 361 i `ingest_dialog.dart`)
sätter en flagga som bara hindrar *nya* kopior från att starta. Det pågående
`await copier(...)` returnerar aldrig.

### Åtgärd

Portera designen från Lumens `copyStream`. Fyra egenskaper måste följa med:

1. **Klockan och förloppsmätaren är samma observation.** Varje chunk
   destinationen tar emot knuffar tidsgränsen framåt. "Baren rörde sig" och
   "monteringen lever" är ett och samma faktum.
2. **Pollning med ticker**, inte en timer som armeras om per chunk. Lumen
   använder en fjärdedel av tidsgränsen, klämd mellan 10 ms och 1 s.
3. **Överge, döda inte.** En blockerad syscall går inte att avbryta, varken i
   Go eller Dart. Låt den ligga och städa bort halvfilen när den returnerar.
4. **Mät tystnad, inte filstorlek.** Därför är ett lågt värde säkert även för
   en 2 GB-video. Detta var en riktig bugg i Lumen en gång: tidsgränsen var
   armerad före överföringen och blev en deadline för hela filen.

I Dart: byt `openWrite()` mot `RandomAccessFile` så varje `writeFrom` ger ett
await att mäta på. Vakthundsmönstret finns redan i huset, se
`lib/core/isolates/preview_pool.dart` med sin timer per utdelat jobb.

Fixa avbrytningen samtidigt, den hör ihop.

### Test

En destination som är en FIFO ingen läser från, eller en monteringspunkt som
rycks undan. Kräv att kopian ger upp inom tidsgränsen och att ingen halvfil
blir kvar.

### Lumen-referens

`copyop.copyStream` (`internal/copyop/copyop.go` rad 183) och
`stallCheckInterval` (rad 246).

### Uppströms?

Kanske. Svårare att argumentera för, men ditt användningsfall med kort direkt
till NAS är precis det som motiverar den.

---

## G4 — Ingen verifiering i efterhand

**Allvar: hög.** Detta är den enda kontrollen mot bitröta.

### Symptom

Cullimingo hashar en gång, vid skrivningen, och aldrig mer. En fil som
degraderar på disken upptäcks aldrig.

Bevisat lokalt med Lumen 2026-09-18, en ändrad byte i en NAS-fil:

```
status : Missing from NAS : 0            ← ser ingenting
verify : DSC_0003.NEF — [NAS hash mismatch]
         Error: 1 of 7 file(s) failed verification
         exitkod 1
```

Storleken är oförändrad, så jämförelse på namn och storlek är blind.

### Åtgärd

Ett fristående verify-pass i kärnan, inte i dialogen:

- Källdrivet: kortet eller mellanlagringen är auktoritet, inte destinationen.
  Att verifiera ett kort kontrollerar det kortets filer, inte hela NAS.
- Hashar hela filen på varje sida som är tillgänglig. Inget samplas.
- **En omonterad destination hoppas över, den underkänns inte.** Men den
  måste namnges i resultatet, annars står "alla filer OK" för något ingen
  tittat på.
- Reservsökning via namn och storlek när sökvägen inte matchar, annars
  rapporteras varje äldre backup som saknad.

### Här behövs en andra frontend

Ett fönster kan inte köra första söndagen i månaden medan du sover. Lösningen
är inte att behålla Go, utan att bygga en andra frontend på samma Dart-kärna:

- `lib/core/files/verified_copy.dart` importerar redan bara `dart:io` och
  `crypto`, alltså inget Flutter.
- Det enda som binder skannern till Flutter är loggningen via talker
  (`lib/core/logging/app_logger.dart` importerar `flutter/foundation.dart`).
  Abstrahera den bakom ett litet gränssnitt.
- Lägg sedan `bin/cullimingo_verify.dart` och bygg med `dart compile exe`.
  Binären ärver exitkoderna, vilket är hela poängen med den.

Samma form som Lumen har idag med `cmd/lumen` ovanpå `internal/`.

### Lumen-referens

`verify.verifyAll` (`internal/verify/verify.go` rad 126), `findCopy` (rad 321)
och `unmountedRoots` (rad 282).

### Uppströms?

Nej, troligen inte. Ett CLI i en GUI-app är utanför Cullimingos uttalade form.

---

## G5 — Ingen fri-utrymmeskontroll

**Allvar: medel.** Degraderar hyfsat, men i onödan.

### Symptom

Destinationen tar slut mitt i en körning. Varje resterande fil misslyckas en
och en i stället för att körningen stoppas innan den börjar.

### Rotorsak

Ingen motsvarighet finns. `IngestPlan.totalBytes` är redan beräknad, så
underlaget finns, det jämförs bara aldrig med något.

### Åtgärd

Kontroll före start, med två detaljer som är lätta att missa:

1. **Gruppera rötter per filsystem** så en delad disk inte dubbelräknas. Vid
   dubbel destination på samma disk krävs 2× utrymmet, vid olika diskar 1×
   vardera.
2. Dart saknar `statvfs`. Antingen FFI, eller läs `df -kP <rot>`. FFI är
   renare och behövs ändå för `dev`-kontrollen i G2, så gör dem tillsammans.

### Lumen-referens

`copyop.CheckSpace` (`internal/copyop/copyop.go` rad 574).

### Uppströms?

Ja, okontroversiell.

---

## G6 — Ingen tvärdatumsökning

**Allvar: medel.** Handlar om ordning och reda, inte om förlust.

### Symptom

Byter du namnmall kopieras allt om som dubbletter. En bild som redan ligger
under ett annat datum hittas inte.

### Rotorsak

`verifiedCopy` (rad 62) tittar bara på den exakta destinationssökvägen:
finns den och är identisk blir det `skipped`, finns den och skiljer sig blir
det `conflict`. Ingen letar i resten av trädet.

### Åtgärd

Portera Lumens tvåpassmetod. Den viktiga finessen är att namn och storlek
ensamt är en **svag** identitet: tre kort numrerar alla sin första bild
`DSC_0001`, och två av dem kan ha byte-exakt samma storlek. Lumen bekräftar
därför en träff mot capture-time innan den hoppar över filen.

- Pass 1 avgör allt som går att avgöra ur indexet och samlar de tvetydiga.
- Pass 2 läser capture-time bara för dessa tvillingar, med begränsad pool.
- Lika capture-time eller ingen på någondera sidan räknas som samma fil.
  Olika betyder olika bilder, och källan kopieras.
- En destination vars struktur redan matchar kostar **noll** extra läsningar.

Förutsättning: Cullimingo har redan EXIF-läsning (`scanExif`, rad 207 i
`folder_scanner.dart`) och `libraw_metadata.dart`, så halva jobbet finns.

### Lumen-referens

`scan.MissingFromDest` (`internal/scan/scan.go` rad 256), `IndexByNameSize`
(rad 336), `captureTimesOf` (`internal/scan/capture.go` rad 627) och
`sortBySrcOrder` (rad 318, återställer ordningen som tvåpassmetoden tappar).

### Uppströms?

Kanske, men det är en stor bit att be någon granska.

---

## Blir Lumen obsolet när dessa sex är fixade?

**Nej.** De sex gör Cullimingo **säker**. De gör den inte **tillräcklig**.

Efter alla sex saknas fortfarande två saker, och den första är den som
faktiskt skaver i ditt flöde:

### F1 — Uppskjuten synk till NAS (~500 rader)

Cullimingos ingest kopierar till en eller två destinationer i **samma
läsning**. Det finns inget andra steg som kan köras senare. Ditt resescenario,
kort till SSD nu och NAS när uppkoppling finns, går alltså inte.

Vad som behövs:

- SSD blir en uttalad mellanlagring med känt tillstånd.
- En synkvy som visar vad som saknas på NAS, byggd på G6:s jämförelsemotor.
- En monteringsvakt som kör synken när NAS dyker upp.
- Kopieringen till NAS läser **alltid om från SSD**, aldrig från kortets
  sökvägar. Det är därför man kan koppla ur kortet mellan stegen.

Lumens beteende att härma, verifierat 2026-09-18:

```
NAS borta  → copy  → 7 filer till SSD, fas 2 uppskjuten,
                     "Connect to VPN ... then run: lumen sync"
NAS åter   → sync  → 7 filer kopierade
             verify→ All 7 files verified OK
             copy  → "already up to date", inget kopieras
```

### F2 — Obevakad körning (ingår i G4)

Exitkoder och cron. Löses av `dart compile exe`-binären i G4, inte av något
i gränssnittet.

### Checklista för att lämna Lumen

| Krav | Täcks av |
|---|---|
| Ingest tappar aldrig filer tyst | G1 |
| Backupen kan inte hamna fel | G2 |
| Nätverksbortfall upptäcks | G3 |
| Bitröta upptäcks i efterhand | G4 |
| Disken tar inte slut mitt i | G5 |
| Inga dubbletter vid ommärkning | G6 |
| Kort till SSD nu, NAS senare | **F1** |
| Körbar från cron med exitkod | **F2** (i G4) |

Alla åtta ikryssade betyder att Lumen är obsolet. Sex av åtta betyder att
Cullimingo är säker att använda för ingest, men att du fortfarande behöver
Lumen för resescenariot och för den månatliga verifieringen.

Total omfattning för hela listan: ungefär 2 400 rader ny Dart plus test,
vilket stämmer med den tidigare uppskattningen på 1 800 till 2 500.
