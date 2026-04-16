# RISCO · FormSync

> **Match policy forms against PUMAA — spot gaps, extras & discrepancies instantly.**

A single-file, zero-dependency web tool for insurance operations. FormSync compares the endorsement forms on a PUMAA page against the forms listed in a policy document, classifies every form into a clear outcome category, and surfaces mismatches, extras, and version discrepancies for rapid review.

---

## Features

| Feature | Description |
|---|---|
| **PUMAA HTML Parser** | Drop the saved PUMAA Endorsements page; forms are extracted and split into System Generated and UW Choice lists automatically |
| **Dual Input Mode** | Paste form numbers manually or upload the policy PDF for auto-extraction |
| **Smart PDF Extraction** | Targets only the *Forms & Endorsements* block in the PDF; falls back to full text if the block is not found |
| **Four-Category Classification** | Every form lands in one of: Matched System, UW Choice, Not in PUMAA, or In PUMAA but Missing from Policy |
| **Year-Aware Fuzzy Matching** | Jaccard bigram similarity with year-proximity scoring detects near-matches across form versions (e.g. `RSC WE 01 2023` vs `RSC WE 01 2024`) |
| **Dual-Format Display** | When a form appears under different notation in PUMAA vs the policy, both codes are shown side-by-side with PUMAA / Policy tags |
| **Discrepancy Flagging** | Flag any row as a discrepancy; auto-suggests a note text for format mismatches; aggregates all flags in a dedicated section |
| **Accept Match** | One-click promotion of a fuzzy-matched form from "Not in PUMAA" to the correct matched category |
| **Side-by-Side View** | Toggle a two-panel view with PUMAA forms on the left and policy forms on the right, with colour-coded status dots |
| **Policy Info Block** | Supports multiple previous-policy entries (number, effective date, expiry date); included in all exports |
| **Export: Full Report** | Structured plain-text report with all four categories and flagged discrepancies |
| **Export: Formatted List** | Short grouped text — ready to paste into an email or notes |
| **Export: CSV** | Downloadable spreadsheet with category, PUMAA number, policy number, form name, and notes |
| **Print / PDF** | `@media print` stylesheet strips UI chrome and renders a clean page |
| **Live Search** | Filters all result rows simultaneously by form number or name |
| **Themes: DARK / LIGHT** | Default dark theme; toggle to light (LIGHT) mode via the header button |

---

## How to Use

### Step 1 — Prepare the PUMAA File

1. Open the **Endorsements** step in PUMAA.
2. Save the page: `Ctrl + S` → *Webpage, Complete*.
3. Drop or select the saved `.html` file in the **Step 1** card.

FormSync will parse the file and display how many System Generated and Underwriter Choice forms were found.

### Step 2 — Provide the Policy Forms

Choose one of two modes:

**Paste Manually** — Paste the form list from the policy. Any of the following formats are accepted:

```
HO 00 03 10 00    Homeowners 3 – Special Form
- (RSC WE 01 2023) War and Cyber War Exclusion
LMA 3201, RSC CP 09 2021
```

**Upload Policy PDF** — Drop the policy PDF. FormSync locates the *Forms & Endorsements* section and extracts form lines automatically. The extracted text appears in an editable textarea for correction before running the comparison.

### Step 3 — Compare & Review

Click **Compare Forms**. Results are displayed in four collapsible sections:

| Icon | Colour | Category |
|---|---|---|
| ✓ | Green | In Policy — System Generated |
| ⚠ | Amber | In Policy — UW Choice |
| ✕ | Red | In Policy — NOT Found in PUMAA |
| ○ | Teal | In PUMAA (System) — Not in Policy |

**For red (Not in PUMAA) rows:**
- Check the fuzzy-match suggestion if one appears.
- Click **✓ Accept Match** to promote to the correct category, or **⚠ Flag** to log as a discrepancy.

**For any row:**
- Type in the **Notes** field to annotate.
- Click **⚠ Flag** to send to the Discrepancy section.
- Click the form number to copy it to the clipboard.
- Click **✕** to remove the row from results and exports.

### Export

| Button | Output |
|---|---|
| 📋 Copy Full Report | Structured plain-text report |
| 📄 Copy Formatted List | Short grouped list for email / notes |
| ⬇ Download CSV | Spreadsheet with all data |
| 🖨 Print / PDF | Browser print dialog |

---

## Project Structure

FormSync is a **single self-contained HTML file**. There are no build steps, no dependencies to install, and no server required.

