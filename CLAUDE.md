# Soliday → Kugelmann → sevdesk · Project Briefing

## Projektübersicht

Ein browserbasiertes Tool das Soliday-Sonnensegel-Angebots-PDFs in strukturierte sevdesk-Angebote umwandelt und dabei die internen Namens- und Umstrukturierungsregeln von Sonnensegel Kugelmann anwendet.

**Deliverable:** `index.html` — Single-File HTML/JS, läuft komplett im Browser. Kein Backend, kein Build-Schritt. Bereitgestellt via GitHub Pages.

**Live:** https://julia-kugelmann.github.io/SoSeKu/

---

## Gesamtablauf (7 Schritte)

```
Schritt 1: PDF → SoSeKu-JSON (Extraktion via Claude API)
Schritt 2: Validierung + Anzeige (Summenprüfung, PDF-iframe, Chat-Feld)
Schritt 3: SoSeKu-Umstrukturierung (Transformation, Chat-Feld)
Schritt 4: Fragebogen (Wandaufbau, Zone, Dachterrasse)
Schritt 5: Preisberechnung (5A Segeltuch, 5B Wand, 5C Montage, 5D Finalisierung)
          → Vorschau finales SoSeKu-JSON
Schritt 6: Konvertierung SoSeKu-JSON → sevdesk-Struktur (JavaScript)
Schritt 7: Import nach sevdesk + direkter Link zum Angebot
```

---

## Schritt 1 — Extraktion → SoSeKu-JSON

Claude liest das Soliday-PDF **1:1** aus und erstellt das interne **SoSeKu-JSON**.
Dieses JSON ist noch nicht das finale sevdesk-JSON — es ist die rohe, unveränderte Soliday-Darstellung.

**Extraktions-Prompt liegt im Google Sheet** — editierbar ohne Code-Änderung.

**Modell:** `claude-haiku-4-5-20251001` → Fallback `claude-sonnet-4-6` bei Context-Fehlern.

### SoSeKu-JSON Felder

```json
{
  "kommission": "string",
  "datum": "string",
  "system": "SOLIDAY-ONE",
  "snap": false,
  "wellenlaenge_aufrollsystem_mm": 8559,
  "positionen": [
    {
      "pos": 1,
      "artikelnummer": "78936A",
      "name": "string",
      "beschreibung": "string",
      "preis_einheit": 63.71,
      "menge": 38.34,
      "einheit": "m²",
      "gesamt": 2442.64,
      "soliday_sektion": "Segel",
      "qm_segel": 38.34
    }
  ],
  "teuerungszuschlag_netto": 104.13,
  "netto_gesamt": 8516.54,
  "mwst": 1618.14,
  "brutto_gesamt": 10134.68
}
```

**Wichtig:** Mehrere Segel werden einzeln erfasst — jedes Segel mit eigener `qm_segel`-Angabe, nicht summiert.

`soliday_sektion` erfasst den originalen Soliday-Abschnittstitel (Segel, Segelzubehör, Hardware, Punkt A–D etc.)

---

## Schritt 2 — Validierung + Anzeige

### Summenprüfung (JavaScript, aus SoSeKu-JSON)

```
Σ (pos.preis_einheit × pos.menge) + teuerungszuschlag_netto  =  netto_gesamt
```

- **Differenz = 0,00 €** → weiter
- **Differenz ≠ 0,00 €** → automatisch neu extrahieren, gesamter Schritt 1 läuft erneut durch

### Anzeige

- **SoSeKu-JSON** visuell aufbereitet
- **PDF im `<iframe>`** — direkt zur Endsumme gescrollt
- **Chat-Feld** — Nutzer kann Korrektur-Prompt eingeben → Extraktion läuft nochmal mit diesem Hinweis

---

## Schritt 3 — SoSeKu-Umstrukturierung

Transformation des SoSeKu-JSONs nach Kugelmann-Regeln:

