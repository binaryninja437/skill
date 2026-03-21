# Note Format Reference

This file defines the clinical data bucket list for Agent 1, section-by-section writing guidance
for Agent 2, formatting rules, and optional output modes.

---

## Agent 1 — Clinical Data Buckets

Agent 1 must extract and organise raw source content into these labelled buckets before passing
to Agent 2. Each bucket should be populated with verbatim or closely paraphrased content from
the source — not interpreted or embellished.

| Bucket | What to capture |
|---|---|
| `demographics` | Patient name/ID (if given), age, sex, date of visit |
| `chief_complaint` | Primary reason for visit, in patient's own words or doctor's restatement |
| `hpi` | Onset, duration, character, severity, associated symptoms, aggravating/relieving factors, relevant timeline |
| `past_history` | Chronic conditions, prior surgeries, hospitalisations |
| `medications` | Name, dose, frequency — exactly as stated |
| `allergies` | Drug / food / environmental allergies |
| `examination` | Vitals first, then system-by-system findings |
| `investigations_done` | Test results already available |
| `investigations_ordered` | Tests the doctor ordered today |
| `assessment` | Working diagnoses, clinical impressions, problem list |
| `plan` | Medications prescribed, tests ordered, referrals, patient instructions, follow-up timing |
| `flags` | Ambiguities, contradictions, or gaps Agent 2 needs to handle |

**If a bucket has no data from the source:** leave it empty and label it `MISSING`. Agent 2 will
render these as `— not documented`.

---

## Agent 2 — Section Order and Writing Guidance

Produce sections in exactly this order. Omit a section only if the doctor explicitly says it is
not applicable (e.g., "no examination done today") — in that case, write
`— not applicable per doctor`.

### Section 1: Chief Complaint
One line: the primary reason for the visit. Use the patient's own words where available, or a
concise clinical restatement. Do not expand or interpret.

```
## Chief Complaint
Chest tightness on exertion for 3 weeks.
```

### Section 2: History of Present Illness
Narrative paragraph or structured bullets. Cover: onset, duration, character, severity,
associated symptoms, aggravating/relieving factors, relevant timeline. Use the doctor's
language — do not paraphrase into lay terms here.

```
## History of Present Illness
- Onset: 3 weeks ago, gradual
- Character: pressure-like, rated 6/10
- Associated: mild SOB, no radiation, no diaphoresis
- Relieving: rest
- Aggravating: walking >100 metres
```

### Section 3: Relevant Past History / Medications / Allergies
Three clearly labelled sub-bullets:

```
## Relevant Past History / Medications / Allergies
- **Past history:** HTN (10 years), DM Type 2 (5 years), CABG 2019
- **Current medications:** Metformin 500 mg BD, Amlodipine 5 mg OD
- **Allergies:** Penicillin (rash) — not documented if absent
```

If any sub-item is absent from source: `— not documented`

### Section 4: Examination Findings
Vitals first (if given), then system-wise findings in the order the doctor described them.
If vitals are partially documented, include what is given and mark the rest `— not documented`.

```
## Examination Findings
- **Vitals:** BP **140/90 mmHg**, HR 82 bpm, SpO₂ — not documented
- **CVS:** S1 S2 heard, no murmurs
- **Chest:** clear to auscultation bilaterally
- **Abdomen:** — not documented
```

### Section 5: Investigations
Two sub-bullets — never merge them:

```
## Investigations
- **Done:** ECG — sinus rhythm, no ST changes; Hb **9.2 g/dL** (dated 12 Mar 2026)
- **Ordered:** Stress echo, fasting lipid profile, HbA1c
```

### Section 6: Assessment
Numbered problem list. One line per problem with working diagnosis or clinical impression.
Do not add ICD codes unless the doctor provided them.

```
## Assessment
1. Stable angina — query exertional, awaiting stress echo
2. Hypertension — suboptimally controlled
3. Type 2 diabetes mellitus — monitoring pending HbA1c
```

