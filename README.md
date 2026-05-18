# RISCO · FormSync

**Single-file insurance endorsement comparison tool for RISCO underwriters.**  
Compares PUMAA endorsement selections against a carrier policy document — in the browser, with no server, no install, and no data leaving the machine.

---

## What It Does

When a policy comes back from the carrier, every endorsement form must be verified against what was selected in PUMAA. Doing this manually takes 20–40 minutes per policy and is error-prone. FormSync does it in under a minute.

It answers three questions at once:

- Which PUMAA-selected forms are confirmed in the policy?
- Which forms are in the policy but were never selected in PUMAA?
- Which PUMAA forms are missing from the policy entirely?

---

## How to Use It

**Step 1 — Save the PUMAA Endorsements page**

Use the built-in **📌 Save PUMAA Live Page** bookmarklet (drag it from the app to your bookmarks bar). On the PUMAA Endorsements screen, click the bookmark. It downloads an HTML file named `RISCO0085938_Edmund_Erin_Pacheco.html` with all checkbox states exactly as shown — ticked forms stay ticked.

> Do not use Ctrl+S. Angular updates checkboxes via JavaScript properties, not HTML attributes. Ctrl+S misses them all. The bookmarklet fixes this before saving.

**Step 2 — Upload the PUMAA file**

Drag and drop (or click) the downloaded HTML file into Step 1. The parser reads both the System Generated and Underwriter's Choice endorsement lists, filtering only checked forms. A badge shows how many forms were loaded and whether checkbox state was reliably captured.

**Step 3 — Provide the policy document**

Two options:

- **PDF upload** — Upload the carrier policy PDF. The tool extracts only the Forms & Endorsements block automatically using PDF.js. The extracted text is editable before running.
- **Manual paste** — Paste form numbers from the policy directly into the text area.

**Step 4 — Compare**

Click **Compare**. Results appear instantly across four sections.

---

## Result Sections

| Section | Colour | Meaning |
|---|---|---|
| **Confirmed in Policy** | Green | PUMAA-selected forms found in the policy. Sub-labelled ✓ System or ⚠ UW Choice per row. |
| **In Policy — Not in PUMAA** | Red | Forms in the carrier policy with no matching PUMAA entry. Review required. |
| **In PUMAA — Not in Policy** | Teal | PUMAA system forms absent from the policy. May need to be added. |
| **Available in PUMAA — Not Selected** | Purple | Forms in the policy that exist in PUMAA's UW Choice list but were not checked. Underwriter should review and select manually if required. |

---

## Auto-Detected Conflicts Panel

After comparison, the tool automatically detects two types of conflicts and surfaces them in a dedicated panel — forms in this panel are removed from their sections until resolved:

**Same form name, different number** (≥ 88% name similarity)  
Example: PUMAA has `LMA3200 — Sanctions Suspension Clause`, policy has `LMA 3201 — Sanctions Suspension Clause`. Both match by name but differ by number. Three actions per conflict:

- **✓ Accept as Match** — merges into Confirmed section with both PUMAA and Policy numbers shown
- **⚑ Add to Discrepancies** — moves to the Discrepancy panel with a pre-filled note
- **✕ Dismiss** — restores both forms to their original sections

**Same number, name differs** (< 45% name similarity)  
The policy document uses a different name for a form that number-matched. Flagged for UW review.

Each conflict card shows the correct source badge — `✓ System Generated`, `⚠ UW Choice`, `In Policy — Not in PUMAA`, or `◈ Available — Not Selected` — so the underwriter knows the weight of each side immediately.

---

## Form Description Normaliser

After every comparison, all policy forms are compared pairwise by name using Jaccard bigram similarity. Surfaces only forms that actually appeared in the policy — PUMAA-only forms are excluded.

| Score | Label |
|---|---|
| ≥ 90% | Possible Duplicate |
| 75–89% | Possible Conflict |
| 58–74% | Similar |

Collapsed panel, hidden when clean.

---

## Discrepancy Panel

Any form row can be flagged with ⚠ Flag. Flagged rows:

- Fade out of their section (pill count decrements)
- Appear in the Discrepancy panel with their note
- Can be returned with **↩ Return to section** — full undo, row restores exactly where it was

---

## Underwriting Notes