- Positionen umbenennen (Kugelmann-Namen)
- Positionen zusammenfassen (gleiche Art.-Nr. → mergen)
- In Sektionen einteilen
- Weglass-Positionen entfernen
- Neue Positionen mit Preis = 0 anlegen (Preis kommt in Schritt 5)
- Fracht-Positionen anlegen (Preis kommt in Schritt 5)
- Montage-Positionen anlegen (Preis kommt in Schritt 5)

**Transformations-Prompt liegt im Google Sheet** — editierbar ohne Code-Änderung.

Entfernte Positionen werden **gespeichert** (mit Preisen) für die Summenprüfung.

### Sektionen (Reihenfolge fix)

```js
[
  { id: "segeltuch",         name: "Segeltuch" },
  { id: "segelsystem",       name: "Segelsystem" },
  { id: "masten",            name: "Masten" },
  { id: "fundamente",        name: "Fundamente" },
  { id: "wandbefestigungen", name: "Wandbefestigungen" },
  { id: "zubehoer",          name: "Zubehör" },
  { id: "montage",           name: "Montage und Fracht" },
]
```

Leere Sektionen werden weggelassen.

### Summenprüfung nach Schritt 3

```
SoSeKu-Summe + Summe entfernter Positionen  =  netto_gesamt (Schritt 1)
```

Differenz muss 0,00 € sein — sonst wurde versehentlich etwas falsch behandelt.

### Anzeige

- Transformiertes JSON visuell aufbereitet
- Summenvergleich sichtbar (SoSeKu-Summe vs. Soliday-Gesamtsumme)
- **Chat-Feld** — Nutzer kann Korrektur-Prompt eingeben

---

## Schritt 4 — Fragebogen

Formular mit vorausgefüllten Defaultwerten.

### Wandaufbau *(Radio-Buttons, Einfachauswahl)*
- ● Ziegel/Beton bis 10 cm Dämmung *(Default)*
- ○ Ziegel/Beton bis 18 cm Dämmung
- ○ Holzbauweise bis 10 cm Dämmung
- ○ Holzbauweise bis 18 cm Dämmung

### Anfahrtszone *(Einfachauswahl)*
- Zone 1 — bis 15 min
- Zone 2 — bis 30 min *(Default)*
- Zone 3 — bis 45 min
- Zone 4 — bis 60 min
- Zone 5 — bis 75 min
- Zone 6 — bis 90 min

### Dachterrasse
- ☐ Dachterrasse *(nicht angehakt = Default)*

### Freitextfeld
Für spätere Zusatzangaben.

---

## Schritt 5 — Preisberechnung & Finalisierung

**Input:** SoSeKu-JSON (Schritt 3) + Formulardaten (Schritt 4) + Google Sheet

### 5A — Segeltuch

Pro Segel eine eigene Position.

**Formel (JS):**
```
(qm_segel + 2) × Default-Stoffpreis (aus Google Sheet)  =  Einzelpreis
```

Google Sheet enthält pro System:
- Welcher Stoff ist der Default
- Welche Stoffe sind verfügbar (mit Preis/m²)

**Positionsdarstellung:**
- **Name:** `Segeltuch - Austronet 950` *(Default-Stoff des Systems)*
- **Beschreibung:**
  ```
  Alternativ:
  0.000,- € Austronet 950
  0.000,- € Austrosail Lotus Linear
  0.000,- € Austrosail Lotus Linear Sonderfarben
  0.000,- € Austrosail Lotus Strahlenschnitt
  0.000,- € Austrosail Lotus Strahlenschnitt Sonderfarben
  Farbe nach Wahl aus Farbfächer des Herstellers
  ```

Ersetzt die Segeltuch-Platzhalter-Position aus Schritt 3.

### 5B — Wandbefestigungen

Für **jede** Wandplatte im Angebot:

**Formel (JS):**
```
neuer Einzelpreis = original Einzelpreis (aus SoSeKu-JSON)
                 + (Anzahl Ankerpunkte × Preis/Ankerpunkt)
```

