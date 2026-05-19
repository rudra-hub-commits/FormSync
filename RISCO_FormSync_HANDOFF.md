# RISCO · FormSync — Developer Handoff

> Read this entire file before making any changes to `index.html`.
> This document is the source of truth for architecture, rules, and current state.

---

## Golden Rules

1. **Never change comparison logic without explicit instruction.** The five categories (c1–c5), fuzzy matching thresholds, and auto-discrepancy detection thresholds are calibrated for real underwriting use. Do not touch them speculatively.
2. **Surgical `str_replace` edits only.** Never rewrite whole sections or regenerate the file. One targeted replacement per logical change.
3. **Syntax-check before delivering.** Run `grep -c "function " index.html` — count must be ≥ 81. Visually verify no unclosed braces around edited regions.
4. **Single-file constraint is intentional.** No external JS files, no build step, no npm. Everything stays in `index.html`.
5. **Theme classes on `<body>`.** Active theme is one of `theme-light` (default), `theme-dark`, `theme-dim`. Never hardcode colours inline — always use CSS variables from `:root`.
6. **`prompt()` is banned.** Any PIN or password entry must use a custom modal with `type="password"`.
7. **Premises Liability is display-only.** Its answer must never appear in any copy export, CSV, print output, or clipboard text. Do not add it to `buildReport`, `copyFormattedList`, `copyForExcel`, or `doCSV`.

---

## File Structure

```
index.html                     Single self-contained file — HTML + CSS + JS
README.md                      User-facing how-to guide
RISCO_FormSync_HANDOFF.md      This file
```

---

## Architecture Overview

### State Variables

| Variable | Type | Purpose |
|---|---|---|
| `pumaaData` | `{sys[], uw[], sysUnchecked[], uwUnchecked[], quoteNum, insured}` | Parsed PUMAA endorsements |
| `lastResult` | `{c1[], c2[], c3[], c4[], c5[]}` | Current comparison result |
| `notes` | `{}` | Per-row notes keyed by `sid_index` |
| `discrepancies` | `{}` | Flagged rows keyed by `dk` string |
| `autoDiscs` | `[]` | Auto-detected conflicts array |
| `exportFilter` | `{notes, confirmed, c3, c4, c5, autoconflicts, discrepancies}` | Controls what appears in all copy exports |
| `chipTemplates` | `[]` | Shared UW note templates (persisted to JSONBin) |
| `htmlLoaded` / `policyLoaded` | `bool` | Gate for Compare button |

---

## Category System

```
c1  = policy forms matched to PUMAA sys checked   → Confirmed — System Generated
c2  = policy forms matched to PUMAA uw checked    → Confirmed — UW Choice
c3  = policy forms with src 'unknown'             → In Policy — NOT Found in PUMAA
c4  = PUMAA sys forms NOT in policy               → In PUMAA (System) — Not in Policy
c5  = unchecked PUMAA forms that ARE in policy    → Available in PUMAA — Not Selected
```

c1 and c2 are merged at render time into a single "Confirmed in Policy" section.
`SEC_CONFIG` array drives all section rendering — title, colour, icon, collapsed state, key name.

---

## Key Functions

| Function | What it does |
|---|---|
| `parseHTMLContent(html)` | Parses PUMAA HTML, extracts sys/uw checked + unchecked forms |
| `extractPolicyForms(raw)` | Three-pass extraction: PUMAA-first → parenthesised → line heuristic |
| `runComparison()` | Orchestrates everything: extract → categorise → detect → render |
| `detectAutoDiscrepancies(c1,c2,c3,c4)` | Name-match ≥88% and name-mismatch <45% detection |
| `renderSections(c1,c2,c3,c4,c5)` | Builds all result section DOM |
| `buildSection(cfg, items)` | Renders one section with rows, flags, notes, fuzzy hints |
| `setupFlag(row, item, cfg, dk, ...)` | Wires flag button — fades row, decrements pill, adds to discrepancies |
| `refreshDiscrepancySection()` | Re-renders discrepancy panel; wires drag-reorder on rows |
| `renderChips()` | Renders UW note template chips with drag handles |
| `wireChipDrag(wrap, container)` | HTML5 drag-reorder for template chips, persists to JSONBin |
| `wireNoteDrag(row)` | HTML5 drag-reorder for UW note rows |
| `wireDiscDrag(row)` | HTML5 drag-reorder for discrepancy panel rows |
| `removeChip(chipId)` | Shows custom password modal for PIN — never uses `prompt()` |
| `openExportModal()` | Opens export filter modal (also via `EXPORT` cheat code) |
| `buildReport()` | Builds full text report, respects `exportFilter` |
| `copyFormattedList()` | Builds formatted list, respects `exportFilter` |
| `copyForExcel()` | Builds TSV for Excel, respects `exportFilter` |
| `updatePremisesLiability()` | Reads `#plCoverage` + `#plResidence`, updates `#plBadge` — display only |
| `addPolicyEntry()` | Adds a policy number/date row to pinfo-card |
| `refreshPolicePreviews()` | Updates clickable policy preview pills |

