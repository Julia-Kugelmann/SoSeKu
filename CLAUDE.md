# Soliday → Kugelmann → sevdesk · Project Briefing

## Project Overview

A web-based tool that converts Soliday solar sail offer PDFs into structured sevdesk quotes, applying Sonnensegel Kugelmann's internal naming and restructuring rules in the process.

**Flow:** Soliday PDF → Claude API (extraction) → Kugelmann transformation rules → Preview → sevdesk API (import)

**Current deliverable:** `index.html` — a single-file HTML/JS tool that runs entirely in the browser. No backend. No build step. Served via GitHub Pages.

---

## Architecture

```
index.html
├── CSS (embedded)
├── HTML views (setup | upload | loading | preview | sending | done | error)
└── JS (embedded)
    ├── EXTRACTION_PROMPT       — system prompt for Claude PDF extraction
    ├── WEGLASS_ARTNR/PATTERNS  — items always removed (Exzentersatz, Spannhilfe)
    ├── transformiere(data)     — core Kugelmann transformation logic
    ├── callClaude(base64, model) — Anthropic API call (Haiku → Sonnet fallback)
    ├── processFile(file)       — PDF → extract → transform → preview
    ├── renderPreview()         — renders transformed sections
    └── startTransfer()         — sevdesk API import
```

**Keys:** Anthropic API key (required) + sevdesk API key (optional, only for transfer). Both stored in `sessionStorage` only — never in source code.

---

## Extraction (Claude API)

Claude reads the PDF and returns a JSON object. Key fields:

```json
{
  "kommission": "string",
  "datum": "string",
  "system": "SOLIDAY-ONE",
  "snap": false,
  "wellenlänge_aufrollsystem_mm": 8559,
  "positionen": [
    {
      "pos": 1,
      "artikelnummer": "78936A",
      "name": "Soliday ONE-4-Eck Strahlenform...",
      "beschreibung": "...",
      "preis_einheit": 63.71,
      "menge": 38.34,
      "einheit": "m²",
      "gesamt": 2442.64,
      "soliday_sektion": "Segel"
    }
  ],
  "teuerungszuschlag_netto": 104.13,
  "netto_gesamt": 8516.54,
  "mwst": 1618.14,
  "brutto_gesamt": 10134.68
}
```

**Important:** `soliday_sektion` captures the original Soliday section name (Segel, Segelzubehör, Hardware, Punkt A, Punkt B, Punkt C, Punkt D, etc.) — used by transformation rules to determine where items go.

**Model:** `claude-haiku-4-5-20251001` → fallback `claude-sonnet-4-6` on context errors.

---

## Transformation Rules (`transformiere(data)`)

The function returns an array of sections, each with a `positionen` array:

```js
[
  { id: "segeltuch",         name: "Segeltuch",          positionen: [...] },
  { id: "segelsystem",       name: "Segelsystem",        positionen: [...] },
  { id: "masten",            name: "Masten",             positionen: [...] },
  { id: "fundamente",        name: "Fundamente",         positionen: [...] },
  { id: "wandbefestigungen", name: "Wandbefestigungen",  positionen: [...] },
  { id: "zubehoer",          name: "Zubehör",            positionen: [...] },
  { id: "montage",           name: "Montage und Fracht", positionen: [...] },
]
```

Empty sections are omitted. Section order is always fixed as above.

### Global Rules

**Always remove (Weglasspositionen):**

- All Exzentersatz variants: `00E13000735`, `00E13000648`, `00E13000649`
- Drahtseilspannhilfe: `00E13002106` (one-time purchase, not per-project)

**Merge identical Art.-Nr.:** Same article number → add quantities, add totals, show as one line. Exception: only merge when Art.-Nr. is truly identical (different Art.-Nr. = different positions even if similar name).

**Prices:** Always take VK Netto prices from Soliday 1:1. Never recalculate. Preis/Einheit × Menge must equal Gesamt exactly.

**Naming:** Always use Kugelmann names (see mappings below), not Soliday names. Text replacements: "Aluminium-Mast" → "Aluminium Rund-Mast", "Edelstahl-Mast" → "Edelstahl Rund-Mast".

---

### Section 1: Segeltuch

**Order:**

1. Segeltuch - Austrosail Nano *(main position, price from Google Sheet = 0)*
2. Segelplatten* *(if present)*
3. Winter-Schutzhülle
4. Schutzabdeckung für 2/1 Sonnensegelecken
5. Segeltuch Befestigungsset
6. Schnappschäkel Edelstahl *(only if under Soliday "Segelzubehör")*
7. Reinigungs- und Pflegeset *(always here, regardless of where Soliday lists it)*

