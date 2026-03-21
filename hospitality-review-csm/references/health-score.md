# Health Score Reference

This file defines the exact formula, dimension weights, tier thresholds, and null-handling rules
for computing the Hospitality Health Score (0–100).

---

## Five Dimensions

| # | Dimension | Weight | What to measure |
|---|---|---|---|
| 1 | Average star rating | 30% | Mean of all review ratings |
| 2 | Rating trend | 20% | Direction of avg rating over last 3 months vs prior 3 months |
| 3 | Review volume & recency | 20% | Number of reviews in the last 30 days |
| 4 | Sentiment ratio | 20% | % of reviews classified as Positive |
| 5 | Owner response rate | 10% | % of reviews that received an owner reply |

---

## Dimension Scoring

### 1. Average Star Rating (30%)

| Rating | Score |
|---|---|
| 4.8 – 5.0 | 100 |
| 4.5 – 4.79 | 90 |
| 4.0 – 4.49 | 80 |
| 3.5 – 3.99 | 60 |
| 3.0 – 3.49 | 40 |
| 2.0 – 2.99 | 20 |
| < 2.0 | 0 |

```python
def score_avg_rating(avg):
    if avg >= 4.8: return 100
    if avg >= 4.5: return 90
    if avg >= 4.0: return 80
    if avg >= 3.5: return 60
    if avg >= 3.0: return 40
    if avg >= 2.0: return 20
    return 0
```

### 2. Rating Trend (20%)

Compare the average rating of the last 3 months vs the prior 3 months.

| Trend | Score |
|---|---|
| Improving (Δ > +0.2) | 100 |
| Stable (−0.2 ≤ Δ ≤ +0.2) | 60 |
| Declining (Δ < −0.2) | 20 |
| Insufficient data (<5 reviews in either window) | 50 (neutral) |

```python
def score_trend(recent_avg, prior_avg):
    if recent_avg is None or prior_avg is None:
        return 50  # neutral — insufficient data
    delta = recent_avg - prior_avg
    if delta > 0.2:  return 100
    if delta >= -0.2: return 60
    return 20
```

### 3. Review Volume & Recency (20%)

Count reviews posted in the last 30 calendar days.

| Reviews/month | Score |
|---|---|
| 10+ | 100 |
| 5–9 | 70 |
| 1–4 | 40 |
| 0 | 10 |

```python
def score_volume(reviews_last_30_days):
    if reviews_last_30_days >= 10: return 100
    if reviews_last_30_days >= 5:  return 70
    if reviews_last_30_days >= 1:  return 40
    return 10
```

### 4. Sentiment Ratio (20%)

% of reviews classified as Positive (see Sentiment Classification below).

| % Positive | Score |
|---|---|
| ≥ 80% | 100 |
| 60–79% | 75 |
| 40–59% | 50 |
| 20–39% | 25 |
| < 20% | 0 |

```python
def score_sentiment(positive_pct):
    if positive_pct >= 80: return 100
    if positive_pct >= 60: return 75
    if positive_pct >= 40: return 50
    if positive_pct >= 20: return 25
    return 0
```

### 5. Owner Response Rate (10%)

% of all reviews that have an owner reply.

| Response rate | Score |
|---|---|
| ≥ 80% | 100 |
| 50–79% | 70 |
| 20–49% | 40 |
| < 20% | 10 |
| Data unavailable | null |

```python
def score_response_rate(response_pct):
    if response_pct is None: return None  # don't fabricate
    if response_pct >= 80: return 100
    if response_pct >= 50: return 70
    if response_pct >= 20: return 40
    return 10
```

---

## Final Health Score Calculation

```python
def compute_health_score(d1, d2, d3, d4, d5):
    """
    d1–d5 are the dimension scores (0–100 or None).
    If d5 (response rate) is None, redistribute its 10% weight to d1 proportionally.
    """
    weights = {1: 0.30, 2: 0.20, 3: 0.20, 4: 0.20, 5: 0.10}
    scores  = {1: d1,   2: d2,   3: d3,   4: d4,   5: d5}

    if scores[5] is None:
        # Redistribute weight 5 to dimension 1
        weights[1] += weights.pop(5)
        scores.pop(5)

    total = sum(scores[k] * weights[k] for k in scores)
    return round(total, 1)
```

---

## Health Tiers

| Score | Tier | Badge |
|---|---|---|
| 80–100 | Healthy | 🟢 |
| 60–79 | Needs Attention | 🟡 |
| 40–59 | At Risk | 🟠 |
| 0–39 | Critical | 🔴 |

---

## Sentiment Classification

Classify each review using both star rating and keyword scan:

```python
POSITIVE_KEYWORDS = ['great', 'excellent', 'amazing', 'love', 'perfect', 'fantastic',
                     'wonderful', 'best', 'delicious', 'friendly', 'clean', 'fast']
NEGATIVE_KEYWORDS = ['slow', 'rude', 'bad', 'terrible', 'worst', 'dirty', 'cold',
                     'expensive', 'wait', 'overpriced', 'disappointing', 'awful']

def classify_sentiment(rating, text):
    text_lower = (text or '').lower()
    pos_hits = sum(1 for k in POSITIVE_KEYWORDS if k in text_lower)
    neg_hits = sum(1 for k in NEGATIVE_KEYWORDS if k in text_lower)

    if rating >= 4 and neg_hits == 0: return 'Positive'
    if rating <= 2: return 'Negative'
    if rating == 3:
        if pos_hits > neg_hits: return 'Positive'
        if neg_hits > pos_hits: return 'Negative'
        return 'Neutral'
    if rating == 4 and neg_hits > 0: return 'Neutral'
    return 'Positive'
```

---

## Honeymoon Effect Warning

If the property has been operating for fewer than 3 months (infer from date of oldest review),
add this note to the health score output:

> ⚠️ **Honeymoon Effect:** This property has fewer than 3 months of reviews. Early ratings tend
> to be inflated. Health score confidence is LOW — revisit after 6+ months of data.

---

## Topic Clustering

Group negative review keywords into these themes (check all reviews, count occurrences):

**Positive themes:** food quality, ambiance, service speed, staff friendliness, value for money, cleanliness, variety

**Negative themes:** slow service, rude/unhelpful staff, food quality issues, noise/crowding, overpriced, long wait times, stock/availability issues, hygiene concerns

A theme is "critical" if it appears in ≥ 20% of negative reviews.
A theme is "improving needed" if it appears in 10–19% of negative reviews.
