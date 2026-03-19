---
name: car-insurance-pipeline
description: >
  Orchestrates a 3-agent pipeline for car insurance document processing. Use this skill whenever
  the user mentions: processing a car insurance PDF, extracting insurance data, creating an
  insurance Excel file, building an insurance dashboard, "car insurance pipeline", "process the
  insurance PDF", "insurance data entry", or "parse my policy". Also triggers when the user uploads
  or references a PDF file alongside insurance-related language (policy, coverage, premium,
  insurer, deductible, VIN, vehicle, insured). Err on the side of triggering — partial matches
  count. If the user says anything resembling "take this insurance PDF and do something useful
  with it", this skill applies.
---

# Car Insurance Pipeline

## Context

This skill drives a sequential 3-agent pipeline that transforms a raw car insurance PDF into two
structured deliverables: a formatted Excel workbook and a self-contained HTML dashboard. Agent 1
extracts all policy data from the PDF into a JSON handoff file. Agent 2 reads that JSON and builds
the Excel file. Agent 3 reads the same JSON and renders the interactive HTML dashboard. Agents
communicate exclusively via the filesystem — no direct agent-to-agent calls.

The pipeline exists because insurance PDFs are unstructured, insurer-specific, and error-prone to
process manually. This skill gives Claude precise constraints so it produces consistent, auditable
outputs regardless of PDF format variation.

---

## Goals

- [ ] Agent 1 extracts every meaningful field from the PDF into `insurance_data.json` — no fabrication, no omissions
- [ ] Agent 2 produces `insurance_data.xlsx` with a formatted Summary sheet and a Raw JSON sheet
- [ ] Agent 3 produces a fully self-contained `dashboard.html` with status badge, SVG chart, and responsive layout
- [ ] All three output files land in `C:\Users\udgar\OneDrive\Desktop\car\`
- [ ] `null` fields are preserved as `null` in JSON, highlighted yellow in Excel, and rendered as `—` in the dashboard
- [ ] The dashboard opens automatically in the default browser after Agent 3 completes
- [ ] Each agent confirms completion before the next is spawned

---

## Constraints

- **Sequential only** — never run agents in parallel; Agent 2 depends on Agent 1's output, Agent 3 depends on Agent 1's output
- **Never fabricate data** — if a field is absent from the PDF, set it to `null`; do not guess or infer values
- **Filesystem handoff only** — agents communicate via `insurance_data.json`; no in-memory passing between agents
- **Self-contained HTML** — the dashboard must have zero external dependencies (no CDN links, no remote fonts)
- **Car folder is fixed** — all files read from and written to `C:\Users\udgar\OneDrive\Desktop\car\` unless the user explicitly overrides the path
- **One PDF at a time** — if multiple PDFs exist in the folder, ask the user which to process before starting Agent 1
- **OS: Windows 11, Shell: bash**

---

## Gotchas

**Do NOT fabricate or infer missing fields.** If the PDF doesn't contain a VIN, policy number, or
premium amount, the value must be `null` in JSON — not an empty string, not "N/A", not a guess.
Fabricated data in insurance records causes downstream trust failures.

**Do NOT skip the JSON handoff step.** Agent 2 and Agent 3 must read from `insurance_data.json` on
disk — not from Claude's context window. If Agent 1 fails to write the file, stop the pipeline and
report the error before spawning Agent 2.

**Do NOT write external dependencies into the HTML dashboard.** No `<link>` tags to Google Fonts,
no `<script src="...cdn...">` tags. The dashboard must open correctly with no internet connection.
Inline all CSS and use inline SVG for the bar chart.

**Do NOT use the word "null" anywhere in the dashboard.** Null values must render as a grey `—`
em-dash character. Writing "null" or "N/A" or leaving a blank cell is a formatting failure.

**Do NOT run Agent 2 or Agent 3 before confirming the previous agent's output exists on disk.**
Always verify `insurance_data.json` exists and is valid JSON before spawning Agent 2. Skipping
this check causes cascading failures that are hard to diagnose.

**Do NOT produce rigid cell-by-cell Excel instructions.** Use the xlsx skill's openpyxl patterns.
Auto-fit column widths, bold section headers, and freeze row 1 — do not hard-code pixel widths or
rely on xlwt/xlrd (deprecated).

**Do NOT open the dashboard using a Python `subprocess` or `os.system` call if it's going to
block** — use `start "" "path"` in bash so the pipeline doesn't hang waiting for the browser
process.

**Do NOT assume the PDF is text-extractable.** Some insurance PDFs are scanned image files. If
`pdfplumber` / `PyMuPDF` returns empty text, warn the user and attempt OCR (pytesseract) before
giving up. Do not silently produce an all-null JSON file.

---

## Workflow Guidance

### Phase 1 — Setup & Validation
Before spawning any agents, confirm: the car folder exists, exactly one PDF is present (or ask the
user to choose), and the necessary Python libraries are available (`pdfplumber` or `PyMuPDF`,
`openpyxl`). Read `references/agent-prompts.md` for the exact sub-agent prompt templates to use
when spawning via the Task tool.

### Phase 2 — Agent 1 (PDF Extraction)
Spawn Agent 1. Its job is extraction and schema compliance. Read `references/json-schema.md` for
the canonical field list and JSON structure Agent 1 must produce. After Agent 1 reports completion,
verify `insurance_data.json` exists and parses as valid JSON before proceeding.

### Phase 3 — Agent 2 (Excel Builder)
Spawn Agent 2. Its job is layout and formatting. Read `references/excel-spec.md` for sheet
structure, formatting rules, and openpyxl patterns. After Agent 2 confirms, verify the `.xlsx`
file exists on disk.

### Phase 4 — Agent 3 (HTML Dashboard)
Spawn Agent 3. Its job is visual rendering. Read `references/dashboard-spec.md` for the full
layout spec, color palette, SVG bar chart pattern, and coverage indicator design. After writing the
file, Agent 3 opens it in the browser and confirms.

### Phase 5 — Completion Report
Tell the user: which PDF was processed, how many fields were extracted (non-null), how many were
null, and where the three output files are saved. If any fields are null, list them so the user
knows what to fill in manually.

---

## References (Read On Demand)

| File | Contents | When to read |
|---|---|---|
| `references/json-schema.md` | Full JSON schema with all fields, types, and null-handling rules | Before spawning Agent 1, or if extraction output looks wrong |
| `references/excel-spec.md` | Sheet layout, openpyxl formatting code patterns, null highlighting rules | Before spawning Agent 2 |
| `references/dashboard-spec.md` | Full HTML/CSS layout spec, color palette, SVG bar chart pattern, coverage indicator design, status badge logic | Before spawning Agent 3 |
| `references/agent-prompts.md` | Ready-to-paste sub-agent prompts for Agents 1, 2, and 3 | Before spawning any sub-agent via the Task tool |

---

## Error Handling Quick Reference

| Situation | Action |
|---|---|
| Multiple PDFs in folder | Ask user to specify before starting |
| PDF is image-only / no extractable text | Attempt OCR; warn user about accuracy |
| `insurance_data.json` missing after Agent 1 | Stop pipeline; report error |
| Excel creation fails | Report with details; retry once |
| Dashboard HTML write fails | Report with details; retry once |
| Browser won't open | Provide file path for manual open |
