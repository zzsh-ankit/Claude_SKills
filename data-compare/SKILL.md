---
name: data-compare
description: >
  Use this skill whenever someone wants to compare, reconcile, diff, or validate
  two data files (CSV or Excel). Trigger on: "compare these files", "reconcile",
  "check differences", "Alteryx vs Fabric", "old format vs new format",
  "source vs target", "what's different between", or any prompt where two data
  files are mentioned alongside a desire to find mismatches or validate data.
  Handles both full comparisons (identical schemas) and partial comparisons
  (different schemas — matched columns only, column inventory for the rest).
  Always invoke this skill before writing any ad-hoc comparison code.
---

## Core principle — files first, code second

**Always read and inspect the actual files before writing any code.**
Never assume column names, primary keys, row counts, or data types.
The script you generate must be tailored to what is actually in the files —
not to a generic template.

---

## Phase 1 — Discover the files

Look for file references in the user's prompt in this order:

1. **Explicit names / paths** — user says "compare FileA and FileB" or pastes paths.
2. **Naming pattern** — scan the working directory for CSVs and Excel files;
   use naming conventions to pair them (e.g. `JPN_HP*` vs `MS_jp_hp*`,
   `*Alteryx*` vs `*Fabric*`, `*source*` vs `*target*`).
3. **Recency fallback** — if exactly two CSV/Excel files exist, use them.
4. **Ask if ambiguous** — if more than two candidates exist and no pattern is
   clear, list them and ask the user which two to compare. Keep this to one
   question.

Supported input formats: `.csv`, `.xlsx`, `.xls`.

---

## Phase 2 — Inspect the files (do this before writing any code)

Read the first few rows of each file and collect:

| What to check | Why it matters |
|---|---|
| Column names (exact) | Detect NBSP / whitespace quirks in headers |
| Row count | Flag if counts differ — may indicate missing data |
| Column count | Determines full vs partial comparison |
| Column overlap | How many columns are shared vs unique to each file |
| PK candidate | Test uniqueness of PRODUCT ID + PERIOD CODE (default); if not unique, test with additional columns or ask user |
| Sample values per key column | Spot date format differences, casing, placeholder styles upfront |

**Normalise headers during inspection:**
- Replace `U+00A0` (non-breaking space) → regular space
- Collapse multiple spaces → single space
- Strip leading/trailing whitespace

Report findings to the user briefly before proceeding:
```
Files found:
  File A: <name>  (<rows> rows, <cols> cols)
  File B: <name>  (<rows> rows, <cols> cols)

Columns: <n> matched | <n> only in A | <n> only in B
Primary key: PRODUCT ID + PERIOD CODE  →  unique on both sides ✓
Comparison type: FULL / PARTIAL
```

---

## Phase 3 — Determine comparison type

| Situation | Type | What gets compared |
|---|---|---|
| All columns in A exist in B and vice versa | **FULL** | All columns |
| Files have different schemas | **PARTIAL** | Matched columns only |

For PARTIAL comparisons, the Column Inventory sheet is mandatory —
it must list every column with its status: `IN BOTH — COMPARED`,
`ONLY IN FILE A`, or `ONLY IN FILE B`.

---

## Phase 4 — Generate the comparison script

Write a self-contained Python script tailored to these specific files.
Use `pandas`, `openpyxl`, `numpy`. Base the script on these rules:

### Primary key
Default: `["PRODUCT ID", "PERIOD CODE"]`
If the user specified different columns, use those instead.
Validate that the PK is unique on both sides before comparing; if not,
report duplicate keys and still compare (using first occurrence per key).

### Loading files
```python
# CSV
pd.read_csv(path, dtype=str, encoding="utf-8-sig", keep_default_na=False, low_memory=False)

# Excel
pd.read_excel(path, dtype=str, keep_default_na=False)
```
Always load everything as strings. Do numeric/date parsing yourself — never
let pandas infer types on load.

### Header normalisation (always apply)
```python
def normalise_columns(cols):
    return (pd.Series(list(cols))
              .str.replace(" ", " ", regex=False)
              .str.replace(r"\s+", " ", regex=True)
              .str.strip()
              .tolist())
```

### Column type classification
Classify each column as `numeric`, `date`, or `text` based on actual data:
- **numeric** — ≥ 80% of non-blank values parse with `pd.to_numeric`
- **date** — ≥ 80% of non-blank values parse with `pd.to_datetime` AND
  the raw values contain `/`, `-`, or `:` (guards against treating
  `202501` or `20250101` as dates)
- **text** — everything else

### Difference categorisation
Every differing cell gets exactly one category label:

