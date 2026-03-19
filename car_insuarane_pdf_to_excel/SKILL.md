---
name: car-insurance-pipeline
description: >
  Orchestrates a 3-agent pipeline for car insurance or auto-repair document processing.
  Use this skill whenever the user mentions: processing a car insurance PDF, repair estimate,
  service invoice, extracting insurance data, creating an insurance Excel file, building an
  insurance dashboard, "car insurance pipeline", "process the insurance PDF", "insurance data
  entry", or "parse my policy/invoice/estimate". Also triggers when the user uploads or
  references a PDF file alongside insurance-related language (policy, coverage, premium,
  insurer, deductible, VIN, vehicle, insured, parts, labour, repair, estimate, proforma).
  Err on the side of triggering — partial matches count.
---

# Car Insurance Pipeline

## What This Skill Does

Transforms any car insurance PDF — proforma invoice, repair estimate, service bill, policy
document — into two structured outputs:

1. **`insurance_data.xlsx`** — a multi-sheet Excel workbook with every line item and summary
2. **`dashboard.html`** — a self-contained HTML dashboard with charts and key figures

Three agents run sequentially and communicate only through the filesystem:

- **Agent 1** reads the PDF → writes `insurance_data.json`
- **Agent 2** reads the JSON → writes `insurance_data.xlsx`
- **Agent 3** reads the JSON → writes `dashboard.html`

---

## Phase 1 — Setup

1. Ask the user for their PDF path. Resolve `WORK_DIR` as the parent directory of the PDF.
2. Confirm the file exists. If multiple PDFs are in the folder, ask the user to choose one.
3. Install required libraries if missing:
   ```bash
   pip install pdfplumber openpyxl --break-system-packages -q
   ```
4. Proceed to Agent 1.

---

## Phase 2 — Agent 1: PDF Extraction

Spawn a subagent with this task (adapt paths at runtime):