**Segeltuch main position:**

- Name: `"Segeltuch - Austrosail Nano"` (default for C, CS, ONE)
- For M/MA system: `"Segeltuch - Austronet 960"` (default)
- Price: from Google Sheet (mark as `_googleSheet: true`, show "—")
- Description (shown italic below name):

  ```
  X.XXX,- € Austrosail Nano
  X.XXX,- € Austrosail Nano Sonderfarben
  X.XXX,- € Soltis 92
  X.XXX,- € Soltis 92 Sonderfarben
  Farbe nach Wahl aus Farbfächer des Herstellers
  ```

  Each line on its own line (not inline).

**Segelplatten (`00E13001416`):**

- Name: `"Segelplatten*"`
- Description: `"*nur beim Austrosail Nano Stoff möglich"`
- Merge all occurrences (appear once per sail corner)

**Winter-Schutzhülle:**

- Art.-Nr.: `73004A001` (standard), `71004A011` or `70004A007` (premium)
- Length: take `wellenlänge_aufrollsystem_mm` from extracted data → **always round UP** to next full meter (`Math.ceil(mm / 1000)`)
- Name: `"Winter-Schutzhülle - [X]m / Austrosail - stone"` or `"Winter-Schutzhülle Premium - [X]m / Austrosail - stone"`
- Unit: always **Stk.** (never m)
- Price: Soliday Gesamt price 1:1

**Pflegeset:**

- `00E13001325` → `"Reinigungs- und Pflegeset C/CS"`
- `00E13001321` → `"Reinigungs- und Pflegeset"`
- Always under Segeltuch, regardless of Soliday placement

---

### Section 2: Segelsystem

**Order (C/CS):** Aufrollsystem → Motor-Aufpreis → Anschlussleitung → Handsender → Windsensor → Windsensor-Zubehör

**Order (ONE):** Aufrollsystem → Bedienset → Winsch/Snap → Snap-Grundplatte → Bedienseil → Verspannungsset Teile → Drahtseil

**Order (M):** Aufrollsystem → Winsch → Snap-Grundplatte → Verspannungsset Teile → Drahtseil

**No bedienseil, no bedienset, no windsensor for M system.**

**Aufrollsystem naming:**

```
SOLIDAY-C:          "Soliday C Segelsystem - [X]m Welle"
SOLIDAY-CS:         "Soliday CS Segelsystem - [X]m Welle"
SOLIDAY-CS-DREIECK: "Soliday CS Segelsystem - [X]m Welle - Dreieck"
SOLIDAY-CS-TWIN:    "Soliday CS-Twin Segelsystem - [X]m Welle"
SOLIDAY-ONE:        "Soliday ONE Segelsystem - [X]m Welle"
SOLIDAY-M:          "Soliday M Segelsystem - [X]m Welle"
```

Wave length from Art.-Nr. suffix (e.g. `78V1ONE09` → 9m, `70001B011` → 11m).

**Key article mappings:**

| Art.-Nr.    | Kugelmann Name                                       |
|-------------|------------------------------------------------------|
| 00E13002183 | Aufpreis für Rohrmotor mit Funkempfänger             |
| 00E13000850 | Anschlussleitung Becker Motor 2m weiß                |
| 00E13001921 | Funk-Handsender 1-Kanal - SWC541+                    |
| 00E13001922 | Funk-Handsender 8-Kanal - SWC548+                    |
| 71003A045   | Sonnen- Windsensor mit Solarzelle - Funk - SC861+    |
| 71003A032   | Sonnen- Windsensor mit Stromanschluss - SC81         |
| 00E13001919 | Sonnen- Windsensor mit Stromanschluss - SC811+       |
| 00E13002076 | SOLIDAY-ONE Bedienset für Aluminium-Mast             |
| 00E13002077 | SOLIDAY-ONE Bedienset für Wandmontage                |
| 00E13002063 | Bedienseil 8mm schwarz/orange                        |
| VPVSTEILEONE| Teile für Verspannungsset SOLIDAY-ONE                |
| VPVSTEILE   | Teile für Verspannungsset SOLIDAY-M                  |
| VPVS008     | Drahtseil Ø4mm mit Gabelterminal L=8m                |
| VPVS009     | Drahtseil Ø4mm mit Gabelterminal L=9m                |
| VPVS011     | Drahtseil Ø4mm mit Gabelterminal L=11m               |
| VPVS013     | Drahtseil Ø4mm mit Gabelterminal L=13m               |
| 00E13000059 | Seilstopp-Winsch mit Kurbel                          |
| 00E13000591 | Snap-Design Winsch inkl. 25cm Kurbel & Adapterplatte |
| 00E13000743 | Snap-Grundplatte mit Fallstoppklemme                 |

