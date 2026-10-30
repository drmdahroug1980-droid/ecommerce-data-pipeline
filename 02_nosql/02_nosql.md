# Module 2 — NoSQL (MongoDB): daily offers of three websites

**Owner:** Eng. Aya Elgarwany · **Sites:** XPRS (JSON), Dream 2000 (JSON), Jumia Egypt (HTML) · **Categories:** phone, tv

> Fill each step with the command you ran, its output and a screenshot (`images/`).

## Polite scraping (all three scrapers, already in `scripts/common.py`)
One request per second on XPRS and Dream 2000, one every 3-5 seconds on Jumia · HTTP 403 = stop (never work around a block) · User-Agent with the bot name and the contact in `SCRAPER_CONTACT` (`.env`) · no search or filter URLs · raw pages kept under `data/offers/raw/<date>/<site>/`.

## Files

| File | Task | Status |
|---|---|---|
| `scripts/common.py` | fetch, raw files, price parsing, offer document | **ready** |
| `scripts/scraper_dream2000.py` | A2 — Shopify `products.json`, one offer per variant | skeleton |
| `scripts/mongo_load.py` | A3 — `catalog.offers` + `catalog.offers_quarantine` (reasons: no price / no stock state / out of scope), indexes, no duplicates on re-load | **ready** — offline test `tests/test_mongo_load.py` (12 checks); run once on the real MongoDB |
| `scripts/scraper_xprs.py` | A5 — Shopify `products.json`, same code as Dream 2000 (replaces 2B, which blocks bots) | skeleton |
| `scripts/scraper_jumia.py` | A6 — HTML category pages (`?page=N`) | skeleton |
| `scripts/overlap_test.py` | A1 — products on 2+ sites per category (keep a category only with 20+) | **done 25-Oct**: phones 53-63, TVs 19-20 (recount after the first full scrape), ACs 6 (dropped) |

## Steps

### 1. Scrapers (A2, A5, A6)
`python 02_nosql/scripts/scraper_dream2000.py` (same for `scraper_xprs.py`, `scraper_jumia.py`). Each writes `data/offers/offers_<site>_<date>.json`.
```
# commands / output
```

### 2. Overlap test (A1)
Run the three scrapers with `--sample`, then `python 02_nosql/scripts/overlap_test.py data/offers/offers_*_<date>.json`. Paste the summary and the decision per category; check 20 rows of `overlap_report.csv` by eye.
```
# commands / output
```

### 3. Load into MongoDB (A3)
`python 02_nosql/scripts/mongo_load.py data/offers/offers_*_<date>.json` — load the same files twice: the counts must not change.
Offline test (no MongoDB needed): `pip install mongomock` then `python 02_nosql/tests/test_mongo_load.py`.
```
# commands / output
```

### 4. Daily scraping from 27 October (A4)
Run the three scrapers and the loader every day until the Airflow DAG takes over (N5). Keep a line per day here: date, offers per site.

### 5. Brand and suffix lists (A7)
Fill `05_etl/config/brand_aliases.csv`, `model_suffixes.csv` and `category_map.csv` from the first full scrape.

### 6. Queries and export (A8)
Count offers per site and category, average price per category and site, in-stock share; `mongoexport` to CSV in `data/`.
```
# commands / output
```

## Deliverables
- [ ] three scrapers · [ ] `mongo_load.py` · [ ] overlap test result · [ ] daily snapshots · [ ] lists in `05_etl/config/` · [ ] queries + CSV · [ ] `images/`