### Section 7: Plan
Per-problem bullets mirroring the Assessment numbering:

```
## Plan
1. **Angina:** GTN spray PRN prescribed; stress echo booked; avoid strenuous activity
2. **HTN:** Increase Amlodipine to 10 mg OD; review BP in 2 weeks
3. **DM:** Continue Metformin; HbA1c result review at follow-up
- **Follow-up:** 2 weeks or sooner if chest pain worsens
- **Urgent review if:** chest pain at rest, pain radiating to arm/jaw, diaphoresis
```

### Section 8: Patient-Friendly Summary
3–5 short paragraphs. Plain language — no jargon, spell out everything. Second person
("You have…", "Your doctor…"). Cover: what the problem seems to be, what was done today,
what medicines/tests are next, when to return or seek urgent care.

```
## Patient-Friendly Summary

You came in today because you have been feeling tightness in your chest when you walk or
exert yourself, which has been happening for about three weeks.

Your doctor thinks this may be related to your heart, specifically a condition called angina,
where the heart muscle is not getting quite enough blood during activity. This is not a
confirmed diagnosis yet — a test called a stress echocardiogram has been arranged to find out more.

Your blood pressure was a little higher than ideal today. Your doctor has increased one of
your blood pressure medicines (Amlodipine) to help bring it down. Please continue your
diabetes medicines as before.

You have been given a GTN spray. If you feel chest tightness, spray once under your tongue
and rest. If the pain does not ease within 5 minutes or comes back, use the spray again —
but if it still does not settle, call an ambulance.

Please come back in 2 weeks, or sooner if you have chest pain while resting, pain spreading
to your arm or jaw, or you feel sweaty and unwell. Your follow-up will also review your
diabetes blood test result.
```

---

## Formatting Rules

- Use Markdown headings (`##`) and bullet points
- Bold key values: **BP 140/90**, **Hb 9.2 g/dL**, drug names, doses
- Keep the note scannable — a busy doctor should read it in under 60 seconds
- If input was brief: add at the very top `> ⚠ Input was brief — several sections marked "not documented".`
- Do NOT use tables inside clinical sections — use labelled bullets

---

## Handling Ambiguity and Contradictions

| Situation | How to handle |
|---|---|
| Finding mentioned but incomplete (e.g. "crackles" — no side stated) | Transcribe exactly as given; do not add bilateral/unilateral |
| Two conflicting values for same field | Include both; flag `[Note: conflicting information in source — please verify]` |
| Abbreviated term not in standard list | Spell out in parentheses on first use: `SOB (shortness of breath)` |
| Ambiguous diagnosis (e.g. "?MI") | Transcribe as-is: `query MI` or `? myocardial infarction` |
| Date missing from investigation result | Note: `[date not documented]` after the result |

---

## Optional Output Modes

The user may request one of these modes. Apply on top of the standard note format.

### `plain text`
Strip all Markdown: remove `##`, `**`, `-` bullets. Replace with plain indented text and
CAPS section headings. For pasting into EMR/EHR systems that do not render Markdown.

### `brief`
Output only three sections: Assessment, Plan, and Patient-Friendly Summary. Skip all others.
Useful for quick referral letters or handover notes.

### `specialty: [name]`
Adapt section terminology for the named specialty while keeping the same section structure:

| Specialty | Adaptations |
|---|---|
| Cardiology | Lead with rhythm, ECG findings; expand CVS exam; use CCS/NYHA grading |
| Paediatrics | Add weight/height/developmental milestones bucket; note parent/guardian present |
| Orthopaedics | Add joint-specific exam findings (ROM, special tests); expand investigations for imaging |
| Psychiatry | Add MSE (Mental State Examination) section; expand social history |
| Obstetrics | Add gestational age, G/P status, fetal movements, CTG findings |

If a specialty not listed above is requested, apply common sense adaptations and note them
at the top of the output.