| Category | Benign / Real | When to assign |
|---|---|---|
| `NUMERIC_PRECISION` | **Benign** | Both parse as numbers; equal within tolerance (`abs_tol=1e-6`, `rel_tol=1e-9`) |
| `DATE_FORMAT` | **Benign** | Both parse as dates; same date, different string format |
| `NUMERIC_DIFF` | **Real** | Both parse as numbers; differ beyond tolerance |
| `DATE_DIFF` | **Real** | Both parse as dates; different dates |
| `CASE_ONLY` | **Real** | Strings identical when uppercased |
| `BLANK_VS_BLANKLIKE` | **Real** | One side is `""`, other is a placeholder (`N/A`, `NULL`, `NOT APPLICABLE`, `NONE`, `NA`, `#N/A`, `-`, `.`) |
| `VALUE_VS_PLACEHOLDER` | **Real** | One side has a real value, other has a blank-like placeholder |
| `TEXT_DIFF` | **Real** | Plain text mismatch, none of the above |
| `MISSING_IN_B` | **Real** | Value in File A, blank in File B |
| `MISSING_IN_A` | **Real** | Value in File B, blank in File A |

Use the actual file names / labels for `MISSING_IN_*`
(e.g. `MISSING_IN_FABRIC`, `MISSING_IN_OLD_FORMAT`).

**Benign categories never contribute to the Real Mismatch count or FAIL verdict.**
Only `NUMERIC_PRECISION` and `DATE_FORMAT` are benign. Everything else is real.

### Numeric comparison
```python
NUM_ABS_TOL = 1e-6
NUM_REL_TOL = 1e-9

def numeric_close(a, f):
    both_nan = a.isna() & f.isna()
    diff = (a - f).abs()
    scale = pd.concat([a.abs(), f.abs()], axis=1).max(axis=1)
    tol = np.maximum(NUM_ABS_TOL, NUM_REL_TOL * scale.fillna(0))
    return both_nan | (diff <= tol)
```

### DATE vs PERIOD CODE sanity check
If both `DATE` and `PERIOD CODE` columns exist, verify that
`parsed_date.strftime("%Y%m") == PERIOD CODE` for ≥ 99% of rows.
Warn if not — it usually means `dayfirst` is wrong.

### Display blanks visibly
Replace `""` with `"(blank)"` in all output cells (detail sheets and
Value Pair Summary) so empty vs placeholder is readable in Excel.

---

## Phase 5 — Output structure

**Output file:** `<Label1>_vs_<Label2>_Comparison_<YYYYMMDD>.xlsx`
Place it in the same directory as the input files.
Auto-version if file exists (`_v2`, `_v3`, …).

**Sheets (in this order):**

| Sheet | Contents |
|---|---|
| Index | One row per sheet — name, description, row count |
| Run Info | File names, row/col counts, PK, tolerances, verdict, date check results |
| Summary | One row per compared column — Raw Mismatch, Benign, Real Mismatch, %, Status, Top Categories |
| Column Inventory | Every column across both files — status: IN BOTH / ONLY IN A / ONLY IN B |
| Measure Totals | SUM per numeric column, both sides, difference, % diff, status |
| Value Pair Summary | Distinct (ColA value, ColB value, Category) pairs + row count + 3 sample keys |
| Duplicate Keys | Any PK that appears more than once, with source label |
| Only in File A | Rows whose key exists only in File A |
| Only in File B | Rows whose key exists only in File B |
| `<Column name>` | One detail sheet per column with **real** mismatches — PK columns + File A value + File B value + Category (+ Difference cols for numeric) |

**Formatting (all sheets):**
- Freeze row 1
- Autofilter on header row
- Column widths auto-sized (min 10, max 55 chars)

**Verdict logic:**
`PASS` only if zero real-mismatch columns AND zero orphan keys.
Everything else is `FAIL`. Print verdict clearly at end of console output.

---

## Phase 6 — Report back to the user

After the script runs, summarise in this order:

1. **Verdict** — PASS or FAIL, one line
2. **Row / key integrity** — keys in both, orphans each side
3. **Real mismatches** — list each column, row count, %, top categories
4. **Format-only differences** — list columns, flag as no-action-needed
5. **Column inventory** — counts for matched / only-A / only-B
6. **One-line interpretation per real mismatch** — what the difference likely means
   (e.g. FX inversion, case normalisation gap, dimension lookup failure)

Do not paste the full Excel contents into chat — the file is the deliverable.

---

## Edge cases to handle gracefully

| Situation | Handling |
|---|---|
| File not found | Clear error naming which pattern/path failed; ask user to confirm path |
| Encoding not UTF-8 | Try `utf-8-sig`, then `latin-1`, then ask |
| PK not unique | Report duplicate count per side; compare using first occurrence; flag in Run Info |
| Zero common keys | Stop before column comparison; report key mismatch clearly |
| Column name has NBSP or double spaces | Normalise silently; note in Run Info if any were found |
| All rows only in one file | Still produce output with empty comparison sheets |
| Very large file (>500k rows) | Warn user that script may take a few minutes; proceed |
| Excel file with multiple sheets | Ask user which sheet to use before proceeding |
