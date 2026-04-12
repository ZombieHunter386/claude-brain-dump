# Chicago Multifamily Pipeline — Data Sources Design

**Date:** 2026-04-11
**Status:** Complete — ready for implementation
**Parent spec:** `2026-04-09-chicago-multifamily-pipeline-design.md`

---

## Overview

This spec defines every data source the pipeline uses, what fields are pulled, how they're stored, and how new sources get added. The core principle: **fetch everything, store everything, score from local data.** API calls happen at fetch time; scoring, re-scoring, and analysis all run instantly against SQLite.

---

## Target Geography

Defined by street boundaries, not community areas:

- **North:** Irving Park Road (~4000 N)
- **East:** Lake Michigan
- **South:** Fullerton Avenue (~2400 N)
- **West:** Western Avenue (~2400 W)

This covers Lincoln Park, Lakeview, and portions of adjacent neighborhoods (North Center, Roscoe Village, parts of Uptown and Lincoln Square near the southern/eastern edges).

Stored in `config/geography.yaml` as a bounding polygon with vertex coordinates. The pipeline filters parcels by point-in-polygon against this boundary using geopandas. Changing the boundary = edit the YAML, re-run fetch for the expanded area.

---

## Pipeline Architecture

```
FETCH (run once per area, refresh on-demand)
  Pull all raw data for all parcels in target geography
  Store as-is in SQLite — no calculations, no scoring
  ~15-25 API calls, ~5-10 minutes for initial pull
      ↓
CONSOLIDATE (run after fetch)
  Find adjacent parcels with same owner
  Create consolidated parcel rows with combined raw data
      ↓
ANALYZE (standalone script, run per area)
  Pull development history (new construction + demo-rebuild permits)
  Match to pre-development parcel characteristics
  Correlation analysis → suggested initial weights
  Output: scoring.yaml with weights + analysis report
      ↓
SCORE (run anytime, instant, no API calls)
  One unified rating system using weights from scoring.yaml
  All derived calculations happen here (FAR ratio, tax increase %, hold duration, etc.)
  Every parcel + consolidated parcel gets a score
      ↓
OUTPUT
  Top N results (default 20, configurable)
      ↓
ENRICH (only for top-scored parcels ready for outreach)
  Contact info lookup — provider TBD in outreach spec brainstorm
  IL SOS LLC registered agent lookup (scrape)
```

---

## Data Storage Principle

All fetched data is stored in SQLite regardless of whether it's currently used in scoring. Scoring references stored columns via YAML config. This means:

- **Adding a new data source** = new fetch module + new DB columns. No changes to scoring code.
- **Using stored data in scoring** = add a YAML entry pointing at the column + a calculation if needed.
- **Removing data from scoring** = remove the YAML entry. Data stays in the DB.
- **Re-scoring** = instant, no API calls. Change weights or add signals, re-run the score step.

---

## Source 1: Cook County Assessor (Socrata API)