> **Task for Agent 1:**
>
> Extract all data from the PDF at `<PDF_PATH>` and save it as `<WORK_DIR>/insurance_data.json`.
> Follow the extraction rules below exactly. Do not fabricate any value — use `null` if a field
> is not present in the PDF.
>
> ### Extraction Rules
>
> **Step 1 — Install and import**
> ```python
> import pdfplumber, json, re, os
> ```
>
> **Step 2 — Cell cleaning helpers**
>
> Define these helpers before any extraction:
>
> ```python
> def clean_text(val):
>     """Join multi-line cell text into a single string."""
>     if val is None: return None
>     return ' '.join(str(val).split())
>
> def clean_number(val):
>     """
>     Handle numbers that pdfplumber splits across lines, e.g. '8.56\n3' -> 8.563.
>     Also strips commas, currency symbols, and whitespace.
>     """
>     if val is None: return None
>     s = ' '.join(str(val).split())           # collapse newlines
>     s = re.sub(r'[₹$€£,\s]', '', s)         # strip currency & commas
>     try: return float(s)
>     except: return None
>
> def clean_code(val):
>     """Strip injected whitespace from part/labour codes."""
>     if val is None: return None
>     return re.sub(r'\s+', '', str(val)).strip()
>
> def fix_merged_amount(description, raw_amount_str):
>     """
>     Recover trailing characters that pdfplumber merged into the amount field.
>     Example: description='WHEEL ALIGNMENT, INSPECTION_ADJUSTMEN', raw='T830.0'
>     Returns: ('WHEEL ALIGNMENT, INSPECTION_ADJUSTMENT', 830.0)
>     """
>     s = str(raw_amount_str).strip() if raw_amount_str else ''
>     m = re.match(r'^([A-Za-z_,./\s]*)(\d[\d.]*)$', s)
>     if m and m.group(1):
>         desc = (description or '') + m.group(1)
>         try: return clean_text(desc), float(m.group(2))
>         except: pass
>     try: return clean_text(description), float(s)
>     except: return clean_text(description), None
> ```
>
> **Step 3 — Dynamic column map**
>
> Never hardcode column indices. Build the map from each table's actual header row:
>
> ```python
> def build_col_map(header_row):
>     """
>     Map semantic column names to indices from whatever header row the PDF provides.
>     Works for any number of columns and any insurer's naming conventions.
>     """
>     col_map = {}
>     for i, cell in enumerate(header_row):
>         if cell is None: continue
>         h = ' '.join(str(cell).lower().split())
>         if any(k in h for k in ['s.no', 's no', 'sr no', 'sr.no', 'sno', '#']):
>             col_map.setdefault('sno', i)
>         if any(k in h for k in ['part no', 'part code', 'labour code', 'code', 'item no']):
>             col_map.setdefault('code', i)
>         if any(k in h for k in ['description', 'particulars', 'item', 'name']):
>             col_map.setdefault('description', i)
>         if any(k in h for k in ['hsn', 'sac']):
>             col_map.setdefault('hsn', i)
>         if 'qty' in h or 'quantity' in h:
>             col_map.setdefault('qty', i)
>         if any(k in h for k in ['unit price', 'rate', 'unit rate', 'mrp']):
>             col_map.setdefault('unit_price', i)
>         if any(k in h for k in ['discount']):
>             col_map.setdefault('discount', i)
>         if any(k in h for k in ['taxable', 'taxable value']):
>             col_map.setdefault('taxable', i)
>         # 'amount' = pre-tax line total (use this for summary matching first)
>         if any(k in h for k in ['part amt', 'labour amt', 'amount']):
>             col_map.setdefault('amount', i)
>         if 'cgst' in h:
>             col_map.setdefault('cgst', i)
>         if 'sgst' in h:
>             col_map.setdefault('sgst', i)
>         if 'igst' in h:
>             col_map.setdefault('igst', i)
>         if 'utgst' in h:
>             col_map.setdefault('utgst', i)
>         if 'tax' in h and 'taxable' not in h and 'cgst' not in h and 'sgst' not in h:
>             col_map.setdefault('tax', i)
>         # 'total' = tax-inclusive line total
>         if 'total' in h and 'taxable' not in h and 'grand' not in h:
>             col_map.setdefault('total', i)
>     return col_map
> ```
>
> **Step 4 — Detect section headers**
>
> Use text keywords to identify which section a table belongs to. Do NOT use page numbers.
>
> ```python
> def detect_section(page_text):
>     t = (page_text or '').lower()
>     if 'labour' in t or 'labor' in t: return 'labour'
>     if 'parts' in t or 'spare' in t or 'material' in t: return 'parts'
>     if 'sublet' in t: return 'sublet'
>     return None
>
> def is_header_row(row):
>     """A header row contains at least one recognizable column keyword."""
>     keywords = ['s.no', 'sno', 'sr', 'description', 'particulars', 'code',
>                 'qty', 'amount', 'total', 'labour', 'part', 'hsn', 'tax', 'rate']
>     text = ' '.join(str(c).lower() for c in row if c)
>     return any(k in text for k in keywords)
>
> def is_data_row(row):
>     """A data row has at least one cell that parses as a number (the line amount or qty)."""
>     for cell in row:
>         if cell and str(cell).strip():
>             cleaned = re.sub(r'[₹$€£,\s]', '', str(cell).strip())
>             try: float(cleaned); return True
>             except: pass
>     return False
> ```
>
> **Step 5 — Extract rows**
>
> ```python
> def extract_rows(table, col_map, section):
>     rows = []
>     for row in table:
>         if not is_data_row(row): continue
>         if len(row) <= max(col_map.values(), default=0): continue
>
>         raw_desc = row[col_map.get('description', -1)] if 'description' in col_map else None
>         raw_amt  = row[col_map.get('amount', -1)]      if 'amount'      in col_map else None
>
>         # Recover chars that pdfplumber merged into the amount field
>         desc, amount = fix_merged_amount(clean_text(raw_desc), raw_amt)
>
>         record = {
>             'sno':        clean_text(row[col_map['sno']])        if 'sno'        in col_map else None,
>             'code':       clean_code(row[col_map['code']])       if 'code'       in col_map else None,
>             'description': desc,
>             'hsn':        clean_code(row[col_map['hsn']])        if 'hsn'        in col_map else None,
>             'qty':        clean_number(row[col_map['qty']])      if 'qty'        in col_map else None,
>             'unit_price': clean_number(row[col_map['unit_price']]) if 'unit_price' in col_map else None,
>             'discount':   clean_number(row[col_map['discount']])  if 'discount'   in col_map else None,
>             'taxable':    clean_number(row[col_map['taxable']])   if 'taxable'    in col_map else None,
>             'amount':     amount,
>             'cgst':       clean_number(row[col_map['cgst']])     if 'cgst'       in col_map else None,
>             'sgst':       clean_number(row[col_map['sgst']])     if 'sgst'       in col_map else None,
>             'igst':       clean_number(row[col_map['igst']])     if 'igst'       in col_map else None,
>             'utgst':      clean_number(row[col_map['utgst']])    if 'utgst'      in col_map else None,
>             'tax':        clean_number(row[col_map['tax']])      if 'tax'        in col_map else None,
>             'total':      clean_number(row[col_map['total']])    if 'total'      in col_map else None,
>         }
>         # Drop completely empty rows (all None except sno)
>         non_null = [v for k, v in record.items() if k != 'sno' and v is not None]
>         if non_null:
>             rows.append(record)
>     return rows
> ```
>
> **Step 6 — Multi-page table extraction**
>
> Key insight: line-item tables always span multiple pages. The first page has a header row.
> Continuation pages do NOT — they start directly with data rows.
> Use column count to decide which `col_map` a continuation table belongs to.
>
> ```python
> parts_items, labour_items, sublet_items = [], [], []
> parts_col_map, labour_col_map, sublet_col_map = {}, {}, {}
> header_fields = {}   # flat policy fields from PDF text
>
> with pdfplumber.open(pdf_path) as pdf:
>     current_section = None
>
>     for page in pdf.pages:
>         page_text = page.extract_text() or ''
>
>         # Detect section from page text (keywords)
>         detected = detect_section(page_text)
>         if detected:
>             current_section = detected
>
>         # Parse header text for flat policy fields (invoice number, dates, VIN, etc.)
>         # Use regex to pull key: value pairs from the page text
>         for line in page_text.split('\n'):
>             # Examples: "Invoice No: BU04B0126PFI01163", "VIN: W1N167..."
>             m = re.match(r'^([A-Za-z0-9 ./,_-]{3,40})\s*[:\-]\s*(.+)$', line.strip())
>             if m:
>                 k = m.group(1).strip().lower()
>                 v = m.group(2).strip()
>                 header_fields[k] = v
>
>         # Extract tables on this page
>         for table in page.extract_tables():
>             if not table or len(table) < 2: continue
>
>             # Determine if first row is a header
>             first_row = table[0]
>             if is_header_row(first_row):
>                 col_map = build_col_map(first_row)
>                 data_rows = table[1:]
>                 # Update section col_map
>                 if current_section == 'parts':    parts_col_map  = col_map
>                 elif current_section == 'labour': labour_col_map = col_map
>                 elif current_section == 'sublet': sublet_col_map = col_map
>             else:
>                 # Continuation table — infer col_map from column count
>                 ncols = len([c for c in first_row if c])
>                 if parts_col_map and abs(ncols - len(parts_col_map)) <= 2:
>                     col_map = parts_col_map
>                 elif labour_col_map and abs(ncols - len(labour_col_map)) <= 2:
>                     col_map = labour_col_map
>                 else:
>                     col_map = parts_col_map or labour_col_map or {}
>                 data_rows = table
>
>             if not col_map: continue
>
>             rows = extract_rows(data_rows, col_map, current_section)
>             if current_section == 'parts':    parts_items.extend(rows)
>             elif current_section == 'labour': labour_items.extend(rows)
>             elif current_section == 'sublet': sublet_items.extend(rows)
> ```
>
> **Step 7 — Extract summary totals from PDF text**
>
> Look for summary lines using flexible regex. Try multiple label patterns:
>
> ```python
> def find_amount_in_text(text, *labels):
>     """Search page text for a labelled amount. Returns float or None."""
>     for label in labels:
>         pattern = re.escape(label) + r'\s*[:\-]?\s*([\d,]+\.?\d*)'
>         m = re.search(pattern, text, re.IGNORECASE)
>         if m:
>             try: return float(m.group(1).replace(',', ''))
>             except: pass
>     return None
>
> full_text = '\n'.join(page.extract_text() or '' for page in pdfplumber.open(pdf_path).pages)
>
> pdf_parts_total  = find_amount_in_text(full_text, 'Parts Amount', 'Parts Total', 'Total Parts')
> pdf_labour_total = find_amount_in_text(full_text, 'Labour Amount', 'Labour Total', 'Total Labour')
> pdf_grand_total  = find_amount_in_text(full_text,
>     'Gross Invoice Value', 'Grand Total', 'Total Amount', 'Net Payable', 'Total Payable')
> ```
>
> **Step 8 — Dual-convention validation**
>
> Some PDFs summarise using the pre-tax `amount` column; others use the tax-inclusive `total`
> column. Try both and accept whichever matches within ₹1.00. Stop if neither matches.
>
> ```python
> def detect_convention(items, pdf_summary):
>     """
>     Returns ('amount'|'total'|'unknown', computed_value).
>     Tolerance of 1.0 accounts for GST rounding across many line items.
>     """
>     if pdf_summary is None:
>         return 'unknown', None
>     s_amt   = sum(r.get('amount') or 0 for r in items)
>     s_total = sum(r.get('total')  or 0 for r in items)
>     if abs(s_amt   - pdf_summary) <= 1.0: return 'amount', s_amt
>     if abs(s_total - pdf_summary) <= 1.0: return 'total',  s_total
>     return 'unknown', s_amt  # log discrepancy but don't block
>
> parts_conv,  parts_computed  = detect_convention(parts_items,  pdf_parts_total)
> labour_conv, labour_computed = detect_convention(labour_items, pdf_labour_total)
>
> validation = {
>     'parts_convention':    parts_conv,
>     'parts_pdf_total':     pdf_parts_total,
>     'parts_computed':      round(parts_computed,  2) if parts_computed  else None,
>     'parts_match':         parts_conv  != 'unknown',
>     'labour_convention':   labour_conv,
>     'labour_pdf_total':    pdf_labour_total,
>     'labour_computed':     round(labour_computed, 2) if labour_computed else None,
>     'labour_match':        labour_conv != 'unknown',
>     'grand_total_pdf':     pdf_grand_total,
>     'warnings':            []
> }
>
> if not validation['parts_match'] and parts_items:
>     validation['warnings'].append(
>         f"Parts total mismatch: PDF={pdf_parts_total}, "
>         f"computed_amount={round(sum(r.get('amount') or 0 for r in parts_items),2)}, "
>         f"computed_total={round(sum(r.get('total') or 0 for r in parts_items),2)}"
>     )
> if not validation['labour_match'] and labour_items:
>     validation['warnings'].append(
>         f"Labour total mismatch: PDF={pdf_labour_total}, "
>         f"computed_amount={round(sum(r.get('amount') or 0 for r in labour_items),2)}, "
>         f"computed_total={round(sum(r.get('total') or 0 for r in labour_items),2)}"
>     )
> ```
>
> **Step 9 — Map header fields to canonical JSON keys**
>
> Use flexible matching against `header_fields` dict (populated in Step 6):
>
> ```python
> def get_field(d, *keys):
>     for k in keys:
>         for dk in d:
>             if k.lower() in dk.lower():
>                 return d[dk]
>     return None
>
> meta = {
>     'invoice_number':    get_field(header_fields, 'invoice no', 'invoice number', 'bill no'),
>     'policy_number':     get_field(header_fields, 'policy no', 'policy number'),
>     'certificate_number':get_field(header_fields, 'certificate no', 'certificate number'),
>     'insurer_name':      get_field(header_fields, 'insurer', 'insurance company', 'insured by'),
>     'insured_name':      get_field(header_fields, 'customer name', 'bill to', 'insured name'),
>     'insured_address':   get_field(header_fields, 'address', 'delivery address'),
>     'insured_contact':   get_field(header_fields, 'mobile', 'phone', 'contact'),
>     'insured_email':     get_field(header_fields, 'email'),
>     'vehicle_registration': get_field(header_fields, 'reg no', 'registration', 'vehicle no'),
>     'vehicle_vin':       get_field(header_fields, 'vin', 'chassis'),
>     'vehicle_engine_number': get_field(header_fields, 'engine no', 'engine number'),
>     'vehicle_model':     get_field(header_fields, 'model', 'variant'),
>     'vehicle_make':      get_field(header_fields, 'make', 'brand'),
>     'vehicle_year':      get_field(header_fields, 'year', 'mfg year', 'model year'),
>     'vehicle_color':     get_field(header_fields, 'colour', 'color'),
>     'vehicle_fuel_type': get_field(header_fields, 'fuel type', 'fuel'),
>     'issue_date':        get_field(header_fields, 'date', 'invoice date', 'bill date'),
>     'coverage_start_date': get_field(header_fields, 'policy start', 'from date', 'start date'),
>     'coverage_end_date': get_field(header_fields, 'policy end', 'to date', 'end date'),
>     'net_premium':       get_field(header_fields, 'net premium', 'net payable'),
>     'total_premium':     get_field(header_fields, 'gross', 'total premium', 'grand total'),
>     'taxes_and_fees':    get_field(header_fields, 'tax amount', 'gst amount', 'total tax'),
>     'payment_mode':      get_field(header_fields, 'payment mode', 'mode of payment'),
>     'agent_name':        get_field(header_fields, 'consultant', 'advisor', 'agent name'),
>     'seller_name':       get_field(header_fields, 'dealer', 'workshop', 'garage', 'seller'),
>     'seller_address':    get_field(header_fields, 'dealer address', 'workshop address'),
>     'mileage':           get_field(header_fields, 'mileage', 'odometer', 'km'),
> }
> ```
>
> **Step 10 — Write JSON**
>
> ```python
> output = {
>     **meta,
>     'parts_items':   parts_items,
>     'labour_items':  labour_items,
>     'sublet_items':  sublet_items,
>     'col_maps': {
>         'parts':  {k: v for k, v in parts_col_map.items()},
>         'labour': {k: v for k, v in labour_col_map.items()},
>     },
>     'pdf_totals': {
>         'parts':       pdf_parts_total,
>         'labour':      pdf_labour_total,
>         'grand_total': pdf_grand_total,
>     },
>     'validation': validation,
> }
>
> json_path = os.path.join(work_dir, 'insurance_data.json')
> with open(json_path, 'w', encoding='utf-8') as f:
>     json.dump(output, f, indent=2, ensure_ascii=False)
>
> print(f"Extracted {len(parts_items)} parts rows, {len(labour_items)} labour rows.")
> print(f"Validation warnings: {validation['warnings']}")
> print(f"Saved: {json_path}")
> ```
>
> If `pdfplumber` returns empty text for any page, warn the user and try OCR:
> ```python
> # pip install pytesseract pillow --break-system-packages -q
> # Uses pdf2image to render pages then pytesseract to OCR them
> ```
> Do not silently produce an all-null JSON.

