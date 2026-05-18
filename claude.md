# Entro — Prosjektkontekst for ny Claude-sesjon
**Programnamn:** SXI-generatoren  
**Firma:** Entro AS  
**Versjon:** 2.0.9 | Single-file HTML applikasjon

---

## Prosjektoversikt

SXI-generatoren er eit nettbasert oppmålingsverktøy frå Entro AS for energimerking av norske bygg. Brukaren lastar opp ei planteikning, teiknar soner som polygon, og eksporterer ein SXI-fil til SIMIEN (norsk energiberegningsprogram).

**Arkitektur:** Éi enkelt HTML-fil (`index.html`) med all kode inline. Ingen bygg-steg, ingen avhengigheiter utanom Three.js r128 frå CDN.

---

## Datamodell

```javascript
floors = [{
  id, name,
  bgImg,          // Image-objekt (ikkje lagra i JSON — lagrast som JPEG base64)
  bgImgData,      // base64 JPEG for lagring
  pdfDoc, pdfPage, totalPages,
  mmPerImgPx,     // kalibrering per etasje (null = brukar global)
  ownCal,         // boolean — brukar eigen kalibrering (ikkje global)
  defaultHoyde,   // etasjehøgde i meter (styrar 3D-stakkinga)
  sc, offX, offY, // zoom/pan for canvas
  zones: [...]
}]

zones = [{
  name,
  pts: [{x, y}],  // biletepiksel-koordinatar
  windows: [...],
  skillevegg: Set([segIdx, ...]),  // veggar som er skiljeveggar
  groupId,        // kopling mellom soner (berre SIMIEN-gruppering)
  bygkat,         // bygningskategori
  hoyde,          // romhøgde (null = brukar etasjens defaultHoyde)
  himling,        // himlingshøgde (number eller null)
  gulvtype, taktype,
  takvinkel,      // grader (null = flatt)
  byggeaar,
  uVegg, uTak, uGulv, uVindu, n50  // manuelle U-verdi-overrides
}]

windows = [{
  type,           // 'vindauge' eller 'dor'
  name,
  breddeMm, hoyMm,
  antal,
  t0, t1,         // 0-1 relativ posisjon langs segmentet (senter-basert)
  segIdx,         // indeks i calcSegments()
  dir,            // 'Nord', 'Sør' osv.
  uVerdi          // manuell U-verdi (undefined = Entro-standard)
}]
```

**Globale variablar:**
```javascript
let globalNorth = 0;           // nord-retning i grader
let globalMmPerImgPx = null;   // global kalibrering
let _nextZoneId = 0;           // pre-increment → første sone = 1
let _nextGroupId = 0;
let _nextFloorId = 2;
let activeFloor = 0;
let zones = [];                // alias til floors[activeFloor].zones
let mmPerImgPx = null;         // alias (global eller eigen per etasje)
let renderer3 = null;          // global — trengst for tema-toggle
let scene3 = null;
let camera3 = null;
```

---

## Viktige funksjonar

```javascript
s2i(sx, sy)     // skjermkoord → biletepiksel: {x:(sx-offX)/sc, y:(sy-offY)/sc}
i2s(ix, iy)     // biletepiksel → skjermkoord: {x:ix*sc+offX, y:iy*sc+offY}

getHoyde(z)     // romhøgde for ei sone — slår opp etasje, brukar defaultHoyde som fallback
calcSegments(pts, fhM)  // bereknar veggsegment med retning, lengd, areal
expandWindows(w, segLenM)  // ekspanderer antal>1 til individuelle vindauge med t0/t1
getAreaM2(z)    // areal i m² (null om ikkje kalibrert)
getTotalAreaM2(z)  // sum av alle kopla soner
uniqueWinName(name, existingWindows)  // legg til (2), (3) osv. for duplikat
```

**Koordinatsystem:**
- `pts[].x/y` er alltid i **original biletepiksel** (ikkje nedskalerte)
- `sc` og `offX/offY` konverterer til skjermkoordinatar
- `mmPerImgPx` = mm per original biletepiksel

---

## Kalibrering

**To metodar:**
1. **Lengde** — klikk start→slutt, tast inn mm
2. **Areal** — klikk på eksisterande sone (brukar sonens pikselareal)

**Flyt:** Dialog opnar seg først (vel metode) → teikn på canvas → dialog kjem att for å taste inn verdi

**Per-etasje kalibrering:** `f.ownCal = true` gjer at etasjen brukar `f.mmPerImgPx` i staden for `globalMmPerImgPx`

---

## SXI-eksport — kritiske detaljar

### Dørformat (VIKTIG — vart feil tidlegare)
```xml
<!-- RIKTIG -->
<door uvalue="2.50" area="2.50" type="Standardvalg" gate="no" id="door#1" name="Dør 1" comment=""></door>

<!-- FEIL (gamle format) -->
<door number="1" height="2.100" width="0.900" ...></door>
```

