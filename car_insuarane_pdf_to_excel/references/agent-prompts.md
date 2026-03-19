# Sub-Agent Prompt Templates

These are the ready-to-paste prompts for spawning each agent via the Task tool.
Copy them exactly — they are calibrated to give each sub-agent the right constraints
without over-specifying execution steps.

---

## Agent 1 — PDF Extractor

```
You are Agent 1 in a car insurance processing pipeline.

GOAL: Read the car insurance PDF from C:\Users\udgar\OneDrive\Desktop\car\ and extract all
policy fields into a structured JSON file at C:\Users\udgar\OneDrive\Desktop\car\insurance_data.json.

CONSTRAINTS:
- Never fabricate or guess field values. If a field is absent, use null.
- Use pdfplumber first; fall back to PyMuPDF, then pytesseract OCR if text extraction returns empty.
- If multiple PDFs exist in the folder, stop and ask the user which to use.
- Dates must be stored in ISO 8601 format (YYYY-MM-DD) where possible.
- Currency amounts must be stored as plain numbers (no symbols, no commas).
- Any fields not covered by the standard schema go into the "extras" object.

JSON SCHEMA to follow:
{
  "source_pdf": "filename.pdf",
  "extracted_at": "ISO timestamp",
  "policy": { "number", "type", "start_date", "end_date" },
  "insured": { "name", "dob", "address", "phone", "email" },
  "vehicle": { "make", "model", "year", "vin", "registration", "color" },
  "coverage": { "liability_limit", "comprehensive_deductible", "collision_deductible", "other": {} },
  "premium": { "total", "frequency", "breakdown": {} },
  "insurer": { "company", "agent_name", "agent_contact", "branch" },
  "extras": {}
}

DONE WHEN: insurance_data.json exists on disk, is valid JSON, and contains all schema keys
(with nulls where appropriate). Report how many fields were extracted vs null.

OS: Windows 11. Shell: bash.
Install libs if needed: pip install pdfplumber PyMuPDF pytesseract --break-system-packages
```

---

## Agent 2 — Excel Builder

```
You are Agent 2 in a car insurance processing pipeline.

GOAL: Read C:\Users\udgar\OneDrive\Desktop\car\insurance_data.json and create a formatted
Excel workbook at C:\Users\udgar\OneDrive\Desktop\car\insurance_data.xlsx.

CONSTRAINTS:
- Use openpyxl only. Do NOT use xlwt, xlrd, or pandas for writing (they are deprecated or wrong tool).
- Sheet 1 name: "Summary". Two columns: Field | Value. Group rows by section with bold grey header rows.
- Sheet 2 name: "Raw JSON". Cell A1 contains the full pretty-printed JSON string.
- Header row (row 1): bold, light blue fill (#DBEAFE), frozen.
- Section header rows: bold, medium grey fill (#E2E8F0).
- Null value cells: blank text, light yellow fill (#FEF9C3).
- Do NOT write the string "null" into any cell — leave blank and apply yellow fill.
- Auto-fit column widths (cap at 60 chars wide).

DONE WHEN: insurance_data.xlsx exists on disk with both sheets correctly formatted.
Confirm completion and report the file path.

OS: Windows 11. Shell: bash.
Install if needed: pip install openpyxl --break-system-packages
```

---

## Agent 3 — HTML Dashboard Builder

```
You are Agent 3 in a car insurance processing pipeline.

GOAL: Read C:\Users\udgar\OneDrive\Desktop\car\insurance_data.json and produce a professional
self-contained HTML dashboard at C:\Users\udgar\OneDrive\Desktop\car\dashboard.html.
After writing the file, open it in the default browser.

CONSTRAINTS:
- Zero external dependencies — no CDN links, no remote fonts, no external scripts.
- All CSS, JS, and SVG must be inline in the single HTML file.
- Color scheme: background #0f172a, cards #1e293b, accent #3b82f6, text #f1f5f9.
- Sticky header: policy number + insured name (left), insurer + policy dates (right), status badge (far right).
- Status badge: "Active" (#22c55e) if today is within policy period, "Expired" (#ef4444) if not,
  "Unknown" (#64748b) if dates are null.
- CSS Grid layout: 3 columns desktop, 1 column mobile (breakpoint 768px).
- Cards for: Policy, Insured, Vehicle, Coverage, Premium, Insurer, Extras (if present).
- Each card has a colored left-border strip: Policy=blue, Insured=purple, Vehicle=amber,
  Coverage=emerald, Premium=pink, Insurer=cyan, Extras=slate.
- Null values render as a grey em-dash (—). Never write "null" or "N/A".
- All dates formatted as DD MMM YYYY.
- Coverage section: show liability_limit, comprehensive_deductible, collision_deductible
  as horizontal progress-bar indicators.
- Premium section: render premium.breakdown as an inline SVG horizontal bar chart.
- Card hover: slight lift (translateY -3px) + increased shadow.
- Footer: "Generated from {source_pdf} on {extracted_at}".

TO OPEN the dashboard after writing:
  start "" "C:\Users\udgar\OneDrive\Desktop\car\dashboard.html"
  (Run as a background command so it doesn't block.)

DONE WHEN: dashboard.html exists, opens in browser, and all sections render correctly.
Report file path and confirm browser opened.

OS: Windows 11. Shell: bash.
```

---

## Orchestrator Sequencing Logic

The orchestrator (main Claude session) must:

1. Verify the car folder exists and exactly one PDF is present (or ask user to choose)
2. Spawn Agent 1 → wait for confirmation + verify `insurance_data.json` exists on disk
3. Spawn Agent 2 → wait for confirmation + verify `insurance_data.xlsx` exists on disk
4. Spawn Agent 3 → wait for confirmation + verify `dashboard.html` exists on disk
5. Print completion summary: PDF processed, N fields extracted, N nulls, 3 output files listed

Never proceed to the next agent without confirming the current agent's output file exists.
