# Action Plan Reference

This file defines theme clustering rules, action plan tier structure, staff training alert format,
and multi-store cross-network patterns.

---

## Theme Clustering Rules

**Step 1 — Identify negative reviews** (rating ≤ 3 or sentiment = Negative).

**Step 2 — Scan for theme keywords** across all negative review texts:

| Theme | Keywords to scan for |
|---|---|
| Slow service | slow, wait, waiting, took long, forever, delay, late |
| Rude/unhelpful staff | rude, unfriendly, attitude, ignored, unhelpful, dismissive |
| Food quality issues | cold, stale, bad taste, undercooked, overcooked, bland, wrong order |
| Noise/crowding | noisy, crowded, too loud, packed, cramped, no seating |
| Overpriced | expensive, overpriced, not worth, pricey, value for money |
| Hygiene concerns | dirty, unclean, cockroach, smell, unhygienic, flies |
| Stock/availability | not available, out of stock, limited menu, didn't have |
| Long wait time | reservation wait, queue, 30 minutes, 1 hour, waited too long |

**Step 3 — Count occurrences** (a review can match multiple themes).

**Step 4 — Tier the themes:**
- **Critical** (act in Week 1): theme appears in ≥ 20% of negative reviews
- **Needs Improvement** (Month 1): theme appears in 10–19% of negative reviews
- **Monitor** (Quarterly): theme appears in 5–9% of negative reviews
- **Ignore** (for now): theme appears in < 5% of negative reviews

---

## Action Plan Structure

### Tab 1 — Critical (Week 1)
Issues found in 20%+ of negative reviews. Format each card as:

```
🔴 [Theme Name]
Appears in [X]% of negative reviews ([N] mentions)

What reviewers say: "[short quote]"

Immediate actions:
1. [Specific action — name the role responsible]
2. [Specific action]
3. [Specific action]
```

### Tab 2 — Improvements (Month 1)
Declining positive themes or 10–19% negative themes. Format:

```
🟠 [Theme Name]
Trend: [Improving / Declining / Stable]

What to do:
1. [Action]
2. [Action]
```

### Tab 3 — Quick Wins (Amplify in marketing)
Top 3 positive themes with highest mention frequency. Format:

```
🟢 [Strength]
Mentioned positively by [X]% of reviewers

Marketing opportunity:
- Use in: [Instagram / Google listing / menu board]
- Sample caption: "[suggested copy]"
```

### Tab 4 — Staff Training
Behavioral issues identified from reviews that name or describe specific staff interactions.
**Do not name individual staff members** — describe the behavior pattern.

```
🔵 Training Priority: [Behavior]
Evidence: "[anonymized quote]"

Training focus:
1. [Specific skill or behavior to train]
2. [Role-play scenario]
3. [KPI to track improvement]
```

---

## Response Strategy

Include this in the action plan for own-property analysis:

**Priority 1 — Respond to all 1★ reviews within 48 hours**
Template:
> "Thank you for sharing your experience. We're sorry to hear this fell short of your expectations.
> We'd love the opportunity to make it right — please reach out to us at [contact]. We take all
> feedback seriously and are working to improve."

**Priority 2 — Respond to all 3★ unanswered reviews**
These represent reachable customers. A good response can turn a 3★ into a return visit.

**Priority 3 — Acknowledge 5★ reviews**
Brief, genuine acknowledgment. Don't use a template — vary the wording.

---

## Multi-Store Cross-Network Patterns

When running a portfolio analysis, surface these insights:

### Best Performer Analysis
Identify the top-scoring store. Extract its top 3 positive themes. Ask:
- Does this store have a different staff ratio?
- Are its items or pricing different?
- Does it have higher response rates?

Surface as: "What [Top Store] does differently that other stores should replicate:"

### Shared Failure Patterns
If the same negative theme appears in 2+ stores, it's a **network-level problem**, not a
store-level one. Flag it as:

```
⚠️ Network-Wide Issue: [Theme]
Affects: [Store A], [Store B], [Store C]
This suggests a systemic issue (training, supplier, or policy) — not a store manager problem.
```

### Signature Items by Store
From positive review mentions, identify the item most associated with each store:
> "Reviewers at [Store A] most often mention [item]. Consider featuring it in that store's local
> marketing."

---

## Weekly Pipeline Output

When the weekly pipeline runs (via `scripts/weekly-pipeline.py`), the action plan should focus on
**changes since last week**:
- New critical issues that weren't flagged before
- Themes that have worsened (% increase in mentions)
- Themes that have improved (% decrease)
- Response rate change (better or worse than last week)

Prefix these with 📈 (improved) or 📉 (worsened) for quick visual scanning.