After Agent 1 completes, verify that `insurance_data.json` exists and is valid JSON before continuing.

---

## Phase 3 — Agent 2: Excel Builder

Spawn a subagent with this task:

> **Task for Agent 2:**
>
> Read `<WORK_DIR>/insurance_data.json` from disk and produce `<WORK_DIR>/insurance_data.xlsx`
> using openpyxl. Do not use xlwt, xlrd, or any deprecated library.
>
> ### Excel Rules
>
> ```python
> import json, openpyxl
> from openpyxl.styles import Font, PatternFill, Alignment
> from openpyxl.utils import get_column_letter
>
> YELLOW = PatternFill('solid', fgColor='FFFF99')
> BLUE   = PatternFill('solid', fgColor='D9E1F2')
> GREY   = PatternFill('solid', fgColor='F2F2F2')
> BOLD   = Font(bold=True)
> INR_FMT = '₹#,##0.00'
>
> def autofit(ws):
>     for col in ws.columns:
>         max_len = max((len(str(c.value or '')) for c in col), default=10)
>         ws.column_dimensions[get_column_letter(col[0].column)].width = min(max_len + 4, 60)
>
> def write_header(ws, headers):
>     for j, h in enumerate(headers, 1):
>         c = ws.cell(1, j, h)
>         c.font = BOLD
>         c.fill = BLUE
>     ws.freeze_panes = 'A2'
>
> def write_items(ws, items, headers):
>     """Write line items. Yellow-fill null cells. Apply INR format to numeric columns."""
>     for i, row in enumerate(items, 2):
>         for j, key in enumerate(headers, 1):
>             val = row.get(key)
>             cell = ws.cell(i, j, val)
>             if val is None:
>                 cell.fill = YELLOW
>             elif isinstance(val, float):
>                 cell.number_format = INR_FMT
>             if i % 2 == 0:
>                 cell.fill = GREY if val is not None else YELLOW
>
> with open(json_path) as f:
>     data = json.load(f)
>
> wb = openpyxl.Workbook()
> wb.remove(wb.active)  # remove default blank sheet
>
> # --- Summary Sheet ---
> ws = wb.create_sheet('Summary')
> rows = [(k, v) for k, v in data.items()
>         if k not in ('parts_items', 'labour_items', 'sublet_items', 'col_maps',
>                      'pdf_totals', 'validation') and not isinstance(v, (list, dict))]
> ws.append(['Field', 'Value'])
> ws['A1'].font = BOLD; ws['A1'].fill = BLUE
> ws['B1'].font = BOLD; ws['B1'].fill = BLUE
> for k, v in rows:
>     r = ws.append([k, v])
> # Highlight nulls yellow
> for row in ws.iter_rows(min_row=2):
>     if row[1].value is None:
>         row[1].fill = YELLOW
> autofit(ws)
>
> # --- Line Item Sheets (only if items exist) ---
> for section, key in [('Parts', 'parts_items'), ('Labour', 'labour_items'),
>                      ('Sublet', 'sublet_items')]:
>     items = data.get(key, [])
>     if not items: continue
>     ws = wb.create_sheet(section)
>     # Use actual keys present in first item as headers
>     headers = [k for k in items[0].keys() if any(r.get(k) is not None for r in items)]
>     write_header(ws, headers)
>     write_items(ws, items, headers)
>     autofit(ws)
>
> # --- Validation Sheet ---
> ws = wb.create_sheet('Validation')
> v = data.get('validation', {})
> ws.append(['Check', 'Result'])
> ws['A1'].font = BOLD; ws['A1'].fill = BLUE
> ws['B1'].font = BOLD; ws['B1'].fill = BLUE
> for k, val in v.items():
>     if k == 'warnings':
>         for w in (val or []):
>             ws.append(['WARNING', w])
>     else:
>         ws.append([k, str(val)])
> autofit(ws)
>
> # --- Raw JSON Sheet ---
> ws = wb.create_sheet('Raw JSON')
> ws.append(['insurance_data.json'])
> ws['A1'].font = BOLD
> ws.append([json.dumps(data, indent=2)])
> autofit(ws)
>
> xlsx_path = os.path.join(work_dir, 'insurance_data.xlsx')
> wb.save(xlsx_path)
> print(f"Saved: {xlsx_path}")
> ```

