# JSON Schema Reference — `insurance_data.json`

This file defines the canonical structure Agent 1 must produce. Every field listed here must appear
in the output JSON — set to `null` if not found in the PDF. Never omit a key, never invent a value.

---

## Full Schema

```json
{
  "source_pdf": "string — filename only, not full path",
  "extracted_at": "string — ISO 8601 timestamp, e.g. 2026-03-19T14:30:00Z",
  "policy": {
    "number": "string | null",
    "type": "string | null — e.g. 'Comprehensive', 'Third Party', 'Third Party Fire & Theft'",
    "start_date": "string | null — ISO date preferred, e.g. 2025-01-01",
    "end_date": "string | null — ISO date preferred"
  },
  "insured": {
    "name": "string | null",
    "dob": "string | null — ISO date if available",
    "address": "string | null — full address as single string",
    "phone": "string | null",
    "email": "string | null"
  },
  "vehicle": {
    "make": "string | null — e.g. 'Toyota'",
    "model": "string | null — e.g. 'Camry'",
    "year": "integer | null — 4-digit year",
    "vin": "string | null — Vehicle Identification Number",
    "registration": "string | null — licence plate / registration number",
    "color": "string | null"
  },
  "coverage": {
    "liability_limit": "number | null — in local currency units",
    "comprehensive_deductible": "number | null",
    "collision_deductible": "number | null",
    "other": {
      "key": "value — any additional named coverage types found in the PDF"
    }
  },
  "premium": {
    "total": "number | null — annual or total premium amount",
    "frequency": "string | null — e.g. 'Annual', 'Monthly', 'Semi-annual'",
    "breakdown": {
      "key": "number — individual coverage line items, e.g. 'Liability: 320.00'"
    }
  },
  "insurer": {
    "company": "string | null — insurance company name",
    "agent_name": "string | null",
    "agent_contact": "string | null — phone or email",
    "branch": "string | null — branch name or location"
  },
  "extras": {
    "key": "value — catch-all for any fields not covered above"
  }
}
```

---

## Field Handling Rules

### Null vs. Missing
- A field that is **present in the PDF but unreadable** → `null` with a note in `extras.extraction_notes`
- A field that is **genuinely absent** from the PDF → `null`
- A field with a **partial value** (e.g. truncated policy number) → include what was found; add note in `extras`

### Dates
- Prefer ISO 8601 format: `YYYY-MM-DD`
- If the PDF uses a regional format (e.g. `19/03/2026`), convert it
- If the year is ambiguous (2-digit), use context to resolve; if unresolvable, store as-found and note it

### Currency / Numbers
- Store as plain numbers (no currency symbols): `1250.00` not `"$1,250.00"`
- If currency is non-USD, add `"currency": "XYZ"` to the relevant section or to `extras`

### The `extras` Object
Use `extras` for:
- Fields present in the PDF that don't fit the schema (e.g. no-claims bonus, endorsements, exclusions)
- Extraction notes or warnings (key: `extraction_notes`)
- Any additional named drivers, additional vehicles, or riders

---

## Example Output (Partial)

```json
{
  "source_pdf": "policy_2025_XYZ.pdf",
  "extracted_at": "2026-03-19T14:30:00Z",
  "policy": {
    "number": "POL-2025-00123",
    "type": "Comprehensive",
    "start_date": "2025-03-01",
    "end_date": "2026-02-28"
  },
  "insured": {
    "name": "Jane Smith",
    "dob": "1988-07-14",
    "address": "42 Maple Street, Austin, TX 78701",
    "phone": "+1-512-555-0198",
    "email": null
  },
  "vehicle": {
    "make": "Honda",
    "model": "Civic",
    "year": 2021,
    "vin": "2HGFC2F59MH123456",
    "registration": "TX-ABC-1234",
    "color": "Silver"
  },
  "coverage": {
    "liability_limit": 100000,
    "comprehensive_deductible": 500,
    "collision_deductible": 750,
    "other": {
      "uninsured_motorist": 50000,
      "medical_payments": 5000
    }
  },
  "premium": {
    "total": 1840.00,
    "frequency": "Annual",
    "breakdown": {
      "liability": 420.00,
      "comprehensive": 310.00,
      "collision": 780.00,
      "medical_payments": 95.00,
      "uninsured_motorist": 235.00
    }
  },
  "insurer": {
    "company": "SafeRide Insurance Co.",
    "agent_name": "Mark Torres",
    "agent_contact": "+1-512-555-0100",
    "branch": "Austin Downtown"
  },
  "extras": {
    "no_claims_years": 3,
    "endorsements": ["Roadside Assistance", "Rental Reimbursement"],
    "extraction_notes": "Email field not present in this PDF template."
  }
}
```

---

## PDF Extraction Libraries

Preferred (in order):
1. `pdfplumber` — best for text-based PDFs with tables
2. `PyMuPDF` (fitz) — faster, good for complex layouts
3. `pytesseract` + `pdf2image` — fallback for scanned/image-only PDFs

Install check:
```bash
pip show pdfplumber PyMuPDF pytesseract --break-system-packages
```
