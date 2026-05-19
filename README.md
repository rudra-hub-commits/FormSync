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

Use the built-in **📌 Save PUMAA Live Page** bookmarklet (drag it from the app to your bookmarks bar). On the PUMAA Endorsements screen, click the bookmark. It downloads an HTML file named `[PolicyNumber]_[InsuredName].html` with all checkbox states exactly as shown — ticked forms stay ticked.

> Do not use Ctrl+S. Angular updates checkboxes via JavaScript properties, not HTML attributes. Ctrl+S misses them all. The bookmarklet fixes this before saving.

**Step 2 — Upload the PUMAA file**

Drag and drop (or click) the downloaded HTML file into Step 1. The parser reads both the System Generated and Underwriter's Choice endorsement lists, filtering only checked forms. A badge shows how many forms were loaded and whether checkbox state was reliably captured.

**Step 3 — Provide the policy document**

Two options:

- **PDF upload** — Upload the carrier policy PDF. The tool extracts only the Forms & Endorsements block automatically using PDF.js. The extracted text is editable before running.
- **Manual paste** — Paste form numbers from the policy directly into the text area.

**Step 4 — Compare**

Click **Compare Forms**. Results appear instantly across four sections.

---

## Result Sections

| Section | Colour | Meaning |
|---|---|---|
| **Confirmed in Policy** | Green | PUMAA-selected forms found in the policy. Sub-labelled ✓ System or ⚠ UW Choice per row. |
| **In Policy — Not in PUMAA** | Red | Forms in the carrier policy with no matching PUMAA entry. Review required. |
| **In PUMAA — Not in Policy** | Teal | PUMAA system forms absent from the policy. May need to be added. |
| **Available in PUMAA — Not Selected** | Purple | Forms in the policy that exist in PUMAA's UW Choice list but were not checked. |

---

## Premises Liability Indicator

Located in the policy info bar above the upload steps. Select the **Coverage/Form** and **Residence Type** to instantly see whether Premises Liability applies:

| Coverage / Form | Residence Type | Answer |
|---|---|---|
| DP3 | (any) | Always YES |
| HO3 / HO4 / HO5 / HO6 | Seasonal / Secondary | YES |
| HO3 / HO4 / HO5 / HO6 | Primary | NO |

> This indicator is **display-only** and is never included in any copy, export, or print output.

---

## Auto-Detected Conflicts Panel

After comparison, the tool automatically detects two types of conflicts:

**Same form name, different number** (≥ 88% name similarity)
Three actions per conflict: **✓ Accept as Match** · **⚑ Add to Discrepancies** · **✕ Dismiss**

**Same number, name differs** (< 45% name similarity)
Flagged for UW review — policy uses a different name for the same number.

---

## Form Description Normaliser

Pairwise name similarity across all policy forms using Jaccard bigram scoring.

| Score | Label |
|---|---|
| ≥ 90% | Possible Duplicate |
| 75–89% | Possible Conflict |
| 58–74% | Similar |

---

## Discrepancy Panel

Any row can be flagged with ⚠ Flag — works uniformly across **all sections** including "In Policy — Not Found in PUMAA":

- Row fades out of its section (pill decrements)
- Row appears in the Discrepancy panel with a note field
- **↩ Return to section** fully restores the row and re-increments the pill

---

## Drag-to-Reorder

Grab the **⠿** handle to reorder:

- **Template chips** — new order saved to JSONBin for all users
- **Underwriting Note rows** — reorder before copying
- **Discrepancy panel rows** — reorder flagged items

---

## Export Section Filter

Click **⚙ Export Filter** or type **`EXPORT`** outside any input to choose which sections appear in all copy exports:

| Section | Default |
|---|---|
| Underwriting Notes | ✓ On |
| Confirmed in Policy | ✓ On |
| In Policy — Not Found in PUMAA | ✓ On |
| In PUMAA (System) — Not in Policy | ✓ On |
| Available in PUMAA — Not Selected | ✓ On |
| Auto-Detected Conflicts | ✓ On |
| Flagged Discrepancies | ✓ On |

Settings apply to Full Report, Formatted List, and Excel copy. Session-only — reset on page reload.

---

## Underwriting Notes

Eight standard templates are pre-loaded (Hydrant Distance, Fire Station Distance, Alarms, Fuel Tank, Property Check, Occupation, Date of Birth, Mandatory Fields). Each has ✎ Edit, ✕ Remove, and ⠿ drag handle.

**Template persistence:** Shared across all users via JSONBin.io. Deleting requires the admin PIN — entered in a masked password field (wrong PIN shows a shake animation).

---

## Exports

| Button | Output |
|---|---|
| 📋 Copy Full Report | Structured text with all sections, notes, conflicts, discrepancies |
| 📄 Copy All (Formatted) | Clean bullet list for email or task notes |
| 📊 Copy for Excel | Tab-separated — paste directly into Excel |
| ⬇ Download CSV | `risco_[quote]_[date].csv` |
| 🖨 Print / PDF | Print-optimised layout |
| ⚙ Export Filter | Choose which sections appear in all copy exports |

---

## Keyboard Cheat Codes

Type outside any input field:

| Code | Action |
|---|---|
| `LIGHT` | Light theme |
| `DARK` | Dark theme |
| `DIM` | Dim / softer dark theme |
| `EXPORT` | Open export section filter |

A HUD shows your keystrokes as you type.

---

## File Structure

```
index.html                     Everything — HTML, CSS, JS, single file
README.md                      This file
RISCO_FormSync_HANDOFF.md      Developer reference
```

No build step. No dependencies to install. No data leaves the browser.

---

## JSONBin Setup (Shared Templates)

1. Create a free account at [jsonbin.io](https://jsonbin.io)
2. Create a new bin with `{"chips":[],"counter":9}` — set to **Private**
3. In `index.html`, replace:
   ```js
   const JSONBIN_ID  = 'YOUR_BIN_ID_HERE';
   const JSONBIN_KEY = '$2a$10$YOUR_MASTER_KEY';
   const ADMIN_PIN   = '1234';   // change before pushing
   ```

---

## For Future Development Sessions

Before editing, read `RISCO_FormSync_HANDOFF.md`. Key rules:

- Never change comparison logic without explicit instruction
- Always apply surgical `str_replace` edits — never rewrite sections wholesale
- `prompt()` is banned — use a custom modal with `type="password"` for any PIN entry
- Premises Liability must never appear in any copy, export, or print output
- Verify the uploaded file is the latest before making changes