After Agent 2 completes, verify the `.xlsx` file exists before continuing.

---

## Phase 4 — Agent 3: HTML Dashboard

Spawn a subagent with this task:

> **Task for Agent 3:**
>
> Read `<WORK_DIR>/insurance_data.json` and produce `<WORK_DIR>/dashboard.html`.
> The HTML must be completely self-contained — no external CSS, no CDN scripts, no web fonts.
> Inline all styles. Use inline SVG for charts.
>
> ### Dashboard Requirements
>
> - **Header**: Document/invoice number, insurer name, vehicle registration, issue date
> - **Key Metrics bar**: Parts total, Labour total, Grand Total — rendered as highlighted cards
> - **Bar chart**: SVG bar chart comparing Parts vs Labour amounts
> - **Coverage/Policy info**: Any available policy or coverage fields
> - **Line items tables**: Parts table and Labour table (scrollable if long)
> - **Validation section**: Show any warnings from the `validation` block
> - **Null rendering**: Never write "null" or "N/A" — render missing values as a grey `—` em-dash
> - **Status badge**: Green "VALIDATED" if both parts_match and labour_match are true; amber
>   "CHECK REQUIRED" if any mismatch; rendered as a coloured pill badge in the header
>
> Use a clean, professional colour scheme (white background, dark navy headers, blue accents).
> The page must render correctly with no internet connection.
>
> After writing the file, open it non-blocking:
> ```python
> import subprocess, sys, os
> p = dashboard_path
> if sys.platform == 'win32':   subprocess.Popen(['cmd', '/c', 'start', '', p])
> elif sys.platform == 'darwin': subprocess.Popen(['open', p])
> else:                          subprocess.Popen(['xdg-open', p])
> print(f"Dashboard saved and opened: {p}")
> ```

