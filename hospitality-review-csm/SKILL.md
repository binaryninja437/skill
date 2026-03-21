---
name: hospitality-review-csm
description: >
  Customer Success Manager for hotels, cafes, restaurants, resorts, B&Bs, and any hospitality
  property — scrapes Google Maps reviews and generates health scores, sentiment analysis, rating
  trends, topic clustering, and competitor benchmarking as a downloadable HTML dashboard.
  TRIGGER AGGRESSIVELY for any of: (1) Any Google Maps URL pasted (maps.google.com,
  google.com/maps, goo.gl/maps, maps.app.goo.gl); (2) "analyze my reviews", "check my Google
  reviews", "what are customers saying about my hotel/cafe/restaurant"; (3) "why is my rating
  dropping", "my stars dropped", "bad reviews lately"; (4) "compare my ratings with a competitor",
  "how do I stack up against nearby hotels/cafes"; (5) "review dashboard", "review health score",
  "CSM report for my property"; (6) "hotel/cafe/restaurant feedback analysis"; (7) Any hospitality
  business owner asking about online reputation, Google listing, or customer feedback. Also trigger
  for Zomato, MagicPin, TripAdvisor reviews, and multi-store/multi-branch comparison. Do NOT
  trigger for general recommendations, bookings, or non-hospitality review analysis.
---

# Hospitality Review CSM

## Context

This skill acts as a Customer Success Manager AI for hospitality businesses. It scrapes Google
Maps reviews via SerpAPI, runs sentiment analysis, computes a health score across five dimensions,
clusters recurring feedback themes, and renders a fully self-contained HTML dashboard. It handles
single-store deep dives, multi-store portfolio comparisons, and competitor benchmarking from the
same pipeline.

The skill exists because hospitality owners need actionable intelligence from their reviews, not
raw data dumps. Every output prioritises what to fix first, why, and how.

---

## Goals

- [ ] Understand user intent: own property or competitor, single store or portfolio
- [ ] Scrape reviews via SerpAPI (knowledge graph flow); fall back to user paste if blocked
- [ ] Compute health score (0–100) across 5 weighted dimensions
- [ ] Classify sentiment, cluster topics, and surface rating trends
- [ ] Generate a self-contained HTML dashboard saved to `<WORK_DIR>/dashboards/`
- [ ] Produce a chat summary with health score, top issues, top strengths, and file path
- [ ] Never fabricate reviews or scores — all analysis must come from real scraped data

---

## Constraints

- **Dynamic paths only** — resolve `WORK_DIR` from user input at runtime; never hardcode any path
- **SerpAPI knowledge graph flow** — always use `engine: google` → `place_id` → `engine: google_maps_reviews`; the `google_maps` search type does NOT work for Indian locations
- **No fabrication** — if data is unavailable, state so clearly; do not invent reviews or metrics
- **Self-contained HTML** — dashboards must render with no internet connection; Chart.js may load from CDN as a deliberate exception since it is a charting library, not content
- **Anonymise reviewers** — always anonymize reviewer names in output (Reviewer A, B, C…)
- **One question round** — ask all intent questions together in Step 1, not one at a time

---

## Gotchas

**Do NOT use `engine: google_maps` with `type: search` for Indian locations.** This consistently
fails. Always use the two-step knowledge graph flow: `engine: google` to get `place_id`, then
`engine: google_maps_reviews` with that `place_id`. Read `references/serpapi-guide.md` before
calling the scraper.

**Do NOT hardcode `WORK_DIR`, script paths, or Python venv paths.** Ask the user for their
working directory at setup. Resolve all file paths dynamically using `os.path.join(work_dir, ...)`.
Never reference any specific user's machine path.

**Do NOT invent health score dimensions you haven't computed.** If owner response rate data is
unavailable from the scrape, show that dimension as `null` in the score breakdown rather than
guessing. A fabricated score destroys trust faster than an incomplete one.

**Do NOT produce a single-run dashboard when the user asked for a portfolio view.** If the user
says "all my stores" or provides multiple URLs, generate individual per-store reports AND a
portfolio comparison dashboard. Missing the portfolio view is the most common output failure.

**Do NOT skip deduplication when appending to a master CSV.** The same review can appear in
consecutive weekly scrapes. Always check for duplicate `review_id` or `(reviewer, date, text)`
triplets before writing to `data/master-reviews.csv`.

**Do NOT label a new property (under 3 months old) the same way as an established one.** New
listings naturally have inflated ratings (honeymoon effect). Flag this explicitly in the health
score commentary so the owner doesn't over-interpret a 4.9★ average.

