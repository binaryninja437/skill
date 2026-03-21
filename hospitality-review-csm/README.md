# hospitality-review-csm

> A Claude Code skill that acts as an AI Customer Success Manager for hotels, cafes, and restaurants — scraping Google Maps reviews and generating health scores, sentiment analysis, and HTML dashboards.

---

## What It Does

This skill scrapes Google Maps reviews via SerpAPI and transforms raw review data into actionable intelligence. It computes a five-dimension Health Score (0–100), clusters feedback into recurring themes, identifies rating trends, and renders a fully self-contained HTML dashboard. It supports single-store deep dives, multi-store portfolio comparisons, and competitor benchmarking from the same pipeline — all without hardcoding any paths to your machine.

---

## Installation

### Option 1: Claude Code Skills Folder

```bash
git clone https://github.com/binaryninja437/skill.git
cp -r skill/hospitality-review-csm ~/.claude/skills/hospitality-review-csm
```

### Option 2: Manual Copy

Download this folder and place it at:
```
~/.claude/skills/hospitality-review-csm/
```

Claude Code auto-detects it from the `SKILL.md` frontmatter.

### Option 3: Cowork Plugin Bundle

Include the `hospitality-review-csm/` folder in your `.plugin` bundle's `skills/` directory.

---

## Requirements

**Python libraries:**
```bash
pip install requests serpapi --break-system-packages
```

**SerpAPI key** — Create a free account at [serpapi.com](https://serpapi.com) and add your key to:
```
<your-work-dir>/config/serpapi-config.json
```

```json
{ "api_key": "YOUR_KEY_HERE" }
```

**Note:** The free SerpAPI tier provides 100 searches/month. Each property scrape uses ~2 searches (knowledge graph + reviews).

---

## Usage

### Example Prompts That Trigger This Skill

```
"Analyze my Google reviews" + [Google Maps URL]
```
```
"Why is my café's rating dropping? Here's my listing: maps.google.com/..."
```
```
"Compare my hotel's reviews against this competitor: [URL]"
```
```
"Run a review health check on all 3 of my restaurant branches"
```
```
"Build me a review dashboard for my resort"
```

---

## Outputs

| File | Description |
|---|---|
| `<work_dir>/data/<slug>-raw-<date>.json` | Raw SerpAPI review data |
| `<work_dir>/data/master-reviews.csv` | Cumulative review history (weekly appends) |
| `<work_dir>/dashboards/review-analysis-<slug>-<date>.html` | Single-store dashboard |
| `<work_dir>/dashboards/portfolio-dashboard-<date>.html` | Multi-store comparison dashboard |
| `<work_dir>/dashboards/weekly-report-<date>.html` | Weekly pipeline report |

---

## Health Score Dimensions

| Dimension | Weight |
|---|---|
| Average star rating | 30% |
| Rating trend (3-month delta) | 20% |
| Review volume & recency | 20% |
| Sentiment ratio | 20% |
| Owner response rate | 10% |

Tiers: 🟢 80–100 Healthy · 🟡 60–79 Needs Attention · 🟠 40–59 At Risk · 🔴 0–39 Critical

---

## Folder Structure

```
hospitality-review-csm/
├── SKILL.md                         # Main skill — context, goals, constraints, gotchas
├── README.md                        # This file
└── references/
    ├── serpapi-guide.md             # SerpAPI call sequence, pagination, failure modes
    ├── health-score.md              # 5-dimension formula, sentiment classification, topic clustering
    ├── dashboard-spec.md            # Full HTML layout spec for single-store and portfolio dashboards
    └── action-plan.md               # Theme clustering rules, action tiers, staff training format
```

Uses **progressive disclosure** — `SKILL.md` stays lean and reference files are read only when needed for each pipeline phase.

---

## Customising

**Work directory:** The skill asks for your working directory at runtime. No paths are hardcoded.

**SerpAPI key:** Stored in `<work_dir>/config/serpapi-config.json` — never in the skill files.

**Design system:** Colors and layout are defined in `references/dashboard-spec.md`. Adjust the design system section to match your brand.

**Adding a new negative theme:** Add keywords to the theme table in `references/action-plan.md`.

---

## Contributing

**New gotchas:** Found a new failure mode? Add a `Do NOT...` statement to the Gotchas section in `SKILL.md`.

**New insurer/platform:** If you've tested this with Zomato or TripAdvisor scraping, add the API call pattern to `references/serpapi-guide.md`.

**Dashboard improvements:** Layout changes go in `references/dashboard-spec.md`. Only update `SKILL.md` if constraints change.

---

## License

MIT License — see `LICENSE` file.

---

*Built with [Claude Code](https://claude.ai/claude-code) · Skill architecture by Anthropic*
