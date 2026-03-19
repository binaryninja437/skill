# Excel Specification — `insurance_data.xlsx`

Agent 2 must produce a two-sheet Excel workbook using `openpyxl`. This file defines the exact
layout, formatting rules, and code patterns to use.

---

## Sheet 1 — Summary

**Purpose:** Human-readable flat view of all extracted data, grouped by section.

### Layout

Two columns:
- Column A: `Field` (bold header)
- Column B: `Value` (bold header)

Row 1: Frozen header row with bold text and light blue fill (`#DBEAFE`).

Sections appear in this order, each preceded by a **bold section header row** with a medium grey
fill (`#E2E8F0`):

1. Policy
2. Insured
3. Vehicle
4. Coverage
5. Premium
6. Insurer
7. Extras (only if `extras` has non-null, non-notes keys)

### Within Each Section

Flatten all nested keys into `Field | Value` rows. Use dot notation for nested keys if needed
(e.g. `coverage.other.uninsured_motorist`). For dict values (like `premium.breakdown` or
`coverage.other`), create one row per key-value pair.

### Null Highlighting

Any cell in Column B whose value is `None` / `null`:
- Display text: leave empty (blank cell — do not write the string "null")
- Cell fill: light yellow (`#FEF9C3`)
- This makes missing data immediately visible

### Formatting Details

```python
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment
from openpyxl.utils import get_column_letter

# Header row
header_fill = PatternFill("solid", fgColor="DBEAFE")
header_font = Font(bold=True)

# Section header rows
section_fill = PatternFill("solid", fgColor="E2E8F0")
section_font = Font(bold=True)

# Null value cells
null_fill = PatternFill("solid", fgColor="FEF9C3")

# Auto-fit column widths
for col in ws.columns:
    max_length = max(len(str(cell.value or "")) for cell in col)
    ws.column_dimensions[get_column_letter(col[0].column)].width = min(max_length + 4, 60)

# Freeze top row
ws.freeze_panes = "A2"
```

---

## Sheet 2 — Raw JSON

**Purpose:** Preserve the source JSON for debugging and auditing.

- Name the sheet `Raw JSON`
- Cell A1: paste the entire JSON string (pretty-printed with `json.dumps(data, indent=2)`)
- Set A1's alignment to `wrap_text=True`
- Set column A width to 120, row 1 height to auto (or 400 as a safe default)
- No other formatting needed on this sheet

---

## Full openpyxl Snippet (Agent 2 Reference)

```python
import json
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment
from openpyxl.utils import get_column_letter

CAR_FOLDER = r"C:\Users\udgar\OneDrive\Desktop\car"
JSON_PATH = f"{CAR_FOLDER}\\insurance_data.json"
XLSX_PATH = f"{CAR_FOLDER}\\insurance_data.xlsx"

with open(JSON_PATH, "r", encoding="utf-8") as f:
    data = json.load(f)

wb = Workbook()
ws_summary = wb.active
ws_summary.title = "Summary"
ws_raw = wb.create_sheet("Raw JSON")

# --- Styles ---
HEADER_FILL = PatternFill("solid", fgColor="DBEAFE")
SECTION_FILL = PatternFill("solid", fgColor="E2E8F0")
NULL_FILL = PatternFill("solid", fgColor="FEF9C3")
BOLD = Font(bold=True)

def write_header(ws):
    ws.append(["Field", "Value"])
    for cell in ws[1]:
        cell.font = BOLD
        cell.fill = HEADER_FILL
    ws.freeze_panes = "A2"

def write_section(ws, label):
    ws.append([label])
    row = ws.max_row
    for col in [1, 2]:
        ws.cell(row=row, column=col).fill = SECTION_FILL
        ws.cell(row=row, column=col).font = BOLD

def write_row(ws, field, value):
    ws.append([field, value if value is not None else ""])
    if value is None:
        ws.cell(row=ws.max_row, column=2).fill = NULL_FILL

def flatten_and_write(ws, section_name, section_data, prefix=""):
    write_section(ws, section_name)
    for key, val in section_data.items():
        full_key = f"{prefix}{key}" if prefix else key
        if isinstance(val, dict):
            for sub_key, sub_val in val.items():
                write_row(ws, f"{full_key}.{sub_key}", sub_val)
        elif isinstance(val, list):
            write_row(ws, full_key, ", ".join(str(v) for v in val))
        else:
            write_row(ws, full_key, val)

write_header(ws_summary)
sections = ["policy", "insured", "vehicle", "coverage", "premium", "insurer"]
labels = ["Policy", "Insured", "Vehicle", "Coverage", "Premium", "Insurer"]
for section, label in zip(sections, labels):
    if section in data:
        flatten_and_write(ws_summary, label, data[section])

if data.get("extras"):
    extras_clean = {k: v for k, v in data["extras"].items() if k != "extraction_notes"}
    if extras_clean:
        flatten_and_write(ws_summary, "Extras", extras_clean)

# Auto-fit columns
for col in ws_summary.columns:
    max_len = max((len(str(cell.value or "")) for cell in col), default=10)
    ws_summary.column_dimensions[get_column_letter(col[0].column)].width = min(max_len + 4, 60)

# Raw JSON sheet
ws_raw["A1"] = json.dumps(data, indent=2)
ws_raw["A1"].alignment = Alignment(wrap_text=True)
ws_raw.column_dimensions["A"].width = 120

wb.save(XLSX_PATH)
print(f"Excel saved to {XLSX_PATH}")
```

---

## Common Mistakes to Avoid

- Do NOT use `xlwt` or `xlrd` — they are deprecated and don't support `.xlsx`
- Do NOT hard-code column widths as pixel values — use the auto-fit pattern above
- Do NOT write the string `"null"` into cells — leave them blank and apply null fill
- Do NOT merge cells for section headers — it breaks sorting and filtering