- Ankerpunkte pro Art.-Nr. → Google Sheet
- Preis/Ankerpunkt je Wandart/Dämmung → Google Sheet (Wandart aus Schritt 4)
- Menge bleibt unverändert

**Beschreibung passt sich an:**
`inkl. Klebeanker an Wand (bis max. 10cm Dämmung)` *(je nach Wandart-Wahl)*

### 5C — Montagekosten

**Input für Kalkulation:**
- System-Typ
- Wellenlänge
- Anzahl Wandbefestigungen (gezählt aus SoSeKu-JSON)
- Anzahl Schraubfundamente / Fundamentpunkte (gezählt aus SoSeKu-JSON)
- Montagezone + Wandart + Dachterrasse (aus Schritt 4)
- Montage-Bausteine → Google Sheet

**Positionen in Sektion "Montage und Fracht":**

1. `System Installation - Soliday [System]`
   - Beschreibung: `Professionelle Ausführung der kompletten Systemmontage durch 2 Soliday Fachmonteure`
   - Preis: aus Bausteinen (System, Wellenlänge, Fundamente, Wandbefestigungen)

2. `Service & Logistik (Zone X)`
   - Beschreibung: `2 Termine (Grund- & Endmontage) inkl. Rüstzeiten & Maschinenpauschale`
   - Preis: aus Bausteinen (Zone, ggf. Dachterrasse)

3. `Dachterrassen-Aufschlag` *(nur wenn in Schritt 4 angehakt)*

4. `Transportzuschlag` *(aus Soliday-PDF, falls vorhanden)*

5. `Vorfrachtpauschale` *(wenn Masten im Angebot)* **oder** `Versandkosten` *(wenn keine Masten)*
   - Preis aus Google Sheet

### 5D — Finales SoSeKu-JSON

Alle Ergebnisse aus 5A–5C werden ins SoSeKu-JSON eingetragen.
Fracht-Preise aus Google Sheet werden eingetragen.
Finales JSON ist vollständig — alle Positionen, alle Preise.

**Anzeige:**
- Finales SoSeKu-JSON visuell aufbereitet
- Gesamtsumme sichtbar zur Prüfung

---

## Schritt 6 — sevdesk-Konvertierung (JavaScript)

Das finale SoSeKu-JSON wird in JavaScript deterministisch in die sevdesk API-Struktur umgewandelt. Kein KI-Schritt.

**Endpoint:** `POST /Order/Factory/saveOrder`

- Sektionen als Trennpositionen (`quantity: 0, price: "0"`)
- Teuerungszuschlag als `discountSave` (`discount: false, percentage: false`)
- Import-Kunde wird gesucht (Name/Kundennummer enthält "import")
- Angebotsnummer wird von sevdesk geholt
- Header/Footer-Texte je nach System

---

## Schritt 7 — sevdesk Import

- Angebot wird per API angelegt
- Direkter Link zum importierten Angebot:
  `https://my.sevdesk.de/#/ar/order/detail/id/{id}`

---

## Google Sheet

Zwei Prompts und alle Preis-/Konfigurationsdaten liegen im Google Sheet — öffentlich lesbar, per einfachem Fetch abrufbar (kein Auth).

| Sheet-Tab | Inhalt |
|-----------|--------|
| Extraktions-Prompt | System-Prompt für Claude (Schritt 1) |
| Transformations-Prompt | Umstrukturierungsregeln (Schritt 3) |
| Stoffe | Verfügbare Stoffe pro System mit Preis/m² und Default |
| Ankerpunkte | Art.-Nr. → Anzahl Ankerpunkte |
| Ankerpreise | Preis/Ankerpunkt je Wandart/Dämmung |
| Montage-Bausteine | Preisbausteine für System Installation & Service & Logistik |
| Fracht | Vorfrachtpauschale / Versandkosten |

---

## API-Keys

- **Anthropic API-Key** (erforderlich) — für Claude PDF-Extraktion
- **sevdesk API-Key** (optional, nur für Schritt 7) — für Import

Beide Keys: nur `sessionStorage`, werden beim Tab-Schließen gelöscht, nie im Quellcode.