---

### Section 3: Masten

**Order:** Shorter masts first → Zuschnitt → Abdeckkappen → Versteifungsrohr → Höhenverstellung/Gleitschiene

**Rule:** Article number and name from Soliday 1:1 — do NOT remap based on actual length in description. The first line of Soliday's article (name + Art.-Nr.) is what counts.

**Mast name mappings:**

| Art.-Nr.    | Kugelmann Name                                         |
|-------------|--------------------------------------------------------|
| 00E13000335 | Droppole Mast l=3m - silber eloxiert                   |
| 00E13000338 | Droppole Mast l=3m - schwarz eloxiert                  |
| 00E13000347 | Droppole Mast l=4m - schwarz eloxiert                  |
| 00E13000348 | Droppole Mast l=5m - schwarz eloxiert                  |
| 00E13000336 | Droppole Mast l=4m - silber eloxiert                   |
| 00E13000337 | Droppole Mast l=5m - silber eloxiert                   |
| 00E13000235 | Aluminium Rund-Mast l=bis 3m - silber eloxiert         |
| 00E13000238 | Aluminium Rund-Mast l=bis 6m - silber eloxiert         |
| 00E13001650 | Aluminium Rund-Mast l=bis 3m - anthrazit               |
| 00E13001652 | Aluminium Rund-Mast l=bis 6m - anthrazit               |
| 00E13000135 | Edelstahl Rund-Mast l=3m - geschliffen                 |
| 00E13001766 | Edelstahl Rund-Mast l=bis 6m - V4A geschliffen         |
| 00E13000131 | Versteifungsrohr für Aluminium Rund-Mast               |
| 00E13000096 | Versteifungsrohr für Edelstahl Rund-Mast               |
| 00E13001135 | Zuschnitt auf Maß - Droppole Mast                      |
| 00E13000133 | Biegung Aluminium-Mast ohne Versteifungsrohr           |
| 00E13000217 | Abdeckkappe Design für Aluminium Rund-Mast             |
| 00E13000216 | Abdeckkappe Design für Edelstahl Rund-Mast             |
| 00E13001689 | Abdeckkappe Design für Aluminium Rund-Mast - anthrazit |

**Gleitschiene → Masten section** (when "inkl. Montage" in Soliday description):

| Art.-Nr.    | Kugelmann Name                                        |
|-------------|-------------------------------------------------------|
| 00E13000052 | Höhenverstellung - Gleitschiene 1,5m                  |
| 00E13000027 | Höhenverstellung - Gleitschiene 1,5m                  |
| 00E13000204 | Höhenverstellung - Gleitschiene 1,5m - mit Seilbremse |
| 00E13000764 | Höhenverstellung - Gleitschiene 1,5m - Droppole       |

**Gleitschiene → Wandbefestigungen section** (when no Mastmontage):

| Art.-Nr.    | Kugelmann Name    |
|-------------|-------------------|
| 00E13000603 | Gleitschiene 1,5m |

---

### Section 4: Fundamente

**Order:** Schwerlast-Schraubfundament → Droppole Dorn → Bodenplatten → Bodenhülse → Base Cube → Abdeckblende

| Art.-Nr.    | Kugelmann Name                                               | Description                                                                                        |
|-------------|--------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 00E13000227 | Schwerlast-Schraubfundament                                  | maschinell eingedreht · inkl. Erdschraube & Justier-Granulat · sofort belastbar - ohne Flurschäden |
| 00E13000324 | Droppole Dorn für Schraubfundament                           |                                                                                                    |
| 00E13001870 | Bodenplatte 10° Droppole                                     |                                                                                                    |
| 00E13001872 | Bodenplatte 2° Droppole                                      |                                                                                                    |
| 00E13001782 | Bodenplatte für Edelstahl Rund-Mast                          |                                                                                                    |
| 00E13001604 | Bodenplatte 10° für Aluminium Rund-Mast                      |                                                                                                    |
| 00E13001606 | Bodenplatte 0° für Aluminium Rund-Mast                       |                                                                                                    |
| 00E13000244 | Bodenhülse Aluminium Rund-Mast                               | für Betonfundament                                                                                 |
| 00E13000302 | Base Cube Grundgestell f. Mast Droppole 10° Neigung          |                                                                                                    |
| 00E13000304 | Base Cube Abdeckungen 5tlg. Holz "Esche"                     |                                                                                                    |
| 00E13001600 | BaseCube Beton L-Form für DP, mit Bodenhülse und Dorn, H=450 |                                                                                                    |
| 00E13001412 | Abdeckblende für Droppole-Mast 2-teilig                      |                                                                                                    |

