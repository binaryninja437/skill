# car-insurance-pipeline

> A Claude Code skill that processes car insurance PDFs into structured Excel workbooks and interactive HTML dashboards via a sequential 3-agent pipeline.

---

## What It Does

This skill orchestrates three sequential sub-agents that transform a raw car insurance PDF into two structured deliverables. Agent 1 extracts all policy fields into a JSON handoff file using `pdfplumber` or OCR fallback. Agent 2 reads that JSON and builds a formatted Excel workbook with a Summary sheet and Raw JSON sheet. Agent 3 reads the same JSON and generates a self-contained HTML dashboard with live policy status, a premium bar chart, and coverage indicators — then opens it in the browser.

---

## Installation

### Option 1: Claude Code Skills Folder (Recommended)

```bash
# Clone the repo
git clone https://github.com/your-org/car-insurance-pipeline.git

# Copy the skill folder into your Claude skills directory
cp -r car-insurance-pipeline ~/.claude/skills/car-insurance-pipeline
```

### Option 2: Manual Copy

Download this folder and place it at:
```
~/.claude/skills/car-insurance-pipeline/
```

Claude Code will auto-detect the skill from the `SKILL.md` file.

### Option 3: Plugin Bundle

If distributing as a Cowork plugin, include the full `car-insurance-pipeline/` folder in your `.plugin` bundle's `skills/` directory.

---

## Usage

### Example Prompts That Trigger This Skill

```
"Process the car insurance PDF in my car folder"
```
```
"Extract data from my insurance PDF and build me a dashboard"
```
```
"Run the car insurance pipeline on the PDF on my Desktop"
```
```
"I have a policy PDF — can you get the data into Excel and show me a dashboard?"
```

The skill description is intentionally broad — it triggers on any combination of insurance + PDF + data processing language.

---

## Required Setup

Before running, ensure the following are available on the target machine:

**Python libraries:**
```bash
pip install pdfplumber PyMuPDF pytesseract openpyxl --break-system-packages
```

**Output folder** (default path — change in SKILL.md Constraints section if different):
```
C:\Users\
```

---

## Outputs

| File | Description |
|---|---|
| `insurance_data.json` | Structured extraction of all policy fields |
| `insurance_data.xlsx` | Formatted Excel workbook (Summary + Raw JSON sheets) |
| `dashboard.html` | Self-contained interactive HTML dashboard |

---

## Folder Structure

```
car-insurance-pipeline/
├── SKILL.md                          # Main skill file (orchestrator instructions)
├── README.md                         # This file
├── references/
│   ├── json-schema.md                # Full JSON schema + extraction rules
│   ├── excel-spec.md                 # Excel layout, formatting, openpyxl patterns
│   ├── dashboard-spec.md             # HTML/CSS/SVG dashboard specification
│   └── agent-prompts.md              # Ready-to-paste sub-agent prompt templates
├── assets/
│   └── (reserved for future template files)
└── scripts/
    └── (reserved for future helper scripts)
```

The skill uses **progressive disclosure** — `SKILL.md` stays lean and Claude reads reference files
only when needed for the relevant agent phase.

---

## Customizing the Output Path

The default car folder is `C:\Users\udgar\OneDrive\Desktop\car\`. To change this:

1. Open `SKILL.md`
2. Under **Constraints**, update: `Car folder is fixed — all files read from and written to <your path>`
3. Update the same path in all four `references/` files

---

## Contributing

### Adding Gotchas

Found a new failure mode? Add a `Do NOT...` statement to the **Gotchas** section in `SKILL.md`.
Good gotchas are specific, actionable, and based on observed failures — not hypothetical warnings.

### Adding PDF Format Support

If you've tested this skill against a new insurer's PDF format, add an example to
`references/json-schema.md` under "Example Output" and note any insurer-specific quirks.

### Improving the Dashboard

Dashboard layout changes belong in `references/dashboard-spec.md`. Keep the HTML skeleton in
the spec file and only update `SKILL.md` if the constraints change.

---

## License

MIT License — see `LICENSE` file (add your own).

---

*Built with [Claude Code](https://claude.ai/claude-code) · Skill architecture by Anthropic*