---

## Development Notes

- **Kein Build-System** — reines HTML/CSS/JS, direkt im Browser öffnen
- **Timestamp** — sichtbar im Header-Untertitel (`YYYY-MM-DD` Format); immer vor jedem Commit und Push aktualisieren
- **Haiku → Sonnet Fallback** — große PDFs die Haiku-Context überschreiten werden automatisch mit Sonnet wiederholt
- **Fehlerbehandlung** — alle API-Fehler zeigen vollständigen HTTP-Status + Response-Body

---

## Transformationsregeln (Kugelmann)

### Global

**Weglass-Positionen (immer entfernen):**
- Exzentersatz: `00E13000735`, `00E13000648`, `00E13000649`
- Drahtseilspannhilfe: `00E13002106`

**Gleiche Art.-Nr. mergen:** Mengen addieren, Gesamtpreise addieren.

**Preise:** VK Netto aus Soliday 1:1 übernehmen. Nie neu berechnen.

**Umbenennung:** Kugelmann-Namen verwenden. "Aluminium-Mast" → "Aluminium Rund-Mast", "Edelstahl-Mast" → "Edelstahl Rund-Mast".

### Sektion 1: Segeltuch

**Reihenfolge:**
1. Segeltuch - [Stoff] *(Preis aus Schritt 5A)*
2. Segelplatten* *(falls vorhanden)*
3. Winter-Schutzhülle
4. Schutzabdeckung für 2/1 Sonnensegelecken
5. Segeltuch Befestigungsset
6. Schnappschäkel Edelstahl *(nur wenn unter Soliday "Segelzubehör")*
7. Reinigungs- und Pflegeset *(immer hier, egal wo Soliday es listet)*

**Winter-Schutzhülle:**
- Art.-Nr.: `73004A001` (standard), `71004A011` oder `70004A007` (premium)
- Länge: `wellenlaenge_aufrollsystem_mm` → **immer aufrunden** (`Math.ceil(mm / 1000)`)
- Name: `Winter-Schutzhülle - [X]m / Austrosail - stone`
- Einheit: immer **Stk.**

**Pflegeset:**
- `00E13001325` → `Reinigungs- und Pflegeset C/CS`
- `00E13001321` → `Reinigungs- und Pflegeset`

### Sektion 2: Segelsystem

**Aufrollsystem-Benennung:**

| System | Name |
|--------|------|
| SOLIDAY-C | Soliday C Segelsystem - [X]m Welle |
| SOLIDAY-CS | Soliday CS Segelsystem - [X]m Welle |
| SOLIDAY-CS-DREIECK | Soliday CS Segelsystem - [X]m Welle - Dreieck |
| SOLIDAY-CS-TWIN | Soliday CS-Twin Segelsystem - [X]m Welle |
| SOLIDAY-ONE | Soliday ONE Segelsystem - [X]m Welle |
| SOLIDAY-M | Soliday M Segelsystem - [X]m Welle |

Wellenlänge aus Art.-Nr.-Suffix (z.B. `78V1ONE09` → 9m).

**Artikel-Mappings Segelsystem:**

| Art.-Nr. | Kugelmann Name |
|----------|----------------|
| 00E13002183 | Aufpreis für Rohrmotor mit Funkempfänger |
| 00E13000850 | Anschlussleitung Becker Motor 2m weiß |
| 00E13001921 | Funk-Handsender 1-Kanal - SWC541+ |
| 00E13001922 | Funk-Handsender 8-Kanal - SWC548+ |
| 71003A045 | Sonnen- Windsensor mit Solarzelle - Funk - SC861+ |
| 71003A032 | Sonnen- Windsensor mit Stromanschluss - SC81 |
| 00E13001919 | Sonnen- Windsensor mit Stromanschluss - SC811+ |
| 00E13002076 | SOLIDAY-ONE Bedienset für Aluminium-Mast |
| 00E13002077 | SOLIDAY-ONE Bedienset für Wandmontage |
| 00E13002063 | Bedienseil 8mm schwarz/orange |
| VPVSTEILEONE | Teile für Verspannungsset SOLIDAY-ONE |
| VPVSTEILE | Teile für Verspannungsset SOLIDAY-M |
| VPVS008 | Drahtseil Ø4mm mit Gabelterminal L=8m |
| VPVS009 | Drahtseil Ø4mm mit Gabelterminal L=9m |
| VPVS011 | Drahtseil Ø4mm mit Gabelterminal L=11m |
| VPVS013 | Drahtseil Ø4mm mit Gabelterminal L=13m |
| 00E13000059 | Seilstopp-Winsch mit Kurbel |
| 00E13000591 | Snap-Design Winsch inkl. 25cm Kurbel & Adapterplatte |
| 00E13000743 | Snap-Grundplatte mit Fallstoppklemme |

