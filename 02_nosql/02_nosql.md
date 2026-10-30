# Module 2 — NoSQL (MongoDB): daily offers of three websites
**Owner:** Aya Elgarwany · **Sites:** XPRS (JSON), Dream 2000 (JSON), Jumia Egypt (HTML) · **Categories:** phone, tv

> Fill each step with the command you ran, its output and a screenshot (`images/`).

## Polite scraping (all three scrapers, already in `scripts/common.py`)
One request per second on XPRS and Dream 2000, one every 3-5 seconds on Jumia · HTTP 403 = stop (never work around a block) · User-Agent with the bot name and the contact in `SCRAPER_CONTACT` (`.env`) · no search or filter URLs · raw pages kept under `data/offers/raw/<date>/<site>/`.

## Files

| File | Task | Status |
|---|---|---|
| `scripts/common.py` | fetch, raw files, price parsing, offer document | **ready** |
| `notebooks/A1_overlap_test.ipynb` | A1 — products on 2+ sites per category (keep a category only with 20+) | **done 25-Oct**: phones 53-63, TVs 19-20, ACs 6 (dropped) |
| `notebooks/A2_dream2000_scraper.ipynb` | A2 — Shopify `products.json`, one offer per variant | **done 26-Oct** (Colab; `scripts/scraper_dream2000.py` is still a skeleton) |
| `scripts/mongo_load.py` | A3 — `catalog.offers` + `catalog.offers_quarantine` (reasons: no price / no stock state / out of scope), indexes, no duplicates on re-load | **done 26-Oct** on the real MongoDB (local, no auth); offline test `tests/test_mongo_load.py` (12 checks) |
| `notebooks/A5_xprs_scraper.ipynb` | A5 — Shopify `products.json`, same code as Dream 2000 (replaces 2B, which blocks bots) | **done 27-Oct** (Colab; `scripts/scraper_xprs.py` is still a skeleton) |
| `notebooks/A6_jumia_scraper.ipynb` | A6 — HTML category pages (`?page=N`) | **done 27-Oct** (Colab; `scripts/scraper_jumia.py` is still a skeleton) |
| `notebooks/A7_lists.ipynb` | A7 — brand / suffix / category lists | **done 27-Oct** |

## Steps

### 1. Scrapers (A2, A5, A6)
The scrapers run as Google Colab notebooks (`notebooks/`): open the notebook > **Runtime > Run all**. Each one saves `<site>_offers_<date>.json` in Google Drive (`depi_scraping/`).

| Notebook | Site | How | Rules |
|---|---|---|---|
| `A2_dream2000_scraper.ipynb` | Dream 2000 | Shopify `products.json`, one offer per variant | 1 request / second |
| `A5_xprs_scraper.ipynb` | XPRS | same code as Dream 2000 | 1 request / second |
| `A6_jumia_scraper.ipynb` | Jumia | HTML category pages `?page=N` | 4 seconds between pages; HTTP 403 = stop |

Output of the last cell, 29-Oct-2026:
```
dream2000: 583 offers | quarantine: 0
| Category | Products on 2+ sites | Decision |
|---|---|---|
| phone | 53-63 (23 on all three) | keep |
Offline test (no MongoDB needed): `pip install mongomock` then `python 02_nosql/tests/test_mongo_load.py`.
```
# run 1 and run 2 on 26-Oct-2026 (local MongoDB on my laptop)
py -3.13 "mongo_load (1).py" ..\..\dream2000_offers_2026-10-26.json
2026-10-26 23:53:21,542 INFO Loaded {'read': 557, 'offers': 557, 'quarantine': 0, 'refused': 0}; quarantine reasons {}

py -3.13 "mongo_load (1).py" ..\..\dream2000_offers_2026-10-26.json
2026-10-26 23:54:55,500 INFO Loaded {'read': 557, 'offers': 557, 'quarantine': 0, 'refused': 0}; quarantine reasons {}
# same counts on the second run = no duplicates
```

### 4. Daily scraping from 27 October (A4)
| Date | Dream 2000 | XPRS | Jumia |
|---|---|---|---|
| 2026-10-27 | 557 | 345 | 904 |
| 2026-10-28 | — | — | — |
| 2026-10-29 | 583 | 357 | 957 |

28-Oct was missed (prices can only be scraped on the day itself). Each day: run the three Colab notebooks, download the three json files to the laptop, then load each one:
```
py -3.13 "mongo_load (1).py" ..\..\dream2000_offers_2026-10-29.json
py -3.13 "mongo_load (1).py" ..\..\xprs_offers_2026-10-29.json
py -3.13 "mongo_load (1).py" ..\..\jumia_offers_2026-10-29.json
# offers loaded: 583 / 357 / 957 (same counts as Colab)
```
If the loader prints `ServerSelectionTimeoutError ... localhost:27017`, MongoDB is stopped: open **Services** > **MongoDB Server** > **Start**, then run the command again.

Run the three scrapers and the loader every day until the Airflow DAG takes over (N5). Keep a line per day here: date, offers per site.

### 5. Brand and suffix lists (A7)
Fill `05_etl/config/brand_aliases.csv`, `model_suffixes.csv` and `category_map.csv` from the first full scrape.

Done 27-Oct-2026 in Colab notebook `A7_lists.ipynb`, from the 27-Oct scrape (Dream 2000 557, XPRS 345, Jumia 904 offers).

| File | Rows | What it does | Result |
|---|---|---|---|
Count offers per site and category, average price per category and site, in-stock share; `mongoexport` to CSV in `data/`.

**Done 27-Oct-2026** in the MongoDB shell of Compass (`use catalog`), on all loaded days (26-Oct = Dream 2000 only, 27-Oct = three sites).

Total: `db.offers.countDocuments()` -> **2363** (1806 on 27-Oct + 557 on 26-Oct), `offers_quarantine` = 0.

Group used by the three queries:
```
{$group:{_id:{date:"$snapshot_date",site:"$site",category:"$category"}, ... }}
```
- offers: `offers:{$sum:1}`
- average price: `avg_price:{$avg:"$sale_price"}`
- in-stock share: `in_stock_pct:{$avg:{$cond:["$in_stock",100,0]}}`

| Date | Site | Category | Offers | Avg price (EGP) | In stock |
|---|---|---|---|---|---|
| 2026-10-26 | dream2000 | phone | 500 | 31,899 | 60.8% |
Screenshots: `images/` (A8 counts, average price, in-stock share, mongoexport).

## Deliverables
- [x] three scrapers · [x] `mongo_load.py` · [x] overlap test result · [x] daily snapshots · [x] lists in `05_etl/config/` · [x] queries + CSV · [x] `images/`

## Steps

### 1. Scrapers (A2, A5, A6)
The scrapers run as Google Colab notebooks (`notebooks/`): open the notebook > **Runtime > Run all**. Each one saves `<site>_offers_<date>.json` in Google Drive (`depi_scraping/`).

| Notebook | Site | How | Rules |
|---|---|---|---|
| `A2_dream2000_scraper.ipynb` | Dream 2000 | Shopify `products.json`, one offer per variant | 1 request / second |
| `A5_xprs_scraper.ipynb` | XPRS | same code as Dream 2000 | 1 request / second |
| `A6_jumia_scraper.ipynb` | Jumia | HTML category pages `?page=N` | 4 seconds between pages; HTTP 403 = stop |

Output of the last cell, 29-Oct-2026:
```
dream2000: 583 offers | quarantine: 0

