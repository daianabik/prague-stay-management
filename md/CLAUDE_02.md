# CLAUDE.md — 02 Data Preparation

> Place this file in the project root. Claude Code reads it automatically on startup.
> Run `01_data_understanding_eda.ipynb` first — this notebook depends on its outputs.

---

## What this notebook is about

This notebook takes the raw data identified in the EDA and turns it into clean, analysis-ready tables. By the end, everything is loaded into a local DuckDB database and validated for referential integrity.

Four things happen in sequence:

1. Clean and enrich `listings` → `listings_enriched.csv`
2. Clean `calendar` → `calendar_clean.csv`
3. Generate synthetic `bookings.csv` and `expenses.csv`
4. Load all five tables into DuckDB and validate foreign keys

---

## Project folder structure

```
project-root/
├── CLAUDE.md                               ← you are here
├── data/
│   ├── raw/
│   │   ├── listings.csv                    ← unzipped by notebook 01
│   │   ├── calendar.csv                    ← unzipped by notebook 01
│   │   └── metro_stations.csv
│   ├── processed/
│   │   ├── listings_enriched.csv           ← output of Section 1
│   │   └── calendar_clean.csv              ← output of Section 2
│   └── synthetic/
│       ├── bookings.csv                    ← output of Section 3.1
│       └── expenses.csv                    ← output of Section 3.2
├── database/
│   └── prague_stay.duckdb                  ← output of Section 4
└── notebooks/
    ├── 01_data_understanding_eda.ipynb
    └── 02_data_preparation.ipynb           ← this notebook
```

> All directories are created automatically by the notebook (`os.makedirs(..., exist_ok=True)`). Do not create them manually.

---

## How to run

Install dependencies (in addition to what notebook 01 already requires):

```bash
pip install pandas numpy duckdb
```

All sections are idempotent — if an output file already exists, the section is skipped. To regenerate everything from scratch, delete the contents of `data/processed/`, `data/synthetic/`, and `database/`.

---

## What each section covers

| Section | Description |
|---------|-------------|
| **0. Imports** | Libraries, random seeds (`seed=42`), path constants, directory creation |
| **1. Listings** | Column selection (79 → 22), price parsing, outlier removal, three-level imputation, column cleaning, Haversine metro distance |
| **2. Calendar** | Drop 100 % NaN columns, parse dates, derive `occupied`, `month`, `season` |
| **3.1 bookings.csv** | Run-length encode consecutive `available=f` days into bookings, with seasonal cancel rates and truncated-normal guest counts |
| **3.2 expenses.csv** | Generate cleaning and maintenance costs for confirmed bookings only |
| **4. DuckDB** | Load all five tables, cast types explicitly, validate all foreign keys |
| **5. Summary** | Table of every transformation and its result |

---

## Data paths and constants

```python
RAW       = '../data/raw/'
PROCESSED = '../data/processed/'
SYNTHETIC = '../data/synthetic/'
DB_PATH   = '../database/prague_stay.duckdb'
```

All paths are relative to the `notebooks/` folder. Do not use absolute paths.

---

## Section 1 — Listings: what was done and why

**Column selection:** 79 columns reduced to 22. Dropped columns fall into three categories:
- 100 % NaN: `license`, `neighbourhood_group_cleansed`, `calendar_updated`
- ~59 % NaN and not needed for KPIs: `neighborhood_overview`, `neighbourhood`
- Redundant or out of scope: all remaining host detail, scrape metadata, URL fields

**Price parsing:** `listings.price` is a string like `$2,272.00`. The `$` is an Inside Airbnb export artefact — confirmed by the median (~2 152 CZK), which matches real Prague market rates. Strip `$` and `,`, convert to float, store as `price_czk`.

**Outlier removal:**
- Upper bound: 99th percentile (~21 823 CZK). Values above are set to NaN before imputation.
- Lower bound: `Entire home/apt` listings priced below 400 CZK are treated as erroneous (likely blocking prices) and set to NaN.

**Three-level price imputation** (applied in order, stopping when a value is found):
1. Median by `neighbourhood_cleansed` + `room_type` + `accommodates`
2. Median by `neighbourhood_cleansed` + `room_type`
3. Global median

After imputation: 0 NaN in `price_czk`. An assertion enforces this.