### Sektion 3: Masten

**Reihenfolge:** Kürzere Masten zuerst → Zuschnitt → Abdeckkappen → Versteifungsrohr → Höhenverstellung/Gleitschiene

**Regel:** Art.-Nr. und Name aus Soliday 1:1 — nicht anhand der tatsächlichen Länge in der Beschreibung remappen.

**Mast-Mappings:**

| Art.-Nr. | Kugelmann Name |
|----------|----------------|
| 00E13000335 | Droppole Mast l=3m - silber eloxiert |
| 00E13000338 | Droppole Mast l=3m - schwarz eloxiert |
| 00E13000347 | Droppole Mast l=4m - schwarz eloxiert |
| 00E13000348 | Droppole Mast l=5m - schwarz eloxiert |
| 00E13000336 | Droppole Mast l=4m - silber eloxiert |
| 00E13000337 | Droppole Mast l=5m - silber eloxiert |
| 00E13000235 | Aluminium Rund-Mast l=bis 3m - silber eloxiert |
| 00E13000238 | Aluminium Rund-Mast l=bis 6m - silber eloxiert |
| 00E13001650 | Aluminium Rund-Mast l=bis 3m - anthrazit |
| 00E13001652 | Aluminium Rund-Mast l=bis 6m - anthrazit |
| 00E13000135 | Edelstahl Rund-Mast l=3m - geschliffen |
| 00E13001766 | Edelstahl Rund-Mast l=bis 6m - V4A geschliffen |
| 00E13000131 | Versteifungsrohr für Aluminium Rund-Mast |
| 00E13000096 | Versteifungsrohr für Edelstahl Rund-Mast |
| 00E13001135 | Zuschnitt auf Maß - Droppole Mast |
| 00E13000133 | Biegung Aluminium-Mast ohne Versteifungsrohr |
| 00E13000217 | Abdeckkappe Design für Aluminium Rund-Mast |
| 00E13000216 | Abdeckkappe Design für Edelstahl Rund-Mast |
| 00E13001689 | Abdeckkappe Design für Aluminium Rund-Mast - anthrazit |

**Gleitschiene → Masten** (wenn "inkl. Montage" in Soliday-Beschreibung):

| Art.-Nr. | Kugelmann Name |
|----------|----------------|
| 00E13000052 | Höhenverstellung - Gleitschiene 1,5m |
| 00E13000027 | Höhenverstellung - Gleitschiene 1,5m |
| 00E13000204 | Höhenverstellung - Gleitschiene 1,5m - mit Seilbremse |
| 00E13000764 | Höhenverstellung - Gleitschiene 1,5m - Droppole |

**Gleitschiene → Wandbefestigungen** (wenn keine Mastmontage):

| Art.-Nr. | Kugelmann Name |
|----------|----------------|
| 00E13000603 | Gleitschiene 1,5m |

### Sektion 4: Fundamente

**Reihenfolge:** Schwerlast-Schraubfundament → Droppole Dorn → Bodenplatten → Bodenhülse → Base Cube → Abdeckblende