**Portal:** https://datacatalog.cookcountyil.gov
**API base:** `https://datacatalog.cookcountyil.gov/resource/{dataset-id}.json`
**Protocol:** Socrata SODA REST API — JSON responses
**Auth:** Free app token (register at https://dev.socrata.com)
**Rate limit:** 1,000 requests/hour with app token
**Pagination:** `$limit` (max 50,000 rows) + `$offset` parameters
**Query language:** SoQL — supports `$where`, `$select`, `$order`, `$group`, geographic functions
**Cost: Free**

### 1A. Parcel Universe

- **Dataset:** `nj4t-kc8j` (historical, 1999–present) or `pabr-t5kh` (current year only)
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/nj4t-kc8j.json`
- **Update frequency:** Monthly
- **What to pull:** Current year records for all PINs within target geography
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `pin` | string | 14-digit parcel ID (primary key) |
| `pin10` | string | 10-digit short PIN |
| `class` | string | Property classification code (203=2-6 unit, 211=7+ unit, 318=commercial, etc.) |
| `lat`, `lon` | float | Coordinates |
| `ward_num` | string | Chicago ward number |
| `zip_code` | string | ZIP code |
| `tax_tif_district_num` | string | TIF district number (null if not in TIF) |
| `tax_tif_district_name` | string | TIF district name |
| `township_code` | string | Township |
| `nbhd_code` | string | Assessor neighborhood code |

- **Pipeline use:** Primary parcel identification, geographic filtering, ward data, TIF overlay detection

### 1B. Parcel Addresses

- **Dataset:** `3723-97qp`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/3723-97qp.json`
- **Update frequency:** Updated periodically (last: December 2025)
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `pin` | string | FK to parcels |
| `prop_address_full` | string | Property street address |
| `prop_address_city_name` | string | City |
| `prop_address_state` | string | State |
| `prop_address_zipcode_1` | string | ZIP |
| `mail_address_name` | string | Owner/mailing name |
| `mail_address_full` | string | Mailing address |
| `mail_address_city_name` | string | Mailing city |
| `mail_address_state` | string | Mailing state |
| `mail_address_zipcode_1` | string | Mailing ZIP |
| `owner_address_name` | string | Owner name (may differ from mail name) |
| `owner_address_full` | string | Owner address |

- **Pipeline use:**
  - **Absentee owner detection:** Compare `prop_address_full` vs `mail_address_full` — different = absentee
  - **LLC detection:** Check `mail_address_name` for patterns: "LLC", "CORP", "INC", "TRUST", "LP", "PARTNERS"
  - **Consolidation grouping:** Group by `owner_address_name` + `mail_address_full` to find same-owner parcels

### 1C. Improvement Characteristics (Single & Multi-Family)

- **Dataset:** `x54s-btds`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/x54s-btds.json`
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `pin` | string | FK to parcels |
| `year` | string | Assessment year |
| `class` | string | Property classification |
| `char_land_sf` | float | Lot size in square feet |
| `char_bldg_sf` | float | Building square footage |
| `char_yrblt` | string | Year built |
| `char_cnst_qlty` | string | Construction quality (Average, Good, etc.) |
| `char_repair_cnd` | string | Repair condition |
| `cdu` | string | Condition/desirability/utility rating |
| `char_beds` | string | Number of bedrooms |
| `char_rooms` | string | Total rooms |
| `char_fbath` | string | Full bathrooms |
| `char_hbath` | string | Half bathrooms |
| `char_type_resd` | string | Building type (2 Story, Split Level, etc.) |
| `char_ext_wall` | string | Exterior wall material |
| `char_heat` | string | Heating type |
| `char_bsmt` | string | Basement present |
| `char_bsmt_fin` | string | Basement finished |
| `char_gar1_att` | string | Garage attached |
| `char_gar1_area` | string | Garage area |
| `char_use` | string | Current use description |
| `char_site` | string | Site desirability |
| `char_air` | string | AC type |
| `pin_is_multicard` | bool | Multi-building flag |
| `pin_num_cards` | int | Number of buildings on parcel |

- **Pipeline use:** Lot size, building SF (for FAR calculation), year built, building condition, building classification

### 1D. Assessed Values

- **Dataset:** `uzyt-m557`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/uzyt-m557.json`
- **Coverage:** 1999–present
- **What to pull:** Last 5 years for all PINs (for trend analysis)
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `pin` | string | FK to parcels |
| `year` | string | Assessment year |
| `mailed_bldg` | float | Initial assessed building value |
| `mailed_land` | float | Initial assessed land value |
| `mailed_tot` | float | Initial total assessed value |
| `certified_bldg` | float | Post-appeal certified building value |
| `certified_land` | float | Post-appeal certified land value |
| `certified_tot` | float | Post-appeal certified total value |
| `board_bldg` | float | Board of Review final building value |
| `board_land` | float | Board of Review final land value |
| `board_tot` | float | Board of Review final total value |
| `board_hie` | float | Homeowner improvement exemption |

- **Pipeline use:**
  - **Land/building value ratio:** `board_land / board_tot` — high ratio = building worth little relative to land = redevelopment candidate
  - **Tax increase trend:** Compare `board_tot` across years — significant increase = higher holding costs
  - **Estimated property taxes:** `board_tot × equalization_factor × local_tax_rate` (see Source 3)

### 1E. Parcel Sales

- **Dataset:** `wvhk-k5uv`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/wvhk-k5uv.json`
- **Coverage:** 1999–present
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `pin` | string | FK to parcels |
| `sale_date` | date | Date of sale |
| `sale_price` | float | Transaction price |
| `seller_name` | string | Seller |
| `buyer_name` | string | Buyer |
| `deed_type` | string | Type of deed (Warranty, QuitClaim, Deed in Trust, etc.) |
| `doc_no` | string | Document number |
| `is_multisale` | bool | Part of multi-property transaction |
| `num_parcels_sale` | int | Number of parcels in sale |
| `sale_filter_same_sale_within_365` | bool | Non-arm's-length flag |
| `sale_filter_less_than_10k` | bool | Nominal sale flag |
| `sale_filter_deed_type` | bool | Non-standard deed flag |

- **Pipeline use:**
  - **Hold duration:** Current date minus most recent `sale_date`
  - **Sale comps:** Query nearby PINs with recent arm's-length sales for $/SF comps (filter out non-arm's-length using `sale_filter_*` flags)
  - **Deed type:** Stored as-is — QuitClaim, Deed in Trust, etc. each carry different signals

### 1F. Appeals

- **Dataset:** `y282-6ig3`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/y282-6ig3.json`
- **Coverage:** 1999–present
- **Pipeline use:** Count of appeals per PIN. Frequent appeals may signal owner frustrated with rising assessment/taxes.
- **Store:** PIN, year, appeal outcome, assessed value change

### 1G. Tax-Exempt Parcels

- **Dataset:** `vgzx-68gb`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/vgzx-68gb.json`
- **Pipeline use:** Exclusion list. Tax-exempt parcels (churches, schools, government) are not acquisition targets.
- **Store:** PIN, exemption type

---

## Source 2: Chicago Data Portal (Socrata API)

**Portal:** https://data.cityofchicago.org
**API base:** `https://data.cityofchicago.org/resource/{dataset-id}.json`
**Protocol:** Same Socrata SODA API as Cook County
**Auth:** Same app token works
**Rate limit:** 1,000 requests/hour with app token
**Cost: Free**

### 2A. Zoning Districts (Current Boundaries)

- **Dataset:** `7cve-jgbp`
- **Endpoint:** `https://data.cityofchicago.org/resource/7cve-jgbp.json`
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `zone_class` | string | Zoning classification (RM-5, RT-4, B3-2, C1-3, etc.) |
| `the_geom` | geometry | MultiPolygon boundary |
| `pd_num` | string | Planned Development number (if PD overlay) |

- **Pipeline use:** Spatial join — each parcel gets assigned its `zone_class` based on lat/lng falling within a zoning polygon. This zone_class feeds into the zoning lookup table for FAR/density/height limits.
- **Processing:** Download all zoning polygons once. Spatial join in geopandas against parcel coordinates. Store resulting `zone_class` on each parcel row.

### 2B. Zoning Density Lookup Table (Built from Ordinance)

- **Not an API source** — this is a static reference table built manually from the Chicago Zoning Ordinance and [Second City Zoning](https://secondcityzoning.org).
- **Stored as:** `config/zoning_lookup.yaml` or a SQLite reference table
- **Fields:**

| Field | Description |
|---|---|
| `zone_class` | e.g., RS-3, RT-4, RM-5, B2-3, C1-5 |
| `max_far` | Maximum floor area ratio |
| `max_height_ft` | Maximum building height in feet |
| `max_density` | Maximum units per lot area (e.g., 1 unit per 1,000 SF) |
| `min_lot_area_per_unit` | Minimum lot area per dwelling unit |
| `setback_front_ft` | Required front setback |
| `setback_side_ft` | Required side setback |
| `setback_rear_ft` | Required rear setback |
| `allows_multifamily` | bool — does this zone allow multifamily by right? |

- **Pipeline use:** After spatial join assigns `zone_class` to each parcel, look up what's allowed. Compare actual built FAR (`char_bldg_sf / char_land_sf`) to `max_far` — big gap = underbuilt = development opportunity.
- **Maintenance:** Update when zoning ordinance changes (rare). Add new zone classes as you expand geography.

### 2C. Building Permits

- **Dataset:** `ydr8-5enu`
- **Endpoint:** `https://data.cityofchicago.org/resource/ydr8-5enu.json`
- **Coverage:** 2006–present, updated daily
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `permit_` | string | Permit number |
| `permit_type` | string | NEW CONSTRUCTION, RENOVATION/ALTERATION, WRECKING/DEMOLITION, etc. |
| `issue_date` | date | When issued |
| `street_number` | string | Address components |
| `street_direction` | string | |
| `street_name` | string | |
| `work_description` | string | Free text describing work |
| `reported_cost` | float | Project cost reported by applicant |
| `community_area` | string | Community area number |
| `ward` | string | Ward number |
| `latitude`, `longitude` | float | Coordinates |

- **Pipeline use:**
  - **Historical analysis script:** Identify parcels where new construction or demo-rebuild happened — these are the "positive examples" for weight calibration
  - **Deferred maintenance signal:** Cross-reference parcel age vs permit history — old building + zero permits in 10+ years = likely deferred maintenance
- **Note:** Match to parcels by address or lat/lng proximity (permits don't have PINs)

### 2D. Building Violations

- **Dataset:** `22u3-xenr`
- **Endpoint:** `https://data.cityofchicago.org/resource/22u3-xenr.json`
- **Coverage:** 2006–present
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `violation_date` | date | When violation issued |
| `violation_code` | string | Violation code |
| `violation_status` | string | OPEN, CLOSED |
| `violation_description` | string | Description |
| `inspection_category` | string | PERMIT, COMPLAINT, PERIODIC, etc. |
| `department_bureau` | string | CONSERVATION, ELEVATOR, etc. |
| `address` | string | Property address |
| `street_number`, `street_direction`, `street_name` | string | Address components |
| `property_group` | string | Groups violations by property |
| `latitude`, `longitude` | float | Coordinates |

- **Pipeline use:** Count of open violations per parcel, age of oldest open violation. CONSERVATION violations = building condition issues. Pattern of repeat violations = distressed/neglectful owner.
- **Match to parcels:** By address or lat/lng proximity

### 2E. Vacant and Abandoned Buildings (311 Reports)

- **Dataset:** `7nii-7srd`
- **Endpoint:** `https://data.cityofchicago.org/resource/7nii-7srd.json`
- **Coverage:** 2010–present
- **Pipeline use:** Vacancy flag. Reported vacant/abandoned = strong motivation signal.
- **Note:** Coverage may be sparse in LP/Lakeview — these are not high-vacancy neighborhoods. Include as a signal but don't rely on it. If the endpoint returns empty, try alternate dataset ID `gjy8-tm9a`.

### 2F. CTA 'L' Rail Stations

- **Dataset:** `3tzw-cg4m`
- **Endpoint:** `https://data.cityofchicago.org/resource/3tzw-cg4m.json`
- **Fields to store:**

| Field | Type | Description |
|---|---|---|
| `station_id` | string | Station identifier |
| `longname` | string | Station name |
| `lines` | string | Which L lines serve the station |
| `the_geom` | geometry | Point coordinates |

- **Pipeline use:** Calculate distance from each parcel to nearest L station. Store the distance in feet on the parcel record. Chicago TOD ordinance allows density bonuses within 1/4 mile (~1,320 ft) of rail stations. LP/Lakeview served by Brown, Red, and Purple lines.
- **Derived field stored on parcel:** `cta_nearest_station_name`, `cta_nearest_station_dist_ft`

---

## Source 3: Cook County Clerk — Tax Data

### 3A. Delinquent Property Tax File

- **Portal:** https://www.cookcountyclerkil.gov/property-taxes/delinquent-property-tax-search
- **Protocol:** Bulk CSV download (NOT an API)
- **Update frequency:** Monthly
- **Coverage:** 20-year delinquent property history
- **Cost: Free**

**Access approach:**
- Download the full delinquent property CSV monthly
- Filter to PINs in target geography
- Store delinquency flag + years delinquent per PIN

**Pipeline use:** Tax delinquency is a primary motivation signal. Duration of delinquency (years) adds weight.

### 3B. Equalization Factor and Tax Rates

- **Source:** Published annually by Cook County Clerk
- **Protocol:** Manual lookup — published as PDF/press release each year
- **Cost: Free**

**What's needed:**
- **Cook County equalization factor** (also called "equalizer" or "multiplier") — applied to assessed value to get equalized assessed value (EAV). Changes annually.
- **Local composite tax rate** — varies by tax code. Parcels in different tax districts have different rates.

**Property tax formula:**
```
Assessed Value (board_tot)
× Equalization Factor
= Equalized Assessed Value (EAV)

EAV × Local Tax Rate = Estimated Annual Property Tax
```

**Storage approach:**
- Store equalization factor in `config/tax_rates.yaml` — update annually
- Store local tax rates by tax code (from Parcel Universe `tax_code` field) — a reference table
- Calculate estimated taxes at score time, not fetch time

---

## Source 4: Illinois Secretary of State — LLC Lookup

- **Portal:** https://apps.ilsos.gov/businessentitysearch/
- **Protocol:** Web search form only — NO API
- **Access method:** Scrape, one entity at a time
- **Cost: Free**

**What's available per entity:**
- LLC/Corp status (Active, Inactive, Dissolved)
- File number and date of incorporation
- Registered agent name and address
- Principal office address
- Date of last annual report

**Access approach:**
- **Enrichment stage only** — run for top-scored parcels where owner is identified as LLC
- ~20-50 lookups per scoring run
- Rate limit: conservative, ~1 request per 3-5 seconds
- Search by entity name from `mail_address_name`

**Concerns:**
- **Fragility:** HTML structure changes break the scraper
- **Terms:** Prohibit bulk downloads, but 20-50 individual lookups is not bulk
- **CAPTCHAs:** Could be added at any time
- **Mitigation:** Low volume makes manual fallback feasible if scraping breaks

**Fields to store:**

| Field | Description |
|---|---|
| `entity_name` | LLC name as registered |
| `status` | Active, Inactive, Dissolved |
| `file_number` | SOS file number |
| `date_incorporated` | Date formed |
| `registered_agent_name` | Person behind the LLC |
| `registered_agent_address` | Agent's address |
| `principal_office_address` | Office address |
| `last_annual_report_date` | Recency of compliance |

---

## Source 5: Contact Info Enrichment

- **Provider:** TBD — to be decided during outreach spec brainstorm
- **Purpose:** Phone numbers and email addresses for property owners identified by the scoring system
- **When it runs:** Enrichment stage only, for top-scored parcels ready for outreach
- **Expected volume:** ~20-50 lookups per scoring run
- **Expected cost:** ~$0.10-0.15/record if using BatchSkipTracing, but provider may change

This source will be fully specified in the outreach design spec.

---

## Historical Analysis Script

**Purpose:** Set initial scoring weights by analyzing what characteristics predicted actual development in the target geography.

**Data used (all from SQLite, no additional API calls):**

1. **Positive examples:** Building Permits (Source 2C) where `permit_type` = `PERMIT - NEW CONSTRUCTION` or `PERMIT - WRECKING/DEMOLITION` (demo-rebuild) within target geography
2. **Pre-development characteristics:** For each positive example, look up the parcel's data from the assessment year before the permit was filed — lot size, building SF, zoning, assessed values, owner type, hold duration, condition, building classification, etc.
3. **Negative examples:** All other parcels in the same geography and time period that were NOT developed

**Analysis approach:**
- Compare distributions of each characteristic between developed vs. undeveloped parcels
- For continuous variables (lot size, hold duration, land/building ratio): compare means, medians, distributions
- For categorical variables (zoning class, LLC vs. individual, absentee vs. resident): compare rates
- Logistic regression to produce weight coefficients — "lot size has 2x the predictive power of hold duration"
- Output sample sizes and confidence intervals so you know which signals are statistically meaningful

**Output:**
- `config/scoring.yaml` with suggested initial weights per signal
- Analysis report showing the underlying data (stored as markdown or displayed in UI)

**Geography:** Uses the same target geography config as the pipeline. When you expand to a new area, re-run the analysis for that area's development history.

**Re-runnable:** Can be re-run anytime — after adding new data sources, after expanding geography, or after enough feedback waves to include your own outreach outcomes as additional signal.

---

## Fetch Module Architecture

Each data source is its own Python module with a standard interface:

```
sources/
  assessor_parcels.py        # Source 1A — Parcel Universe
  assessor_addresses.py      # Source 1B — Parcel Addresses
  assessor_characteristics.py # Source 1C — Improvement Characteristics
  assessor_values.py         # Source 1D — Assessed Values
  assessor_sales.py          # Source 1E — Parcel Sales
  assessor_appeals.py        # Source 1F — Appeals
  assessor_exempt.py         # Source 1G — Tax-Exempt Parcels
  cdp_zoning.py              # Source 2A — Zoning Districts
  cdp_permits.py             # Source 2C — Building Permits
  cdp_violations.py          # Source 2D — Building Violations
  cdp_vacant.py              # Source 2E — Vacant Buildings
  cdp_cta_stations.py        # Source 2F — CTA Stations
  clerk_delinquent.py        # Source 3A — Delinquent Tax File
  sos_llc.py                 # Source 4 — IL SOS LLC Lookup (enrichment)
```

**Standard module interface:**

Each module implements:
- `fetch(geography_config, db_path)` — pull data from API/file, store raw in SQLite
- `get_source_name()` — returns human-readable source name
- `get_dataset_id()` — returns Socrata dataset ID (or null for non-Socrata sources)

**Pipeline runner** (`fetch_all.py`):
- Reads `config/geography.yaml` for target boundary
- Calls each module's `fetch()` in sequence
- Handles Socrata pagination automatically (shared utility)
- Logs fetch time, row counts, errors per source
- Skips sources that were recently fetched (configurable staleness threshold)

**Adding a new data source:**
1. Create a new module in `sources/` following the interface
2. Add any new columns to the SQLite schema
3. Register the module in the pipeline runner
4. Fetched data is now in SQLite — available for scoring via YAML config

---

## Shared Socrata Utilities

Since most sources use the same Socrata API pattern, a shared utility handles:

- **Authentication:** App token management
- **Pagination:** Automatic `$offset` iteration when results exceed `$limit`
- **Rate limiting:** Sleep/backoff when approaching 1,000 req/hr
- **Retry:** Exponential backoff on 429/500 errors
- **Geographic filtering:** Shared `$where` clause builder for target geography (lat/lng bounding box as a first pass, precise polygon filtering in Python after fetch)
- **Caching:** Track last fetch timestamp per source per geography — skip if data is fresh enough

---

## Data Volume Estimates

| Source | Rows (LP+LV+adjacent, ~20-25k parcels) | Est. Size |
|---|---|---|
| Parcel Universe | ~25k | ~8 MB |
| Parcel Addresses | ~25k | ~5 MB |
| Improvement Characteristics | ~25k (some multi-card) | ~10 MB |
| Assessed Values (5 years) | ~125k | ~25 MB |
| Parcel Sales (1999-present) | ~50-80k | ~15 MB |
| Appeals | variable | ~5 MB |
| Tax-Exempt Parcels | small subset | ~1 MB |
| Zoning Districts | ~500 polygons | ~5 MB |
| Building Permits (2006-present) | ~20-40k | ~10 MB |
| Building Violations | ~30-60k | ~12 MB |
| Vacant Buildings | sparse | ~1 MB |
| CTA Stations | ~30 stations | <1 MB |
| Delinquent Tax File (filtered) | small subset | ~2 MB |
| **Total** | | **~100-150 MB** |

SQLite handles this trivially. Full fetch takes ~5-10 minutes. Scoring runs in under 30 seconds.

---

## Rate Limit and Cost Summary

| Source | Protocol | Rate Limit | Cost |
|---|---|---|---|
| Cook County Assessor | Socrata REST | 1,000 req/hr with token | Free |
| Chicago Data Portal | Socrata REST | 1,000 req/hr with token | Free |
| Cook County Clerk (delinquent taxes) | Bulk CSV download | N/A | Free |
| Cook County Clerk (eq factor / rates) | Manual lookup | N/A | Free |
| IL Secretary of State | Web scrape | ~1 req/3-5 sec (conservative) | Free |
| Contact enrichment | TBD | TBD | TBD (see outreach spec) |

**Total pipeline cost per run: Free** (enrichment cost TBD in outreach spec)