---

## Premises Liability Indicator

**Location in HTML:** Inside `#policyInfoCard`, after `#piPreviews`, before card close tag.

**IDs:** `#plWidget`, `#plCoverage` (select), `#plResidence` (select), `#plBadge`, `#plNote`

### Logic table

| Coverage/Form | Residence Type | Result |
|---|---|---|
| DP3 | (any) | ✓ Always YES — indigo badge |
| HO3 / HO4 / HO5 / HO6 | Seasonal / Secondary | ✓ YES — green badge |
| HO3 / HO4 / HO5 / HO6 | Primary | ✕ NO — red badge |
| (nothing selected) | — | "Select above" — grey |

### Hard rules
- Widget has `no-print` class — hidden on print/PDF
- **Never** include in `buildReport`, `copyFormattedList`, `copyForExcel`, or `doCSV`
- `resetPolicyInfo()` resets both selects and calls `updatePremisesLiability()`

---

## Cheat Codes (type outside any input)

| Code | Action |
|---|---|
| `LIGHT` | Light theme |
| `DARK` | Dark theme |
| `DIM` | Dim theme |
| `EXPORT` | Open export section filter modal |

Add new codes to `ALL_CHEATS` object and handle in the keydown listener.

---

## Export Filter

Keys: `notes` · `confirmed` · `c3` · `c4` · `c5` · `autoconflicts` · `discrepancies`

All default `true`. Session-only. Applied in `buildReport`, `copyFormattedList`, `copyForExcel`.
Premises Liability is never in this filter — excluded at the source.

---

## Flagging Behaviour (uniform across all sections including c3)

**Flag:** row fades to 38% opacity → pill decrements → entry in `discrepancies{}` → `refreshDiscrepancySection()`

**↩ Return:** finds `.flag-btn.flagged[data-dk="..."]` and clicks programmatically → full reverse.

---

## Drag-to-Reorder

| Target | Drag src var | Persist |
|---|---|---|
| UW Note rows (`#unotesList`) | `noteDragSrc` | DOM only |
| Discrepancy rows (`#discRows`) | `discDragSrc` | DOM only |
| Template chips (`#unotesChips`) | `chipDragSrc` | `renderChips()` + JSONBin |

---

## JSONBin (Shared Templates)

Schema: `{ chips: [{id, label, text}], counter: number }`

- Read on startup via `loadChipsFromBin()`
- Written on every chip change via `saveChipsToBin()`
- Delete requires `ADMIN_PIN` — custom password modal, shake on wrong PIN
- Replace `JSONBIN_ID`, `JSONBIN_KEY`, `ADMIN_PIN` before pushing to Git

---

## PIN Modal

Created fresh per call, removed on close. Features:
- `type="password"` input — masked characters
- `@keyframes shake` + input clear on wrong PIN
- Enter submits · Escape closes · backdrop click closes

---

## Theme System

| Class | Description |
|---|---|
| `theme-light` (default) | Full light palette |
| `theme-dark` | Matches `:root` dark defaults |
| `theme-dim` | Softer dark, `--bg:#1a1f2e` |

---

## Known Constraints / Gotchas

- **`norm()`** — strips spaces/slashes/dots/dashes/parens, uppercases. Do not change.
- **`lastResult` mutation** — `detectAutoDiscrepancies()` splices forms out of sections into `autoDiscs`. Intentional.
- **`cfg.key === 'c1c2'`** — export filter maps this to `exportFilter.confirmed`.
- **PDF.js** — pinned to `3.11.174`. Do not upgrade without testing.
- **Note key format** — `sid_index` where sid = `s1`, `s3`, `s4`, `s5`.
- **`no-print`** — PL widget, export buttons, chips, action bar. Never remove from PL widget.
- **Bookmarklet** — downloads as `[PolicyNumber]_[InsuredName].html`. Reads Angular `.checked` JS property. Do not simplify.