**Special rule — Base Cube:** When `00E13000302` is present, always insert after the Abdeckungen:

```js
{ artikelnummer: "", name: "Beschwerung Base Cube", einheit: "pauschal", preis: 0 }
```

---

### Section 5: Wandbefestigungen

**Order:** Wandplatten → Wandschellen → Gegenplatten → Design-Wandhalterungen → Sonstiges

**"edelstahl poliert" rule:** Add ` - edelstahl poliert` suffix when Soliday description contains "Design" AND the article exists in both standard and Design variants.

| Art.-Nr.    | Kugelmann Name                                                     |
|-------------|--------------------------------------------------------------------|
| 00E13000709 | Edelstahl Wandbefestigung 150/150/6                                |
| 00E13000711 | Edelstahl Wandbefestigung 50/270/8 - senkrecht                     |
| 00E13000046 | Edelstahl Wandbefestigung CS 150/156/6                             |
| 00E13002080 | Edelstahl Wandbefestigung 150/280/6 mit 2 Laschen                  |
| 00E13001855 | Edelstahl Wandbefestigung 150/150/6 - edelstahl poliert            |
| 00E13001857 | Edelstahl Wandbefestigung 50/270/8 - senkrecht - edelstahl poliert |
| 00E13001856 | Edelstahl Wandbefestigung 270/50/8 - waagrecht - edelstahl poliert |
| 00E13001910 | Edelstahl Wandplatte rund ø180mm h=149mm                           |
| 00E13000243 | Wandschelle für Aluminium Rund-Mast                                |
| 00E13000110 | Gegenplatte für Wandschelle Aluminium Rund-Mast                    |
| 00E13001940 | Design Wandhalterung für Aluminium Rund-Mast - 5cm                 |
| 00E13000248 | Design Wandhalterung für Edelstahl Rund-Mast - 5cm                 |
| 00E13000320 | Design Wandhalterung für Droppole - 5cm                            |
| 00E13000321 | Design Wandhalterung für Droppole - 10cm                           |
| 00E13000322 | Design Wandhalterung für Droppole - 20cm                           |
| 00E13000112 | Befestigungsplatte für Gleitschiene                                |
| 00E13001025 | Sonderkonsole *(price 0, manual entry in sevdesk)*                 |

---

### Section 6: Zubehör

**Order:**

1. Design-Adapter mit Lasche Droppole *(always first)*
2. Design-Kreuzadapter mit Bügel
3. Design-Kreuzadapter
4. Droppole Ring-Adapterplatten
5. Design-Mastschellen
6. Seilsicherung
7. Schwerlastrolle / Pulley / Professionelle Seilrolle
8. Dyneema Seil *(after Seilrollen)*
9. Schnappschäkel *(only if under Soliday "Hardware", not "Segelzubehör")*
10. Schäkel mit Schnellverschluss
11. Ringbolzen

**Dyneema Seil rule:** Art.-Nr. `00E13000107` when under Soliday "Segelzubehör" → always goes to **Zubehör** section (not Segeltuch). Merge all lengths. Name: `"Dyneema Seil 4mm"` for ONE/M, `"C/CS: Zusätzliches Pylonseil 4mm mit Dyneema-Kern"` for C/CS.