---

## Phase 5 — Completion Report

Tell the user:
- Which PDF was processed
- How many parts rows and labour rows were extracted
- Whether totals matched (VALIDATED or warnings)
- Where the three output files are saved
- Any null fields in the metadata that the user may want to fill in manually

---

## Error Handling

| Situation | Action |
|---|---|
| Multiple PDFs in folder | Ask user which to process before starting |
| PDF is image-only / no text | Attempt OCR with pytesseract; warn about accuracy |
| `insurance_data.json` missing after Agent 1 | Stop pipeline; report error |
| Parts/Labour total mismatch > ₹1 | Log warning in validation block; continue but flag it |
| Excel creation fails | Report error; retry once |
| Dashboard HTML fails | Report error; retry once |
| Browser won't open | Provide file path for manual open |

---

## Critical Rules

**Never hardcode column indices.** Always use `build_col_map()` from the actual header row. PDFs from different insurers have different column orders and counts.

**Never hardcode page numbers.** Use keyword detection (`detect_section()`) and column count to identify which section a table belongs to.

**Never assume the summary convention.** Use `detect_convention()` to check whether the PDF's summary figures match the pre-tax `amount` column or the tax-inclusive `total` column. Accept within ₹1.00 tolerance.

**Never fabricate values.** If a field is absent, use `null`. Do not write empty strings, "N/A", or guesses.

**Never skip the JSON handoff.** Agent 2 and Agent 3 must read from disk — not from context.

**Never use the word "null" in the dashboard.** Render as `—`.

**Never make blocking subprocess calls** to open the browser. Use `subprocess.Popen`.