```
risco_formsync.html
│
├── <style>          — CSS custom properties (tokens), component styles,
│                      dark theme (:root), light theme (body.light),
│                      print stylesheet (@media print)
│
├── <body>           — App shell
│   ├── .hdr         — Header with logo, title, clock, theme toggle, version tag
│   ├── .stepper     — 3-node progress indicator
│   ├── .pinfo-card  — Policy info block (multiple previous-policy entries)
│   ├── #card1       — Step 1: PUMAA HTML upload zone
│   ├── #card2       — Step 2: Policy forms (manual textarea / PDF upload + tabs)
│   ├── .action-bar  — Compare Forms, Demo, Reset, Side-by-Side toggle
│   ├── #resultsArea — Stats row, results table, discrepancy section, export row
│   └── .ideas-panel — Collapsible roadmap panel
│
└── <script>         — All application logic (no external JS dependencies)
    ├── State         — pumaaData, notes, discrepancies, lastResult
    ├── PUMAA parser  — parseHTMLContent()
    ├── Policy parser — extractPolicyForms() (3-pass: regex / parens / line)
    ├── PDF extractor — handlePDF() via PDF.js
    ├── Comparison    — runComparison(), renderStats(), renderSections()
    ├── Fuzzy match   — findFuzzyMatch(), jacc(), extractYear(), stripYear()
    ├── SBS view      — renderSBS(), toggleSBS()
    ├── Discrepancy   — setupFlag(), refreshDiscrepancySection()
    ├── Exports       — buildReport(), doCopy(), doCSV(), copyFormattedList()
    ├── Theme toggle  — toggleTheme() + localStorage persistence
    └── Utilities     — norm(), escHtml(), showToast(), filterRows()
```

---

## External Dependencies

| Library | Version | Purpose |
|---|---|---|
| [PDF.js](https://mozilla.github.io/pdf.js/) | 3.11.174 | Client-side PDF text extraction |
| [Outfit](https://fonts.google.com/specimen/Outfit) | — | UI typeface |
| [JetBrains Mono](https://fonts.google.com/specimen/JetBrains+Mono) | — | Form numbers and monospace fields |

All other logic is vanilla JavaScript. No frameworks, no bundler.

---

## Themes

FormSync ships with two themes:

| Theme | Class | Description |
|---|---|---|
| **LIGHT** | `body.light` (default) | Clean light theme with white cards and indigo accents |
| **DARK** | `body.dark` | Deep navy background with frosted-glass cards |

The active theme is toggled via a button in the header and persisted to `localStorage` so it survives page refreshes.

---

## Parsing Notes

### PUMAA HTML Format

The parser targets `.card-body` elements and classifies them by their `.subtitle` text:
- Contains `"system generated"` → System Generated list
- Contains `"underwriter"` → UW Choice list

Each `.endrosment-list` row yields a form number (from `.container1`, with inputs/spans stripped) and a description (from `.col-form-label`).

### Policy Text Format

The three-pass extractor handles all common formats:

```
HO 00 03 10 00                   ← space-delimited
(RSC WE 01 2023) Description     ← parenthesised
- (HO 04 96 04 91) No Section II ← bulleted with parens
LMA 3201, RSC CP 09 2021         ← comma-separated
```

Forms containing at least one digit and matching the alphanumeric pattern are captured. Known PUMAA forms are matched first (preserving exact policy notation for dual-format display); unrecognised forms are placed in the *Not in PUMAA* category.

### PDF Block Detection

The extractor looks for a line matching `/FORMS?\s+AND\s+ENDORSEMENTS?/i` and captures subsequent lines until it encounters a known section-end header (e.g. `ADDITIONAL INTEREST`, `PREMIUM`, `DECLARATIONS`) or five consecutive non-form lines.

---

## Browser Requirements

Any modern browser with support for:
- `FileReader` API
- `DOMParser`
- `navigator.clipboard`
- ES2018+ (async/await, spread, optional chaining)

No installation, no server, no login. Open the file in a browser and use it.

---

## Roadmap

Items marked ✓ Built are already shipped. Planned features include:

- Smart Form Validation Engine (AI-assisted conflict detection)
- Policy Version Diff (renewal vs prior year endorsement diff)
- Required Forms Rule Engine (per-state mandatory form alerts)
- Batch Policy Processing (multiple PDFs vs one PUMAA page)
- One-Click Shareable Report (self-contained HTML snapshot)
- Comparison History & Audit Log (localStorage persistence)
- Form Description Normaliser (duplicate/conflict detection)
- RISCO Email Draft Generator (auto-generate agent follow-up email)

---

*RISCO · FormSync v1.0 — Built for insurance operations teams.*