| Category | Products on 2+ sites | Decision |
|---|---|---|
| phone | 53-63 (23 on all three) | keep |
`python 02_nosql/scripts/mongo_load.py data/offers/offers_*_<date>.json` — load the same files twice: the counts must not change.
Offline test (no MongoDB needed): `pip install mongomock` then `python 02_nosql/tests/test_mongo_load.py`.
```
# run 1 and run 2 on 26-Oct-2026 (local MongoDB on my laptop)
py -3.13 "mongo_load (1).py" ..\..\dream2000_offers_2026-10-26.json
2026-10-26 23:53:21,542 INFO Loaded {'read': 557, 'offers': 557, 'quarantine': 0, 'refused': 0}; quarantine reasons {}

py -3.13 "mongo_load (1).py" ..\..\dream2000_offers_2026-10-26.json
2026-10-26 23:54:55,500 INFO Loaded {'read': 557, 'offers': 557, 'quarantine': 0, 'refused': 0}; quarantine reasons {}
# same counts on the second run = no duplicates
```

### 4. Daily scraping from 27 October (A4)
| Date | Dream 2000 | XPRS | Jumia |
|---|---|---|---|
| 2026-10-27 | 557 |345| 904 |
| 2026-10-28 | — | — | — |
| 2026-10-29 | 583 | 357 | 957 |

28-Oct was missed (prices can only be scraped on the day itself). Each day: run the three Colab notebooks, download the three json files to the laptop, then load each one:
```
py -3.13 "mongo_load (1).py" ..\..\dream2000_offers_2026-10-29.json
py -3.13 "mongo_load (1).py" ..\..\xprs_offers_2026-10-29.json
py -3.13 "mongo_load (1).py" ..\..\jumia_offers_2026-10-29.json
# offers loaded: 583 / 357 / 957 (same counts as Colab)
```
If the loader prints `ServerSelectionTimeoutError ... localhost:27017`, MongoDB is stopped: open **Services** > **MongoDB Server** > **Start**, then run the command again.


Run the three scrapers and the loader every day until the Airflow DAG takes over (N5). Keep a line per day here: date, offers per site.

### 5. Brand and suffix lists (A7)
Fill `05_etl/config/brand_aliases.csv`, `model_suffixes.csv` and `category_map.csv` from the first full scrape.

Done 27-Oct-2026 in Colab notebook `A7_lists.ipynb`, from the 27-Oct scrape (Dream 2000 557, XPRS 345, Jumia 904 offers).

| File | Rows | What it does | Result |
|---|---|---|---|
Count offers per site and category, average price per category and site, in-stock share; `mongoexport` to CSV in `data/`.

**Done 27-Oct-2026** in the MongoDB shell of Compass (`use catalog`), on all loaded days (26-Oct = Dream 2000 only, 27-Oct = three sites).

Total: `db.offers.countDocuments()` -> **2363** (1806 on 27-Oct + 557 on 26-Oct), `offers_quarantine` = 0.

Group used by the three queries:
```
{$group:{_id:{date:"$snapshot_date",site:"$site",category:"$category"}, ... }}
```
- offers: `offers:{$sum:1}`
- average price: `avg_price:{$avg:"$sale_price"}`
- in-stock share: `in_stock_pct:{$avg:{$cond:["$in_stock",100,0]}}`

| Date | Site | Category | Offers | Avg price (EGP) | In stock |
|---|---|---|---|---|---|
| 2026-10-26 | dream2000 | phone | 500 | 31,899 | 60.8% |
Screenshots: `images/` (A8 counts, average price, in-stock share, mongoexport).

## Deliverables
- [ ] three scrapers · [ ] `mongo_load.py` · [ ] overlap test result · [ ] daily snapshots · [ ] lists in `05_etl/config/` · [ ] queries + CSV · [ ] `images/`