### makeProfile() — kritisk feil som vart fiksa
Siste slot MÅ vere `2345-0000`, IKKJE `2345-2400`. Feil her gjer at SIMIEN ikkje les profilen.

### Energimerke per bygningskategori
Éin `<energymark26>` per unik `building_type`. Namn: `"Energimerke Kontor"`, `"Energimerke Skole"` osv. `total_floor_area` er summen av soner i den kategorien.

### Bygningskategoriar (z.bygkat → SIMIEN)
```javascript
'Småhus'      → type:'Småhus',      subtype:'Enebolig'
'Boligblokker'→ type:'Boligblokk',  subtype:'Leilighet'
'Barnehager'  → type:'Barnehage',   subtype:'Barnehagebygning'
'Kontorbygg'  → type:'Kontorbygning', subtype:'Kontorer, enkle'
'Skolebygg'   → type:'Skolebygning',  subtype:'Undervisningslokaler'
// ... (sjå BYGKAT_SXI i koden)
```

### Panelovnar
```xml
<new_local_heater_dir_el heater_type="frittstaende" name="Panelovner"
  capacity="[totalBra*0.05]" convective_share="0.50" ...></new_local_heater_dir_el>
```
50 W/m² — `capacity` i kW = `totalBra * 0.05`

---

## Kopling av soner (groupId)

- `groupId` er ein string (`'g1'`, `'g2'` osv.)
- Berre SIMIEN-gruppering — geometri og areal bereknast per sone
- Berre soner med **same bygningskategori** kan koplast
- Kvar kopla sone kan ha **eigen tak- og gulvtype**
- Kopla soner vises med lilla boks i sidepanelet

---

## Autolagring vs. manuell lagring

**Manuell lagring (.entro):** Full oppløysing PNG — ingen skalering
**Autolagring (localStorage):** JPEG 85% av `bgImg` slik den er — ingen skalering

Begge lagrar `bgImgData` og brukar same `sc`/`offX`/`offY`. Ingen `bgImgScale` lenger (vart fjerna etter fleire bugg-rundar).

---

## Kjende manglar / ikkje implementert

- Snapping mellom soner på ulike etasjar
- Import av eksisterande SXI
- Validering av overlappande soner
- Offline-modus (Three.js krev CDN)

---

## Easter egg — Lumon Industries

Aktiverast ved å rotere nord-kompassen **3 fulle runder samanhengande (1080°)**. Kontinuerleg rotasjon — hopp >90° nullstillar teljaren. Deaktiverast ved **1 runde (360°)** i Lumon-modus.

Funksjonen `window._trackNorthRotation(newNorth)` vert kalla frå `setGlobalNorth()`.

---

## Utviklingsworkflow

```bash
# Etter kvar endring — syntax-sjekk
python3 -c "
import re, subprocess, tempfile, os
with open('index.html') as f: html = f.read()
m = re.search(r'<script>([\s\S]*?)</script>\s*</body>', html)
with tempfile.NamedTemporaryFile(mode='w', suffix='.js', delete=False) as f:
    f.write(m.group(1)); fname = f.name
r = subprocess.run(['node', '--check', fname], capture_output=True, text=True)
print(r.stdout or 'OK'); print(r.stderr or '')
os.unlink(fname)
"
```

**Viktig:** Alltid `cp /mnt/user-data/outputs/index.html /home/claude/index.html` først, arbeid i `/home/claude/`, kopier til `/mnt/user-data/outputs/` når ferdig.

---

## Prosjektinstruks til Claude (lim inn i Project Instructions)

```
Eg jobbar med SXI-generatoren, eit nettbasert oppmålingsverktøy (single-file HTML) for
energimerking av norske bygg. Programmet integrerer med SIMIEN via SXI-eksport.

Programnamn: SXI-generatoren (firmaet heiter Entro AS, ikkje programmet)
Noverande versjon: 2.0.9

Prinsipp:
- Sonekopling (groupId) påverkar berre SIMIEN-gruppering, ikkje geometri
- Global kalibrering som standard, per-etasje som opt-in
- SXI-output må validerast mot referansefiler — SIMIEN er kresen på feltnamn
- UI-forenkling er foretrekt framfor kompleksitet
- Alltid syntax-sjekk med node --check etter endringar
- Arbeid alltid i /home/claude/, lever til /mnt/user-data/outputs/

Domene: sone, etasje, takvinkel, kalibrering, fasade, skiljevegg,
SXI-eksport, himling, bygningskategori, SIMIEN, energimerking

Viktig bughistorikk:
- Dørformat: <door uvalue area type gate> (IKKJE number/height/width)
- makeProfile(): siste slot = 2345-0000 (IKKJE 2345-2400)
- findWindowAtScreen() må berre søke i aktiv etasje
- bgImgScale er fjerna — autolagring og manuell lagring er no identiske
```