| Art.-Nr.    | Kugelmann Name                                             |
|-------------|------------------------------------------------------------|
| 00E13001787 | Design-Adapter mit Lasche Droppole                         |
| 00E13001742 | Design-Kreuzadapter mit Bügel                              |
| 00E13001741 | Design-Kreuzadapter                                        |
| 00E13000206 | Droppole Ring-Adapterplatte komplett                       |
| 00E13001937 | Droppole Ring-Adapterplatte komplett für Wellenbefestigung |
| 00E13001735 | Design-Mastschelle für Aluminium Rund-Mast                 |
| 00E13001734 | Design-Mastschelle für Edelstahl Rund-Mast                 |
| 00E13000843 | Seilsicherung komplett zum Verstauen des Seils             |
| 00E13002149 | Schwerlastrolle PULLEY-X10                                 |
| 00E13001965 | Schwerlastrolle für Seile bis 8mm                          |
| 00E13000042 | Professionelle Seilrolle 8 für Seile bis 8mm               |
| 00E13000107 | Dyneema Seil 4mm *(ONE/M)*                                 |
| 00E13000417 | Schnappschäkel Edelstahl *(only if under Hardware)*        |
| 00E13000026 | Schäkel mit Schnellverschluss Edelstahl                    |
| 00E13000066 | Ringbolzenset M8x80                                        |
| 00E13000127 | Ringbolzen M8x100mm                                        |

---

### Section 7: Montage und Fracht

**Order (fixed):**

1. System Installation *(price from Google Sheet = 0)*
2. Service & Logistik *(price from Google Sheet = 0)*
3. Dachterrassen Aufschlag *(if selected — future fragebogen step)*
4. Transportzuschlag *(if in Soliday PDF)*
5. Vorfrachtpauschale *(always last, price from Google Sheet = 0)*

**System Installation Art.-Nr. by system:**

| System      | Art.-Nr. | Name                              |
|-------------|----------|-----------------------------------|
| SOLIDAY-C   | INST-C   | System Installation - Soliday C   |
| SOLIDAY-CS  | INST-CS  | System Installation - Soliday CS  |
| SOLIDAY-ONE | INST-ONE | System Installation - Soliday ONE |
| SOLIDAY-M   | INST-M   | System Installation - Soliday M   |

**Descriptions:**

- System Installation: `"Professionelle Montage durch zwei Soliday-Fachmonteure"`
- Vorfrachtpauschale: `"Baustromgestellung bauseits: 230V - 16A\nElektrische Anschlüsse sind bauseits vorzunehmen"`

**Fracht articles from Soliday:**

| Art.-Nr. | Kugelmann Name                 |
|----------|--------------------------------|
| 99FRACHT5| Transportzuschlag bis 5m Länge |
| 99FRACHT6| Transportzuschlag ab 5m Länge  |

---

## sevdesk Import

**Endpoint:** `POST /Order/Factory/saveOrder`

**Section headers** are sent as positions with `quantity: 0, price: "0"` and the section name as `name`. sevdesk renders these as section dividers (Artikelnummer = 0 equivalent).

**Teuerungszuschlag** is sent as `discountSave` with `discount: false, percentage: false` (= fixed amount surcharge), not as a line item.

**Import customer:** The tool looks for a contact named or numbered "import" in sevdesk and uses that as the recipient for all imported offers.

---

## File Reference

| File                          | Description                                                         |
|-------------------------------|---------------------------------------------------------------------|
| `index.html`                  | Current tool — single HTML file, served via GitHub Pages            |
| `soliday-sevdesk05-12-02.txt` | Original tool (direct Soliday→sevdesk, no Kugelmann transformation) |
| `Anpassungsregeln_v03.txt`    | Transformation rules document (partial, session continues)          |

---

## Development Notes

- **No build system** — pure HTML/CSS/JS, open directly in browser or upload to server
- **API key storage** — `sessionStorage` only, cleared on tab close
- **Error handling** — all API errors show full HTTP status + response body in error view
- **Timestamp** — visible in header subtitle (`YYYY-MM-DD` format); always update before every commit and push
- **Haiku → Sonnet fallback** — large PDFs that exceed Haiku context automatically retry with Sonnet

---

## Known Gaps / TODO

- **Fragebogen step:** A form step before preview where user inputs: Montagezone, Dachterrassen-Aufschlag (ja/nein + Betrag), mounting specifics. This populates the Montage section prices.
- **Google Sheet prices:** Segeltuch prices, System Installation, Service & Logistik, Vorfrachtpauschale all come from a Google Sheet. Currently hardcoded as 0.
- **Segeltuch description by system:** Which fabric options appear in the description depends on the system (defined in Google Sheet). Currently shows C/CS/ONE options for all systems.
- **Systems not yet tested:** SOLIDAY-X, SOLIDAY-XS, SOLIDAY-MA, SOLIDAY-FLEX, SOLIDAY-SANDY, SONNENSEGEL-MASS, RAFF-M-DRAHTSEIL, RAFF-M-FUEHRUNGSSCHIENE.
- **Unknown articles:** Any Art.-Nr. not in the mappings → take Soliday name 1:1, assign section by best guess.