**Do NOT put raw sentiment keywords in the action plan without grouping by theme.** Listing 40
individual complaints is useless. Always cluster into themes first (read `references/action-plan.md`)
and surface only the top 3–5 issues per tier.

**Do NOT make the dashboard dependent on external CSS files or fonts.** Everything except
Chart.js must be inline. No Google Fonts `<link>` tags, no external stylesheets.

---

## Workflow Guidance

### Phase 1 — Intent Gathering
Ask the user three things together (not separately): Is this your own property or a competitor?
Single store or multiple stores / "all my branches"? Any specific concern driving this analysis?
Skip questions where context is already clear from the user's message.

### Phase 2 — Scraping
Try SerpAPI first using the knowledge graph flow. Read `references/serpapi-guide.md` for the
exact API call sequence and pagination pattern. If scraping fails (CAPTCHA, missing key, quota),
fall back to asking the user to paste review text manually.

For multi-store runs, scrape each store sequentially and save each output to
`<WORK_DIR>/data/store-<slug>-data.json` before proceeding to analysis.

### Phase 3 — Analysis
Compute health score, sentiment, topic clusters, and rating trend. Read
`references/health-score.md` for the exact weighting formula and tier thresholds. For competitor
or portfolio mode, run the same analysis per property and produce a comparative summary.

### Phase 4 — Dashboard Generation
Build the HTML dashboard. Read `references/dashboard-spec.md` for the full layout specification,
design system, and Chart.js patterns for each chart type. Save to
`<WORK_DIR>/dashboards/<filename>.html` using the naming conventions below.

**Naming conventions:**
- Single store: `review-analysis-<store-slug>-<YYYY-MM-DD>.html`
- Portfolio: `portfolio-dashboard-<YYYY-MM-DD>.html`
- Weekly report: `weekly-report-<YYYY-MM-DD>.html`

### Phase 5 — Chat Summary
Output a structured summary in chat (see template in References). Then tell the user the full
path to the dashboard file.

---

## Path Resolution — First Step Every Time

Before any file I/O, ask the user for their working directory if not already provided:

```python
import os
work_dir = os.path.abspath(user_provided_path)  # e.g. "/Users/jane/hospitality-reports"
dashboards_dir = os.path.join(work_dir, "dashboards")
data_dir       = os.path.join(work_dir, "data")
scripts_dir    = os.path.join(work_dir, "scripts")
logs_dir       = os.path.join(work_dir, "logs")
for d in [dashboards_dir, data_dir, scripts_dir, logs_dir]:
    os.makedirs(d, exist_ok=True)
```

Never hardcode any path. Pass `work_dir` to every agent or subprocess call at runtime.

---

## Chat Summary Template

```
## Review Analysis: [Property Name]

**Health Score: XX/100** 🟢/🟡/🟠/🔴 [Tier]

**Key Metrics**
- Average Rating: X.X ★
- Reviews Analyzed: XX
- Sentiment: XX% positive, XX% negative
- Response Rate: XX%

**Top Issues** (act now)
1. [Issue] — XX% of negative reviews

**Top Strengths** (amplify in marketing)
1. [Strength]

📊 Dashboard saved: <WORK_DIR>/dashboards/[filename].html
```

---

## Error Handling

| Problem | Action |
|---|---|
| Google CAPTCHA blocks scrape | Fall back: ask user to paste reviews manually |
| Too few reviews (<10) | Note limitation and lower confidence; proceed |
| Non-hospitality URL | Tell user, ask for correct URL |
| SerpAPI key missing | Check `<WORK_DIR>/config/serpapi-config.json` |
| Pipeline fails | Check `<WORK_DIR>/logs/pipeline-log.txt` |
| Multi-store: one store fails | Log the failure, continue with remaining stores |

---

## References (Read On Demand)

| File | Contents | When to read |
|---|---|---|
| `references/serpapi-guide.md` | Exact SerpAPI call sequence, pagination, Indian location fix, multi-store pattern | Before scraping |
| `references/health-score.md` | 5-dimension weighting formula, tier thresholds, null handling | Before computing health score |
| `references/dashboard-spec.md` | Full layout spec for single-store (9 sections) and portfolio (9 sections), design system, Chart.js patterns | Before generating HTML |
| `references/action-plan.md` | Theme clustering rules, action plan tiers, staff training alert format, multi-store cross-network patterns | Before generating action plan |