| Art.-Nr. | Kugelmann Name | Beschreibung |
|----------|----------------|--------------|
| 00E13000227 | Schwerlast-Schraubfundament | maschinell eingedreht · inkl. Erdschraube & Justier-Granulat · sofort belastbar - ohne Flurschäden |
| 00E13000324 | Droppole Dorn für Schraubfundament | |
| 00E13001870 | Bodenplatte 10° Droppole | |
| 00E13001872 | Bodenplatte 2° Droppole | |
| 00E13001782 | Bodenplatte für Edelstahl Rund-Mast | |
| 00E13001604 | Bodenplatte 10° für Aluminium Rund-Mast | |
| 00E13001606 | Bodenplatte 0° für Aluminium Rund-Mast | |
| 00E13000244 | Bodenhülse Aluminium Rund-Mast | für Betonfundament |
| 00E13000302 | Base Cube Grundgestell f. Mast Droppole 10° Neigung | |
| 00E13000304 | Base Cube Abdeckungen 5tlg. Holz "Esche" | |
| 00E13001600 | BaseCube Beton L-Form für DP, mit Bodenhülse und Dorn, H=450 | |
| 00E13001412 | Abdeckblende für Droppole-Mast 2-teilig | |

**Sonderregel Base Cube:** Wenn `00E13000302` vorhanden, immer danach einfügen:
```js
{ artikelnummer: "", name: "Beschwerung Base Cube", einheit: "pauschal", preis: 0 }
```

### Sektion 5: Wandbefestigungen

**Reihenfolge:** Wandplatten → Wandschellen → Gegenplatten → Design-Wandhalterungen → Sonstiges

**"edelstahl poliert" Regel:** Suffix ` - edelstahl poliert` wenn Soliday-Beschreibung "Design" enthält und Artikel in Standard- und Design-Variante existiert.

| Art.-Nr. | Kugelmann Name |
|----------|----------------|
| 00E13000709 | Edelstahl Wandbefestigung 150/150/6 |
| 00E13000711 | Edelstahl Wandbefestigung 50/270/8 - senkrecht |
| 00E13000046 | Edelstahl Wandbefestigung CS 150/156/6 |
| 00E13002080 | Edelstahl Wandbefestigung 150/280/6 mit 2 Laschen |
| 00E13001855 | Edelstahl Wandbefestigung 150/150/6 - edelstahl poliert |
| 00E13001857 | Edelstahl Wandbefestigung 50/270/8 - senkrecht - edelstahl poliert |
| 00E13001856 | Edelstahl Wandbefestigung 270/50/8 - waagrecht - edelstahl poliert |
| 00E13001910 | Edelstahl Wandplatte rund ø180mm h=149mm |
| 00E13000243 | Wandschelle für Aluminium Rund-Mast |
| 00E13000110 | Gegenplatte für Wandschelle Aluminium Rund-Mast |
| 00E13001940 | Design Wandhalterung für Aluminium Rund-Mast - 5cm |
| 00E13000248 | Design Wandhalterung für Edelstahl Rund-Mast - 5cm |
| 00E13000320 | Design Wandhalterung für Droppole - 5cm |
| 00E13000321 | Design Wandhalterung für Droppole - 10cm |
| 00E13000322 | Design Wandhalterung für Droppole - 20cm |
| 00E13000112 | Befestigungsplatte für Gleitschiene |
| 00E13001025 | Sonderkonsole *(Preis 0, manuelle Eingabe in sevdesk)* |

### Sektion 6: Zubehör

**Reihenfolge:**
1. Design-Adapter mit Lasche Droppole *(immer zuerst)*
2. Design-Kreuzadapter mit Bügel
3. Design-Kreuzadapter
4. Droppole Ring-Adapterplatten
5. Design-Mastschellen
6. Seilsicherung
7. Schwerlastrolle / Pulley / Professionelle Seilrolle
8. Dyneema Seil *(nach Seilrollen)*
9. Schnappschäkel *(nur unter Soliday "Hardware", nicht "Segelzubehör")*
10. Schäkel mit Schnellverschluss
11. Ringbolzen

