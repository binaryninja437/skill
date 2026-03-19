# HTML Dashboard Specification — `dashboard.html`

Agent 3 must produce a fully self-contained HTML file with zero external dependencies.
Everything — CSS, JavaScript, SVG — must be inline. This file defines the complete visual
and structural specification.

---

## Table of Contents
1. [Color Palette](#color-palette)
2. [Page Layout](#page-layout)
3. [Sticky Header](#sticky-header)
4. [Status Badge Logic](#status-badge-logic)
5. [Section Cards](#section-cards)
6. [Coverage Progress Bars](#coverage-progress-bars)
7. [Premium SVG Bar Chart](#premium-svg-bar-chart)
8. [Null Value Rendering](#null-value-rendering)
9. [Date Formatting](#date-formatting)
10. [Footer](#footer)
11. [Full HTML Skeleton](#full-html-skeleton)

---

## Color Palette

```
Background:       #0f172a  (deep navy)
Card background:  #1e293b  (dark slate)
Card shadow:      rgba(0,0,0,0.4)
Accent / heading: #3b82f6  (electric blue)
Text primary:     #f1f5f9  (near white)
Text secondary:   #94a3b8  (slate gray)
Null value:       #475569  (muted gray — for em-dash)
Active badge:     #22c55e  (green)
Expired badge:    #ef4444  (red)
Card borders:     section-specific (see below)
```

### Section Card Left-Border Colors

| Section  | Border Color |
|----------|-------------|
| Policy   | `#3b82f6` (blue) |
| Insured  | `#8b5cf6` (purple) |
| Vehicle  | `#f59e0b` (amber) |
| Coverage | `#10b981` (emerald) |
| Premium  | `#ec4899` (pink) |
| Insurer  | `#06b6d4` (cyan) |
| Extras   | `#64748b` (slate) |

---

## Page Layout

```
┌──────────────────────────────────────────────────────────┐
│  STICKY HEADER (full width, navy + blue accent)          │
│  Left: Policy # + Insured Name   Right: Insurer + Dates  │
│  Far right: Active/Expired badge                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  [Card: Policy]  [Card: Insured]  [Card: Vehicle]        │
│  [Card: Coverage]  [Card: Premium]  [Card: Insurer]      │
│  [Card: Extras] (if present)                             │
│                                                          │
├──────────────────────────────────────────────────────────┤
│  FOOTER: "Generated from {source_pdf} on {extracted_at}" │
└──────────────────────────────────────────────────────────┘
```

CSS Grid: `grid-template-columns: repeat(3, 1fr)` on desktop,
`repeat(1, 1fr)` below 768px breakpoint.

Card hover effect:
```css
.card:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 24px rgba(0,0,0,0.5);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}
```

---

## Sticky Header

```html
<header style="position:sticky; top:0; z-index:100; background:#0f172a;
               border-bottom: 2px solid #3b82f6; padding: 14px 24px;
               display:flex; justify-content:space-between; align-items:center;">
  <div>
    <span style="font-size:1.1rem; font-weight:700; color:#f1f5f9;">
      Policy #{policy.number}
    </span>
    <span style="color:#94a3b8; margin-left:12px;">{insured.name}</span>
  </div>
  <div style="display:flex; align-items:center; gap:16px;">
    <span style="color:#94a3b8;">{insurer.company}</span>
    <span style="color:#f1f5f9;">{policy.start_date} – {policy.end_date}</span>
    <!-- STATUS BADGE HERE -->
  </div>
</header>
```

---

## Status Badge Logic

Compare today's date (computed via JavaScript `new Date()`) against `policy.start_date` and
`policy.end_date`.

```javascript
const today = new Date();
const start = new Date("{policy.start_date}");
const end = new Date("{policy.end_date}");
const isActive = today >= start && today <= end;
const badgeColor = isActive ? "#22c55e" : "#ef4444";
const badgeText = isActive ? "Active" : "Expired";
```

Badge HTML:
```html
<span style="background:{badgeColor}; color:#fff; padding:4px 12px;
             border-radius:12px; font-size:0.8rem; font-weight:600;">
  {badgeText}
</span>
```

If either date is null, show a grey "Unknown" badge (`#64748b`).

---

## Section Cards

Each card follows this pattern:

```html
<div class="card" style="background:#1e293b; border-radius:10px;
     padding:20px; box-shadow:0 4px 12px rgba(0,0,0,0.4);
     border-left: 4px solid {section_color};">
  <h3 style="color:#3b82f6; font-size:0.85rem; text-transform:uppercase;
             letter-spacing:0.08em; margin:0 0 14px 0;">{Section Name}</h3>
  <table style="width:100%; border-collapse:collapse;">
    <tr>
      <td style="color:#94a3b8; font-size:0.85rem; padding:5px 0;">Field label</td>
      <td style="color:#f1f5f9; font-size:0.85rem; text-align:right;">Value</td>
    </tr>
    <!-- repeat for each field -->
  </table>
</div>
```

For null values, see the Null Value Rendering section below.

---

## Coverage Progress Bars

Render `liability_limit`, `comprehensive_deductible`, and `collision_deductible` as visual
bar indicators. Use the maximum value in that set as the 100% baseline.

```html
<div style="margin-top:6px;">
  <div style="display:flex; justify-content:space-between; font-size:0.8rem; color:#94a3b8;">
    <span>Liability Limit</span>
    <span>$100,000</span>
  </div>
  <div style="background:#0f172a; border-radius:4px; height:8px; margin-top:3px;">
    <div style="background:#3b82f6; width:100%; height:8px; border-radius:4px;"></div>
  </div>
</div>
```

Calculate `width` as `(value / max_value * 100).toFixed(1) + "%"`. If value is null, skip the
bar and show `—` instead.

---

## Premium SVG Bar Chart

Render the `premium.breakdown` object as a horizontal bar chart using inline SVG.
Place this inside the Premium card, below the field table.

```html
<svg width="100%" height="{chart_height}" style="margin-top:14px;">
  <!-- Y-axis labels + bars, one per breakdown entry -->
  <!-- Example for one bar: -->
  <text x="0" y="18" fill="#94a3b8" font-size="11">Liability</text>
  <rect x="110" y="6" width="{bar_width}" height="14"
        fill="#3b82f6" rx="3"/>
  <text x="{bar_width + 116}" y="18" fill="#f1f5f9" font-size="11">$420</text>
  <!-- repeat for each breakdown item, spacing rows 28px apart -->
</svg>
```

Bar width calculation: `(value / total_premium) * 200` (max bar width = 200px).
Chart height: `breakdown_count * 28 + 10`.

---

## Null Value Rendering

Wherever a value is `null` in the JSON, render:

```html
<span style="color:#475569;">—</span>
```

Never render the string `"null"`, `"N/A"`, or an empty string for null fields.

---

## Date Formatting

All dates displayed in the dashboard must be in `DD MMM YYYY` format (e.g. `01 Mar 2025`).

Use this JavaScript helper:
```javascript
function formatDate(isoStr) {
  if (!isoStr) return "—";
  const d = new Date(isoStr);
  if (isNaN(d)) return isoStr; // fallback: show as-is if unparseable
  return d.toLocaleDateString("en-GB", { day:"2-digit", month:"short", year:"numeric" });
}
```

---

## Footer

```html
<footer style="text-align:center; padding:20px; color:#475569; font-size:0.8rem;
               border-top:1px solid #1e293b; margin-top:40px;">
  Generated from <strong style="color:#64748b;">{source_pdf}</strong>
  on {formatted_extracted_at}
</footer>
```

---

## Opening the Dashboard (Cross-Platform)

After writing `dashboard.html`, open it without blocking the process:

```python
import subprocess, sys, os

dashboard_path = os.path.join(work_dir, "dashboard.html")

if sys.platform == "win32":
    subprocess.Popen(["cmd", "/c", "start", "", dashboard_path])
elif sys.platform == "darwin":
    subprocess.Popen(["open", dashboard_path])
else:
    subprocess.Popen(["xdg-open", dashboard_path])
```

Never use `os.system` or a blocking `subprocess.run` — the pipeline must not hang waiting for the browser.

---

## Full HTML Skeleton

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Insurance Policy — {policy.number}</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: system-ui, -apple-system, sans-serif;
      background: #0f172a;
      color: #f1f5f9;
      font-size: 14px;
      min-height: 100vh;
    }
    .grid {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 20px;
      padding: 24px;
      max-width: 1200px;
      margin: 0 auto;
    }
    @media (max-width: 768px) {
      .grid { grid-template-columns: 1fr; }
    }
    .card {
      background: #1e293b;
      border-radius: 10px;
      padding: 20px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.4);
      transition: transform 0.2s ease, box-shadow 0.2s ease;
    }
    .card:hover {
      transform: translateY(-3px);
      box-shadow: 0 8px 24px rgba(0,0,0,0.5);
    }
    /* ... remainder of styles ... */
  </style>
</head>
<body>
  <!-- HEADER -->
  <!-- GRID OF CARDS -->
  <!-- FOOTER -->
  <script>
    // Status badge, date formatting, and bar chart width calculations go here
  </script>
</body>
</html>
```

Agent 3 should flesh out this skeleton using the data from `insurance_data.json`,
replacing all `{placeholder}` values with actual data rendered inline.