A notes card above Step 1 holds session-level observations. Eight standard templates are pre-loaded (Hydrant Distance, Alarms, Occupation, Date of Birth, etc.). Each template has ✎ Edit and ✕ Remove. New templates can be added. All notes appear at the top of every copy export.

**Template persistence:** Templates are shared across all users via JSONBin.io. Any team member can add or edit templates. Deleting a template requires the admin PIN. Templates survive page refresh for all users.

---

## Exports

| Button | Output |
|---|---|
| 📋 Copy Full Report | Structured text report: policy info, notes, all four sections, conflicts, discrepancies |
| 📄 Copy Formatted List | Clean bullet list for pasting into emails or task notes |
| 📊 Copy for Excel | Tab-separated rows — paste directly into Excel (Type / Form Number / Form Name / Comment) |
| ⬇ Download CSV | Saves `risco_[quote]_[date].csv` with all form data and notes |
| 🖨 Print / PDF | Print-optimised layout, all UI chrome hidden |

---

## Comparison Logic (Technical)

### PUMAA Parsing

`parseHTMLContent()` uses DOMParser on the saved PUMAA HTML. It detects whether the file was saved with the fixed bookmarklet by checking for `checked` attributes on checkboxes. If present, only checked forms are loaded. If absent (old Ctrl+S file), all forms are included as a fallback.

Unchecked forms from both System and UW Choice lists are stored separately in `sysUnchecked` / `uwUnchecked` and used to build the c5 category.

### Policy Extraction — Three Passes

`extractPolicyForms()` runs three sequential passes:

1. **PUMAA-first** — for every known PUMAA form, builds a flexible regex and searches the full policy text. Guards against false positives.
2. **Parenthesised entries** — captures `(FORM NUM) Description` patterns.
3. **Line-by-line heuristic** — splits lines on whitespace/commas, identifies form-number-like chunks.

Each form goes through `tryAdd()` which normalises, deduplicates, and assigns `src: 'sys' | 'uw' | 'unknown'`.

### Categorisation

```
c1 = policy forms with src === 'sys'           → Confirmed — System Generated
c2 = policy forms with src === 'uw'            → Confirmed — UW Choice
c3 = policy forms with src === 'unknown'       → In Policy, Not in PUMAA
c4 = PUMAA sys forms not in policy             → In PUMAA, Not in Policy
c5 = unchecked PUMAA forms that are in policy  → Available — Not Selected
```

c1 and c2 are merged at display time into a single "Confirmed in Policy" section with per-row origin badges. The separation is preserved internally for exports and flagging.

### Fuzzy Matching

`findFuzzyMatch()` for c3 forms — Jaccard bigram on form numbers, year-aware scoring (strips trailing 4-digit years, scores proximity ±0/1/2), name bonus up to +12%. Threshold: 58%. Already-matched PUMAA forms (c1/c2) are excluded from suggestions.

### Auto-Discrepancy Detection

`detectAutoDiscrepancies()` — name-match threshold 88% (full names, ≥ 5 chars each), name-mismatch threshold < 45%. Forms pulled out of sections immediately on detection. Deduplicates pairs via a `seenPair` Set.

---

## Theme

Type `LIGHT`, `DARK`, or `DIM` anywhere outside an input field to switch theme. A small HUD shows your keystrokes. Default is DARK.

---

## File Structure

```
index.html          Everything — HTML, CSS, JS in one self-contained file
README.md           This file
```

No build step. No dependencies to install. No data leaves the browser.

---

## JSONBin Setup (Shared Templates)

1. Create a free account at [jsonbin.io](https://jsonbin.io)
2. Create a new bin with `{"chips":[],"counter":9}` as initial content — set to **Private**
3. Copy your **Bin ID** and **Master Key** from the dashboard
4. In `index.html`, replace:
   ```js
   const JSONBIN_ID  = 'YOUR_BIN_ID_HERE';
   const JSONBIN_KEY = '$2a$10$YOUR_MASTER_KEY';
   const ADMIN_PIN   = '1234';   // change this before pushing
   ```

---

## For Future Development Sessions

Before editing, read the `RISCO_FormSync_HANDOFF.md` file. Key rules:

- Never change comparison logic without explicit instruction
- Always apply surgical `str_replace` edits — never rewrite sections wholesale
- Run a syntax check (`acorn` or equivalent) before delivering
- Verify the uploaded file is the latest before making changes
