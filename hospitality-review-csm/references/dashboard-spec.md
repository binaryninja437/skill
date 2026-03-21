# Dashboard Specification

This file defines the full layout, design system, and Chart.js patterns for both the
single-store dashboard and the portfolio (multi-store) dashboard.

---

## Table of Contents
1. [Design System](#design-system)
2. [Single-Store Dashboard — 9 Sections](#single-store-dashboard)
3. [Portfolio Dashboard — 9 Sections](#portfolio-dashboard)
4. [Chart.js Patterns](#chartjs-patterns)
5. [Common Mistakes](#common-mistakes)

---

## Design System

```
Font:           'Segoe UI', Arial, Helvetica, sans-serif
Background:     #F9F9F7
Cards:          #FFFFFF, border: 1px solid #E0E0E0
Shadows:        NONE — flat design only
Primary:        #BD7144  (burnt orange)
Secondary:      #334D5C  (dark slate)
Positive:       #27AE60  (green)
Negative:       #E74C3C  (red)
Warning:        #F39C12  (amber)
Table headers:  bg #334D5C, text white
Table zebra:    alternating #FFFFFF / #F5F5F3
Table totals:   bg #BD7144, text white, font-weight bold
```

**Critical rule:** No external CSS, no Google Fonts `<link>` tags, no external stylesheets.
Chart.js CDN is the only allowed external resource:
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
```

---

## Single-Store Dashboard

### Section 1 — Header
```html
<header style="background:#334D5C; color:white; padding:24px 32px; display:flex; justify-content:space-between; align-items:center;">
  <div>
    <div style="width:48px;height:48px;background:#BD7144;border-radius:50%;display:flex;align-items:center;justify-content:center;font-size:1.4rem;font-weight:700;">
      <!-- First letter of property name -->
    </div>
    <h1 style="font-size:1.5rem; margin:8px 0 4px;">[Property Name] — Review Analysis</h1>
    <p style="opacity:0.8; font-size:0.9rem;">Generated on [DD MMM YYYY]</p>
  </div>
  <div>
    <!-- Health badge: 🟢/🟡/🟠/🔴 + score + tier label -->
    <span style="background:[tier_color]; padding:8px 20px; border-radius:20px; font-size:1.1rem; font-weight:700;">
      [SCORE]/100 — [TIER]
    </span>
  </div>
</header>
```

### Section 2 — KPI Row (4 cards)
Cards: Average Rating ★, Total Reviews, Sentiment % (positive), Response Rate %

Each card:
```html
<div style="background:#fff; border:1px solid #E0E0E0; border-radius:8px; padding:20px; text-align:center;">
  <div style="font-size:2rem; font-weight:700; color:#334D5C;">[value]</div>
  <div style="font-size:0.85rem; color:#666; margin-top:4px;">[label]</div>
</div>
```

Layout: `display:grid; grid-template-columns: repeat(4,1fr); gap:16px;`

### Section 3 — Health Score Banner
Large score circle (100px diameter) with the numeric score centred in `#BD7144`.
Five dimension pills in a row: `[dim_name]: [score]/100` — grey pill, turns burnt orange if score < 60.

### Section 4 — Crosstab Table (Weekly Performance)
Columns: Week | Reviews | Avg Rating | Sentiment % | Response Rate | Score

```html
<table style="width:100%; border-collapse:collapse; font-size:0.88rem;">
  <thead>
    <tr style="background:#334D5C; color:white;">
      <th style="padding:10px 14px; text-align:left;">Week</th>
      <!-- ... -->
    </tr>
  </thead>
  <tbody>
    <!-- Alternating rows: #fff / #F5F5F3 -->
    <!-- Totals row: background #BD7144, color white, font-weight bold -->
  </tbody>
</table>
```

### Section 5 — Charts Row 1 (2-column)
- **Left:** Stacked bar chart — review volume per week, stacked by sentiment (green/red/grey)
- **Right:** Dual line chart — actual avg rating (blue line) vs 4.8 target (dashed orange line)

### Section 6 — Charts Row 2 (3-column)
- **Left:** Donut chart — sentiment breakdown (Positive/Neutral/Negative)
- **Center:** Horizontal bar chart — topic performance scores with a vertical 80% target line
- **Right:** Bar chart — star rating distribution (1★ through 5★)

### Section 7 — Action Plan (4-tab panel)
Tabs: Critical | Improvements | Quick Wins | Staff Training

Each tab contains 3–4 action cards. Card left-border colours:
- Critical: `#E74C3C` (red)
- Improvements: `#F39C12` (amber)
- Quick Wins: `#27AE60` (green)
- Staff Training: `#3498DB` (blue)

Tab switching via inline JavaScript — no external libraries.

### Section 8 — Recent Reviews (2-column)
- **Left column:** Top 5 positive reviews — left border `#27AE60`
- **Right column:** Top 5 negative reviews — left border `#E74C3C`

Each review card shows: anonymized reviewer name, star rating, date, review text snippet (≤150 chars).

### Section 9 — Footer
```html
<footer style="text-align:center; padding:20px; color:#999; font-size:0.8rem; border-top:1px solid #E0E0E0;">
  This report is generated from Google Maps review data. All reviewer names have been anonymized.
  Generated on [date] by Hospitality Review CSM.
</footer>
```

---

## Portfolio Dashboard

### Section 1 — Header
Title: "Portfolio Performance Report", store count, date range.

### Section 2 — Network KPI Row
Same 4-card format but aggregated across all stores: portfolio avg rating, total reviews, network sentiment %, network response rate.

### Section 3 — Store Scorecards (3-column grid)
Each card shows: rank badge (🥇 gold / 🥈 silver / ⚠️ red for lowest), store name, health score, avg rating, review count.

Lowest-performing store gets: `border: 2px solid #E74C3C; background: #FFF5F5;`

### Section 4 — Comparison Crosstab
Rows: one per store + Network Total row (burnt orange bg).
Columns: Store | Health Score | Avg Rating | Reviews | Sentiment % | Response Rate | Trend

### Section 5 — Charts Row 1 (2-column)
- **Left:** Grouped bar — avg rating per store + 4.8 target line
- **Right:** Stacked bar — sentiment breakdown per store

### Section 6 — Charts Row 2 (2-column)
- **Left:** Horizontal bar — health scores per store + 80-point target line
- **Right:** Radar chart — 5 health dimensions per store (one dataset per store)

### Section 7 — Red Flag Alert
Full-width alert box with `border-left: 6px solid #E74C3C; background: #FFF5F5;` for the lowest-performing store. Shows top 3 action items inline.

### Section 8 — Action Plan Tabs
4 tabs covering network-level priorities (not per-store).

### Section 9 — Footer
Same as single-store.

---

## Chart.js Patterns

### Stacked Bar (sentiment volume)
```javascript
new Chart(ctx, {
  type: 'bar',
  data: {
    labels: weekLabels,
    datasets: [
      { label: 'Positive', data: posData, backgroundColor: '#27AE60', stack: 'sentiment' },
      { label: 'Neutral',  data: neuData, backgroundColor: '#F39C12', stack: 'sentiment' },
      { label: 'Negative', data: negData, backgroundColor: '#E74C3C', stack: 'sentiment' },
    ]
  },
  options: { responsive: true, plugins: { legend: { position: 'bottom' } },
             scales: { x: { stacked: true }, y: { stacked: true } } }
});
```

### Dual Line (actual vs target)
```javascript
datasets: [
  { label: 'Avg Rating', data: ratingData, borderColor: '#334D5C', tension: 0.3 },
  { label: '4.8 Target', data: Array(weekLabels.length).fill(4.8),
    borderColor: '#BD7144', borderDash: [6, 4], pointRadius: 0 }
]
```

### Radar (portfolio dimensions)
```javascript
new Chart(ctx, {
  type: 'radar',
  data: {
    labels: ['Avg Rating', 'Trend', 'Volume', 'Sentiment', 'Response Rate'],
    datasets: stores.map((s, i) => ({
      label: s.name,
      data: s.dimension_scores,
      borderColor: COLORS[i],
      backgroundColor: COLORS[i] + '33',
    }))
  },
  options: { scales: { r: { min: 0, max: 100 } } }
});
```

---

## Common Mistakes

- Do NOT use shadows on cards — flat design only
- Do NOT use external CSS or fonts — everything inline except Chart.js CDN
- Do NOT display raw reviewer names — always anonymize (Reviewer A, B, C…)
- Do NOT skip the tab-switching JavaScript — the 4-tab action plan requires it inline
- Do NOT render `null` as the word "null" — use `—` (em-dash) for missing values
- Do NOT forget the portfolio totals row in burnt orange — it's visually critical for the comparison table
