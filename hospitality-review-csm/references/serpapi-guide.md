# SerpAPI Guide — Google Reviews Scraping

This file defines the exact API call sequence, pagination pattern, and known failure modes.
Read this before making any SerpAPI call.

---

## Required Config

The SerpAPI key must be stored at `<WORK_DIR>/config/serpapi-config.json`:

```json
{
  "api_key": "YOUR_SERPAPI_KEY_HERE"
}
```

Load it at runtime:
```python
import json, os
config_path = os.path.join(work_dir, "config", "serpapi-config.json")
with open(config_path) as f:
    api_key = json.load(f)["api_key"]
```

---

## Step 1 — Get Place ID via Knowledge Graph

**Always use this step first.** Do NOT attempt `engine: google_maps` with `type: search` — it
does not reliably return place IDs for Indian locations or smaller establishments.

```python
import requests

params = {
    "engine": "google",
    "q": business_name,          # e.g. "Dessertino Connaught Place Delhi"
    "api_key": api_key,
}
resp = requests.get("https://serpapi.com/search", params=params)
data = resp.json()
place_id = data.get("knowledge_graph", {}).get("place_id")

if not place_id:
    # Fallback: try local_results
    local = data.get("local_results", [])
    if local:
        place_id = local[0].get("place_id")

if not place_id:
    raise ValueError(f"Could not find place_id for: {business_name}")
```

---

## Step 2 — Fetch Reviews (Paginated)

Once you have `place_id`, fetch reviews in pages of up to 10:

```python
all_reviews = []
next_page_token = None
max_reviews = 100  # configurable

while len(all_reviews) < max_reviews:
    params = {
        "engine": "google_maps_reviews",
        "place_id": place_id,
        "api_key": api_key,
        "sort_by": "newestFirst",  # always sort newest first for trend analysis
    }
    if next_page_token:
        params["next_page_token"] = next_page_token

    resp = requests.get("https://serpapi.com/search", params=params)
    data = resp.json()

    reviews = data.get("reviews", [])
    if not reviews:
        break

    all_reviews.extend(reviews)
    next_page_token = data.get("serpapi_pagination", {}).get("next_page_token")
    if not next_page_token:
        break

print(f"Fetched {len(all_reviews)} reviews for place_id={place_id}")
```

---

## Review Object Schema

Each review from SerpAPI has this shape (fields may vary):

```json
{
  "user": { "name": "...", "reviews": 12, "photos": 3 },
  "rating": 4,
  "date": "a month ago",
  "iso_date": "2026-02-15T00:00:00.000Z",
  "snippet": "Great ambiance but slow service...",
  "likes": 2,
  "response": { "date": "3 weeks ago", "snippet": "Thank you for your feedback!" }
}
```

Normalise each review into a flat dict:
```python
def normalise_review(r, idx):
    return {
        "reviewer_id": f"Reviewer {chr(65 + idx % 26)}",  # anonymise
        "rating": r.get("rating"),
        "date_str": r.get("date"),
        "iso_date": r.get("iso_date"),
        "text": r.get("snippet", ""),
        "has_response": bool(r.get("response")),
        "response_text": (r.get("response") or {}).get("snippet"),
    }
```

---

## Multi-Store Pattern

Run the full scrape (Steps 1+2) once per store. Save each result separately:

```python
stores = [
    {"name": "Store Name A", "slug": "store-a"},
    {"name": "Store Name B", "slug": "store-b"},
]

for store in stores:
    try:
        place_id = get_place_id(store["name"], api_key)
        reviews  = fetch_reviews(place_id, api_key, max_reviews=100)
        out_path = os.path.join(work_dir, "data", f"store-{store['slug']}-data.json")
        with open(out_path, "w") as f:
            json.dump({"store": store, "reviews": reviews}, f, indent=2)
        print(f"Saved {len(reviews)} reviews for {store['name']}")
    except Exception as e:
        print(f"FAILED for {store['name']}: {e}")
        # Log and continue — don't abort the whole run
```

---

## Known Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| `place_id` not in `knowledge_graph` | Business name too vague | Add city/area to the query |
| 429 Too Many Requests | SerpAPI quota hit | Wait 60s or reduce `max_reviews` |
| Empty `reviews` list | CAPTCHA or geo-block | Fall back to user-paste |
| `iso_date` missing | Some older reviews have only `date` (relative string) | Parse relative dates with `dateparser` |
| Reviews in non-English | SerpAPI returns original language | Note it; don't force-translate in the analysis |

---

## Saving Raw Data

Always persist raw SerpAPI output before processing:

```python
raw_path = os.path.join(work_dir, "data", f"{slug}-raw-{today}.json")
with open(raw_path, "w", encoding="utf-8") as f:
    json.dump(all_reviews, f, indent=2, ensure_ascii=False)
```

This allows re-running analysis without re-scraping, which costs API credits.
