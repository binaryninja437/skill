# consultation-scribe

> A Claude Code skill that converts raw doctor's notes or encounter transcripts into structured clinical notes (SOAP format) plus a patient-friendly summary — and opens the result in a browser instantly.

---

## What It Does

This skill runs a 3-agent pipeline that transforms unstructured consultation notes into a clean, structured clinical document. Agent 1 parses the source into labelled clinical buckets. Agent 2 writes the full structured note — Chief Complaint, HPI, Exam, Investigations, Assessment, Plan, and a plain-language Patient-Friendly Summary. Agent 3 opens the saved note in a markdown viewer for immediate review. Every missing field is explicitly marked `— not documented` rather than silently omitted or fabricated.

---

## Installation

### Option 1: Claude Code Skills Folder

```bash
git clone https://github.com/binaryninja437/skill.git
cp -r skill/consultation-scribe ~/.claude/skills/consultation-scribe
```

### Option 2: Manual Copy

Download this folder and place it at:
```
~/.claude/skills/consultation-scribe/
```

### Option 3: Cowork Plugin Bundle

Include the `consultation-scribe/` folder in your `.plugin` bundle's `skills/` directory.

---

## Optional: Markdown Viewer

For a fully rendered (styled HTML) view of the note, install the markdown viewer server.
Without it, the skill falls back to opening the `.md` file directly in your browser.

```bash
# Example — install in the standard skills location
mkdir -p ~/.claude/skills/markdown-novel-viewer/scripts
# Copy server.cjs to that location
```

If Node.js is not installed or the viewer script is not found, the skill automatically
falls back to opening the file directly in your browser.

---

## Usage

### Example Prompts That Trigger This Skill

```
"Write up my notes" + [paste raw consultation notes]
```
```
"Make a SOAP note from this:" + [encounter transcript]
```
```
"Format this consultation for the EMR" + [bullet-point notes]
```
```
"Give me a patient-friendly version of these notes"
```
```
"Document this case — specialty: cardiology" + [notes]
```

### Optional Modes

Append any of these to your request:

| Mode | Effect |
|---|---|
| `plain text` | Strips Markdown formatting — safe to paste into any EMR system |
| `brief` | Outputs Assessment + Plan + Patient Summary only |
| `specialty: [name]` | Adapts terminology for cardiology, paediatrics, orthopaedics, psychiatry, obstetrics, etc. |

---

## Output

The skill saves `consultation-note.md` in the same directory as your source file (or a
directory you specify), then opens it in the browser automatically.

---

## Clinical Safety Principles

This skill follows strict documentation discipline:

- **Never fabricates** clinical values, lab results, drug doses, or examination findings
- **Marks every missing field** as `— not documented` — never silently omits sections
- **Never adds ICD codes** unless the doctor provided them
- **Never resolves contradictions** — flags them for physician verification
- **Never adds medical advice** beyond what the doctor has already documented

---

## Folder Structure

```
consultation-scribe/
├── SKILL.md                        # Main skill — context, goals, constraints, gotchas
├── README.md                       # This file
└── references/
    ├── note-format.md              # Clinical bucket list, section writing guidance, formatting rules, optional modes
    ├── clinical-style.md           # Standard abbreviation list, units, drug naming, severity grading
    └── viewer-setup.md             # Cross-platform viewer launch, port handling, fallback logic
```

---

## Contributing

**New gotchas:** Found a documentation failure mode? Add a `Do NOT...` statement to the Gotchas section in `SKILL.md`.

**New specialty adaptations:** Add a row to the specialty table in `references/note-format.md`.

**New abbreviations:** Add to the standard abbreviation list in `references/clinical-style.md`.

---

## License

MIT License — see `LICENSE` file.

---

*Built with [Claude Code](https://claude.ai/claude-code) · Skill architecture by Anthropic*