**Dyneema Seil Regel:** Art.-Nr. `00E13000107` unter "Segelzubehör" → immer in **Zubehör** (nicht Segeltuch). Alle Längen mergen. Name: `Dyneema Seil 4mm` (ONE/M), `C/CS: Zusätzliches Pylonseil 4mm mit Dyneema-Kern` (C/CS).

| Art.-Nr. | Kugelmann Name |
|----------|----------------|
| 00E13001787 | Design-Adapter mit Lasche Droppole |
| 00E13001742 | Design-Kreuzadapter mit Bügel |
| 00E13001741 | Design-Kreuzadapter |
| 00E13000206 | Droppole Ring-Adapterplatte komplett |
| 00E13001937 | Droppole Ring-Adapterplatte komplett für Wellenbefestigung |
| 00E13001735 | Design-Mastschelle für Aluminium Rund-Mast |
| 00E13001734 | Design-Mastschelle für Edelstahl Rund-Mast |
| 00E13000843 | Seilsicherung komplett zum Verstauen des Seils |
| 00E13002149 | Schwerlastrolle PULLEY-X10 |
| 00E13001965 | Schwerlastrolle für Seile bis 8mm |
| 00E13000042 | Professionelle Seilrolle 8 für Seile bis 8mm |
| 00E13000107 | Dyneema Seil 4mm *(ONE/M)* |
| 00E13000417 | Schnappschäkel Edelstahl *(nur unter Hardware)* |
| 00E13000026 | Schäkel mit Schnellverschluss Edelstahl |
| 00E13000066 | Ringbolzenset M8x80 |
| 00E13000127 | Ringbolzen M8x100mm |

### Sektion 7: Montage und Fracht

Positionen werden in Schritt 3 angelegt (Preis = 0), Preise werden in Schritt 5C/5D befüllt.

**Reihenfolge (fix):**
1. System Installation - Soliday [System]
2. Service & Logistik (Zone X)
3. Dachterrassen-Aufschlag *(nur wenn angehakt)*
4. Transportzuschlag *(aus Soliday-PDF, falls vorhanden)*
5. Vorfrachtpauschale *(wenn Masten vorhanden)* oder Versandkosten *(wenn keine Masten)*

**Betreff-Texte je System:**

| System | Betreff |
|--------|---------|
| SOLIDAY-C | Angebot - SOLIDAY C |
| SOLIDAY-CS | Angebot - SOLIDAY CS |
| SOLIDAY-ONE | Angebot - SOLIDAY ONE |
| SOLIDAY-M | Angebot - SOLIDAY M |
| SOLIDAY-MA | Angebot - SOLIDAY MA |
| SOLIDAY-FLEX | Angebot - SOLIDAY FLEX |
| SOLIDAY-SANDY | Angebot - SOLIDAY SANDY |
| SONNENSEGEL-MAß | Angebot - SOLIDAY Fix Segel |
| SONNENSEGEL-MAß-SNAP | Angebot - SOLIDAY Fix Segel mit SNAP System |

---

## Datei-Referenz

| Datei | Beschreibung |
|-------|--------------|
| `index.html` | Aktuelles Tool — Single HTML File, via GitHub Pages |
| `soliday-sevdesk05-12-02.txt` | Original-Tool (direkt Soliday→sevdesk, ohne Kugelmann-Transformation) |
| `Anpassungsregeln_v03.txt` | Transformationsregeln-Dokument (teilweise, Session läuft weiter) |

---

## Bekannte Lücken / TODO

- Systeme noch nicht getestet: SOLIDAY-X, SOLIDAY-XS, SOLIDAY-MA, SOLIDAY-FLEX, SOLIDAY-SANDY, SONNENSEGEL-MASS, RAFF-M-DRAHTSEIL, RAFF-M-FUEHRUNGSSCHIENE
- Unbekannte Art.-Nr.: Soliday-Name 1:1 übernehmen, Sektion nach bestem Urteil zuweisen
- Google Sheet Tabs und Fetch-URL noch zu definieren
