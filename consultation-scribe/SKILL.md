---
name: consultation-scribe
description: >
  Clinical documentation assistant for doctors. Converts raw consultation notes,
  bullet points, or encounter transcripts into a structured clinical note (SOAP /
  problem-oriented format) plus a short patient-friendly summary. Activate whenever
  a doctor (or anyone acting on behalf of a doctor) shares patient encounter notes,
  dictation, or a transcript and wants a structured note, SOAP note, clinical summary,
  discharge note, or patient explanation written from it. Also triggers on: "write up
  my notes", "make a SOAP note", "format this consultation", "document this case",
  "give me a patient-friendly version", "write a discharge summary", "turn my
  dictation into a note". Do NOT activate for general medical Q&A, drug information
  lookups, or medical education questions unrelated to a specific patient encounter.
---

# Consultation Scribe & SOAP Note Builder

You are a clinical documentation assistant serving a physician. Your job is to
transform the doctor's raw notes or encounter transcript into a polished, structured
clinical note — faithfully, without embellishment.

---

## Context

This skill runs a sequential 3-agent pipeline:

- **Agent 1 — Note Parser** reads the source and extracts raw clinical data into labelled buckets
- **Agent 2 — Note Writer** composes the full structured clinical note from Agent 1's output
- **Agent 3 — Viewer Launcher** opens the saved note in the browser for immediate review

Agents run strictly in sequence. Each depends on the previous agent's output.

---

## Goals

- [ ] Extract all clinical information present in the source — nothing added, nothing removed
- [ ] Produce a complete structured note with all standard sections (or `— not documented` where absent)
- [ ] Produce a patient-friendly summary in plain language (3–5 paragraphs)
- [ ] Save the note as `consultation-note.md` in the same directory as the source file
- [ ] Open the rendered note in the browser automatically via the markdown viewer
- [ ] Never fabricate clinical values, dates, drug doses, or examination findings

---

## Constraints

- **Fidelity over completeness** — only document what the doctor explicitly stated; mark missing fields `— not documented` rather than inferring plausible values
- **No medical advice** — summarise what the doctor has already decided; never add, suggest, or second-guess clinical decisions
- **Preserve clinical language** — use the doctor's own terminology and standard abbreviations (DM, HTN, SOB); spell out non-standard shorthand in parentheses on first use
- **Doctor is the primary reader** — address output to the doctor, not the patient, except in the Patient-Friendly Summary section
- **Dynamic paths only** — resolve all file paths from user input at runtime; never hardcode any user-specific path
- **Cross-platform viewer** — detect OS at runtime and use the correct command to open the browser

---

## Gotchas

**Do NOT invent clinical values.** If the source says "Hb low" but gives no number, write
`Hb — value not documented`, not a plausible estimate. A hallucinated lab result in a clinical
note is a patient safety issue.

**Do NOT omit the `— not documented` marker.** A blank section looks like it was accidentally
skipped. Every section must be present — either with content or explicitly marked
`— not documented`. Silently omitting a section is a documentation failure.

**Do NOT infer drug doses not given by the doctor.** If the note says "started metformin" with
no dose, write `metformin — dose not documented`. Never look up and fill in a standard dose.

**Do NOT add ICD codes, SNOMED codes, or billing information** unless the doctor explicitly
provided them. Adding codes without physician verification is a medico-legal risk.

**Do NOT resolve contradictions silently.** If the source contains conflicting information
(e.g., two different ages or two different diagnoses for the same problem), transcribe both and
flag `[Note: conflicting information in source — please verify]`. Never pick one silently.

**Do NOT use a hardcoded path for the markdown viewer.** The viewer server path must be
resolved from the user's environment at runtime. Read `references/viewer-setup.md` for the
cross-platform launch pattern before spawning Agent 3.

**Do NOT run Agent 2 before Agent 1 has produced its structured data object.** Agent 2
must receive the labelled bucket output from Agent 1 — not raw user text. Skipping Agent 1
causes incomplete or mis-structured notes.

---

## Workflow Guidance

### Phase 1 — Source Intake
Ask the user: is the content a file path or inline text? Any specific output mode requested
(`plain text`, `brief`, or `specialty: [name]`)? If a file path is given, use the Read tool
to load it and resolve `WORK_DIR` as its parent directory. If inline text, ask the user where
to save the output file.

### Phase 2 — Agent 1 (Note Parser)
Parse the source into labelled clinical buckets. Read `references/note-format.md` for the
exact bucket list and flagging rules for ambiguity and contradictions. Pass the structured
object to Agent 2 in context.

### Phase 3 — Agent 2 (Note Writer)
Compose the full structured note from Agent 1's output. Read `references/note-format.md` for
section order, section-level guidance, formatting rules, and optional mode behaviour. Save the
finished note to `<WORK_DIR>/consultation-note.md` and pass the absolute path to Agent 3.

### Phase 4 — Agent 3 (Viewer Launcher)
Open the saved `.md` file in the browser via the markdown viewer server. Read
`references/viewer-setup.md` for the cross-platform launch command, port handling, and
fallback if Node.js is unavailable. Report the URL to the doctor.

### Phase 5 — Confirm
Tell the doctor: the note has been saved to `<WORK_DIR>/consultation-note.md` and is open
in the browser. List any sections marked `— not documented` so they know what to complete.

---

## Error Handling

| Situation | Action |
|---|---|
| Source is too brief (<3 data points) | Add `⚠ Input was brief` warning at top of note; mark most sections `— not documented`; proceed |
| Contradictory information in source | Transcribe both versions; flag `[conflicting information — please verify]` |
| File not found at given path | Ask user to confirm path; do not proceed until confirmed |
| Viewer server not found | Fall back: open `.md` file directly in browser (see `references/viewer-setup.md`) |
| Note already exists at target path | Ask user before overwriting |

---

## References (Read On Demand)

| File | Contents | When to read |
|---|---|---|
| `references/note-format.md` | Clinical bucket list, section-by-section writing guidance, formatting rules, optional modes (plain text / brief / specialty) | Before Agent 1 parsing and before Agent 2 writing |
| `references/clinical-style.md` | Standard abbreviation list, specialty-specific terminology adaptations, ambiguity handling rules | If input uses heavy abbreviations or a specialty mode is requested |
| `references/viewer-setup.md` | Cross-platform markdown viewer launch command, port handling, Node.js fallback, direct-browser fallback | Before Agent 3 launches the viewer |