**Other column cleaning:**
- `bedrooms`: imputed by median within `room_type`
- `beds`: imputed by median within `accommodates`
- `review_scores_rating`: filled with global median
- `host_since`: parsed to datetime
- `host_is_superhost`, `instant_bookable`: mapped from `'t'`/`'f'` strings to booleans

**Haversine distance:** Vectorised implementation using NumPy broadcasting — computes all pairwise distances between listings and metro stations in one pass (no Python loop). Adds four columns: `nearest_metro_id`, `nearest_metro_name`, `nearest_metro_line`, `metro_distance_m`. Also adds `near_metro` (boolean, threshold ≤ 500 m).

---

## Section 2 — Calendar: what was done and why

Dropped `price` and `adjusted_price` — both are 100 % NaN (confirmed in EDA). The calendar is only used for availability.

New columns derived:
- `occupied` — integer (0/1), True when `available == 'f'`
- `month` — integer month number
- `season` — string (`zima`/`jaro`/`léto`/`podzim`)

---

## Section 3 — Synthetic data: methodology

### bookings.csv

Bookings are generated by run-length encoding consecutive `available=f` days in the raw calendar. Each uninterrupted block becomes one booking.

Design decisions:
- `length_of_stay` is capped at 30 days — the project scope is short-term rentals only
- `guest_count` follows a truncated normal distribution; the mode scales with capacity (100 % for studios, 75 % for small apartments, 55 % for large ones), reflecting the EDA finding that larger apartments are not necessarily fully occupied
- `price_per_night` adds ±15 % noise around the cleaned price from `listings_enriched.csv`
- `status` (Confirmed/Cancelled) uses seasonal cancel rates derived from EDA occupancy patterns:
  - Winter and November: 20 %
  - Summer (June–August): 10 %
  - Shoulder months: 13 %
- `booking_created_at` is set to 1–60 days before `check_in`

A validation assertion confirms that `guest_count` never exceeds `accommodates` for any listing.

### expenses.csv

Generated only for `status = Confirmed` bookings.

- `cleaning_cost`: non-linear scale by number of bedrooms, with ±15 % noise
  - Studio (0 bedrooms): 600 CZK base
  - 1 bedroom: 900 CZK
  - 2 bedrooms: 1 400 CZK
  - 3 bedrooms: 2 000 CZK
  - 4+ bedrooms: 2 600 CZK + 400 CZK per additional bedroom
- `maintenance_cost`: random event, 5 % probability, 500–3 000 CZK when it occurs

---

## Section 4 — DuckDB schema

```
metro_stations ──► listings_enriched ──► bookings ──► expenses
                         │
                         └──────────────► calendar
```

| Table | Data type | PK | FK |
|-------|-----------|----|----|
| `metro_stations` | Real | `station_id` | — |
| `listings` | Real | `id` | `nearest_metro_id → metro_stations.station_id` |
| `calendar` | Real | `listing_id + date` | `listing_id → listings.id` |
| `bookings` | Synthetic | `booking_id` | `listing_id → listings.id` |
| `expenses` | Synthetic | `expense_id` | `booking_id → bookings.booking_id` |

All five foreign key constraints are validated after loading. The notebook asserts 0 violations before closing the connection.

Tables are dropped and recreated on each run (`DROP TABLE IF EXISTS`), so the notebook is fully idempotent.

---

## Conventions used in this project

- `RAW`, `PROCESSED`, `SYNTHETIC`, `DB_PATH` — use these constants, never hardcode paths
- Random seeds are fixed: `random.seed(42)`, `np.random.seed(42)` — outputs are reproducible
- Do not use `listings['price']` for any calculation — it is a raw string. Use `listings['price_czk']`
- Do not use `calendar['price']` or `calendar['adjusted_price']` — both are 100 % NaN
- DuckDB types are cast explicitly in `CREATE TABLE ... AS SELECT` — do not rely on auto-inference for downstream queries

---

## What comes next

All cleaned and synthetic tables are now in `database/prague_stay.duckdb`. The next notebook (`03_kpi_analysis.ipynb`) queries this database to compute:

- **KPI #1** — RevPAR by metro proximity (`near_metro`, `estimated_revenue_l365d`)
- **KPI #2** — Vacancy Loss Rate by season (`calendar.occupied`, `calendar.season`)
- **KPI #3** — Capacity Utilization for large apartments (`bookings.guest_count`, `listings.accommodates`)
