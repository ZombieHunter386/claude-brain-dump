# Chicago Off-Market Multifamily Pipeline — System Design

**Date:** 2026-04-09 (updated 2026-04-11)
**Status:** Architecture, data model, Review UI, outreach overview, and feedback loop overview complete. Data sources spec complete. Scoring system defined (no separate spec needed). One subsystem spec still required before coding (outreach) — see Spec Suite table below.

---

## Spec Suite — Full File Plan

This system requires multiple spec files before it is ready to hand off for implementation. Each file must be complete before coding begins. This file is the **master architecture spec** — all other specs are subsystem specs that plug into it.

| File | Status | What it covers |
|---|---|---|
| `2026-04-09-chicago-multifamily-pipeline-design.md` | **This file** — complete | System architecture, pipeline stages, data model, UI layout, outreach flow, feedback loop |
| `2026-04-11-pipeline-data-sources-design.md` | **Complete** | All data sources, API endpoints, field mappings, fetch architecture, historical analysis script, rate limits |
| ~~`pipeline-scoring-design.md`~~ | **Not needed** — scoring system fully defined in master spec (Section 2) + data sources spec | Signals come from data sources; weights from historical analysis; scoring math is weighted sum with normalization |
| `YYYY-MM-DD-pipeline-outreach-design.md` | Not started — needs brainstorm session | Message templates per channel (mail, email); sequence timing and scheduling logic; contact enrichment provider; outreach positioning; broker route workflow |

**Coding should not begin until all specs exist and are approved.**

---

## Goals

1. **Automate deal sourcing end-to-end** — screen LP/Lakeview parcels through configurable scoring, draft outreach, and surface qualified leads with minimal manual input. The pipeline does the work; you review and approve.
2. **Surface both as-of-right and entitlement plays** — identify sites that work under current zoning AND sites that could be worth significantly more after rezoning. Both are valid deal types.
3. **Understand why a site is good** — explainable scores, not just a number. Each site should show what it is today, what it could become, and why the owner might sell.
4. **Qualify and hand off leads** — act as a deal-sourcing SDR: find, qualify, do first outreach as a prospective buyer, then introduce qualified leads to developer partner for closing.
5. **Get smarter over time** — iterate scoring based on response rates and handoff outcomes. Wave notes capture qualitative learning; weight adjustments are manual and deliberate.
6. **Build toward doing your own deals** — the pipeline trains you as much as it trains itself. Build pattern recognition for what makes a good Chicago multifamily site, working toward your own entitlement plays.
7. **Laptop-first, portable** — no hosting required; SQLite file shareable by email with partner.

---

## Constraints & Decisions

- **Tooling:** Python pipeline + Flask web UI + SQLite — single language, single portable file, no hosting
- **Vibe coding:** Code will be rewritten by Claude as needed; reuse of existing CRE app is not a priority
- **Scoring parameters:** Exact signals and weights are TBD — initial weights set by historical analysis script, then adjusted manually after outreach waves. Scoring is fully configurable via YAML; changing a weight = edit the file, adding a new signal = edit code + add entry.
- **Free databases only for fetch** — paid enrichment only after scoring, for top-N parcels selected for outreach
- **No Street View** — replaced with "Open in Google Maps" link; avoids Google API key dependency
- **Geography is configurable** — defined as a street-bounded polygon in YAML config; can be narrowed or expanded without code changes. Initial target: Irving Park (N) to Fullerton (S) to Western (W) to Lake Michigan (E) — covers Lincoln Park, Lakeview, and portions of adjacent neighborhoods.
- **Outreach positioning:** All outreach is from you as a prospective buyer/developer exploring opportunities — not representing a third party. No broker's license; stay on the right side of bird-dogging vs. brokerage.
- **Data sources are pluggable** — the pipeline must make it easy to add, remove, or swap data sources without restructuring. All data pulled from any source is stored in SQLite regardless of whether it's currently used in scoring. Scoring references which stored fields to use via YAML config. This means you can pull data now, decide later whether it factors into scoring, and never lose anything you've already fetched.

---

## Architecture Overview

Five layers, each with a single responsibility:

```
CONFIG (YAML)
  Geography polygon · scoring weights · zoning lookup · outreach templates
  └─ Initial weights from historical analysis, adjusted manually after each wave
        ↓
PIPELINE (Python scripts — run on demand)
  Fetch → Consolidate → Analyze → Score → Enrich
  └─ Fetch and Score are decoupled: fetch hits APIs, score runs locally
        ↓
DATABASE (SQLite — single portable file)
  All parcels in target geography with all fetched data
  └─ Email .sqlite to partner; re-score without re-fetching
        ↓
REVIEW UI (Flask — localhost:5000)
  Map + ranked list + score breakdown + outreach drafting + feedback report
        ↓
OUTREACH (APIs — gated by human approval)
  Gmail API · Lob API · phone (manual) · in-person (manual)
```

**Key principles:**
- **Fetch once, score forever** — API calls happen at fetch time (~5-10 min). Scoring runs instantly against local SQLite data. Re-score as many times as you want with different weights.
- **Data collection is separate from scoring logic** — the pipeline stores all fetched data in SQLite regardless of whether it's used in scoring. Scoring pulls from stored columns based on YAML config. Adding a new data source = new fetch module + new DB columns. Using that data in scoring = new YAML entry. Removing data from scoring doesn't delete it.
- Pipeline and UI are decoupled — pipeline runs as a script, not triggered from the browser
- Config drives parameter tuning; new signals require a code change + new config entry
- Nothing sends without explicit approval in the UI

---

## Section 1 — Config (YAML)

Four config areas:

**`config/geography.yaml`** — search area polygon defined by street boundaries (Irving Park / Lake Michigan / Fullerton / Western). Change this to expand/narrow search without touching code.

**`config/scoring.yaml`** — all scoring weights and the top-N threshold. Versioned so each scoring run records which version was used. Changing weights = edit this file. Adding a new signal = edit code + add entry here. Initial weights set by the historical analysis script.

**`config/zoning_lookup.yaml`** — static reference mapping zone_class → max FAR, max height, max density, setbacks, multifamily-by-right flag. Built from Chicago Zoning Ordinance / Second City Zoning.

**`config/tax_rates.yaml`** — Cook County equalization factor (updated annually) and local tax rates by tax code. Used to estimate property taxes from assessed values.

Outreach message templates stored as separate files (`config/templates/`) referenced from scoring config.

---

## Section 2 — Pipeline

### Fetch (run once per area, refresh on-demand)
- **What it does:** Pulls ALL raw data for ALL parcels in the target geography from all configured data sources. Stores as-is in SQLite — no calculations, no scoring.
- **Sources:** Cook County Assessor (7 datasets), Chicago Data Portal (5 datasets), Cook County Clerk (delinquent tax bulk CSV). See `2026-04-11-pipeline-data-sources-design.md` for full details.
- **Duration:** ~5-10 minutes for initial pull (~20-25 API calls)
- **Data volume:** ~100-150 MB for the target geography (~20-25k parcels)
- **Refresh:** On-demand. Data can be weeks/months old — that's acceptable.

### Consolidate (run after fetch)
- **What it does:** Finds adjacent parcels with the same owner and creates consolidated parcel rows.
- **Logic:** Group parcels by `owner_address_name` + `mail_address_full`. For groups with 2+ parcels, check proximity (lat/lng within ~50ft or sequential Cook County block/lot numbers). Create a new consolidated row with combined lot size and combined raw data.
- **Output:** Consolidated rows added to the database alongside individual parcels.

### Analyze (standalone script, run per area)
- **What it does:** Sets initial scoring weights by analyzing what parcel characteristics predicted actual development in the target geography.
- **Logic:** Identifies parcels where new construction or demo-rebuild permits were filed (2006-present). Looks up each parcel's characteristics from the year before the permit. Compares developed vs. undeveloped parcels across all stored signals. Runs correlation analysis / logistic regression.
- **Output:** `config/scoring.yaml` with suggested initial weights + analysis report.
- **Re-runnable:** After expanding geography, adding data sources, or accumulating enough outreach feedback.

### Score (run anytime, instant, no API calls)
- **What it does:** Applies a single unified scoring system to every parcel + consolidated parcel. All derived calculations happen here (FAR ratio, tax increase %, hold duration, estimated taxes, CTA distance, etc.).
- **Logic:** Simple weighted sum. Each signal is `normalized_value × weight`, summed and normalized to 0-100. Continuous signals (hold duration, FAR gap, etc.) are normalized to 0-1 using ranges determined by the historical analysis. Binary signals (is_absentee, tax_delinquent, etc.) are 0 or 1 — same math, no special handling. Weights come from `config/scoring.yaml`.
- **Everything gets scored** — no exclusions before scoring. All parcels in the target geography receive a score.
- **Output:** Top N results (default 20, configurable) with **filterable exclusions** — e.g., hide tax-exempt, city-owned, or specific property classes from the top list without removing them from the database. Change filters anytime.
- **Consolidation scoring:** Each individual parcel is scored on its own data. Consolidated groups are also scored as a single entity using combined lot size and combined data. Both appear in results independently.
- **Key property:** Scoring is decoupled from fetching. Change weights, add signals, re-score — instant, free, no API calls.

### Enrich (only for top-scored parcels ready for outreach)
- **What it does:** Looks up contact info for top-scored parcels.
- **Sources:** Contact enrichment provider (TBD in outreach spec) + IL SOS LLC registered agent lookup (scrape, ~20-50 lookups).
- **When:** Only after you've reviewed scores and selected parcels for outreach.

---

## Section 3 — Database (SQLite)

### Schema

**`parcels` table** — PIN is unique key (Cook County format: `14-21-101-001-0000`)

| Column group | Columns |
|---|---|
| Identity | pin, address, lat, lng, ward_num, zip_code |
| Owner | owner_name, owner_address, mail_name, mail_address, is_absentee, is_llc |
| Building | property_class, lot_size_sf, building_sf, year_built, condition, building_classification, zone_class |
| Values | assessed_land, assessed_building, assessed_total, land_building_ratio, estimated_annual_tax, tax_increase_pct_1yr, tax_increase_pct_5yr |
| Sales | last_sale_date, last_sale_price, hold_duration_years, deed_type |
| Signals | tax_delinquent, delinquency_years, open_violations_count, oldest_violation_age_days, appeal_count, has_vacancy_report, years_since_last_permit |
| Zoning | max_far, built_far, far_gap, allows_multifamily_by_right, tif_district, cta_nearest_station, cta_distance_ft |
| Scoring | score, score_version, consolidation_group_id |
| Status | stage, first_seen_date, last_updated_date, last_fetched_date |

Raw data from each source is also stored in source-specific tables (see data sources spec). The parcels table contains both raw fields and derived/calculated fields populated at score time.

Stage values: `scored` / `outreach` / `responded` / `introduced` / `dead`

All parcels in the target geography are stored. The top-N threshold determines what surfaces in the UI, but everything is browsable.

**`consolidation_groups` table** — synthetic group_id for adjacent same-owner parcels

| Column | Notes |
|---|---|
| group_id | auto-increment PK |
| pins | JSON array of PINs |
| combined_lot_size_sf | sum of constituent lots |
| owner_name | shared owner |
| detected_date | date flagged |

**`contacts` table** — one row per contact per parcel or consolidation group

| Column | Notes |
|---|---|
| contact_id | auto-increment PK |
| pin | FK to parcels (null if consolidation group) |
| consolidation_group_id | FK to consolidation_groups (null if single parcel) |
| name | owner name, broker name, etc. |
| phone | from skip trace |
| email | from skip trace |
| mailing_address | from assessor or skip trace |
| role | `owner` / `broker` / `agent` |
| source | `assessor` / `skip_trace` |

**`outreach` table** — one row per outreach attempt per site per channel

| Column | Notes |
|---|---|
| outreach_id | auto-increment PK |
| pin | FK to parcels (or consolidation_group_id) |
| contact_id | FK to contacts |
| channel | `mail` / `email` / `phone` / `in_person` |
| sent_date | when sent or attempted |
| response_date | null until response received |
| response_type | `interested` / `not_interested` / `no_response` / `referred_broker` |
| handed_off | bool — set when lead is introduced to developer partner |
| handed_off_date | date introduced |
| notes | free text |

**`waves` table** — one row per outreach wave, for tracking learning over time

| Column | Notes |
|---|---|
| wave_id | auto-increment PK |
| start_date | when wave was kicked off |
| end_date | when wave was closed out |
| parcels_contacted | count |
| responses_received | count |
| leads_introduced | count |
| notes | free text — qualitative learning from this wave |
| config_version | which scoring.yaml version was active |

### Key design decisions

- **Store everything in target geography** — all parcels, all data from all sources, regardless of whether it's currently used in scoring. Allows re-scoring with new weights or new signals without re-fetching API data.
- **Upsert on PIN** — `INSERT ... ON CONFLICT(pin) DO UPDATE SET ...` — new API pulls refresh existing rows, add new parcels, never touch `first_seen_date`
- **Spatial logic in Python** — geopandas handles geographic filtering, zoning spatial joins, CTA distance calculations, and adjacency checks before writing to SQLite; SQLite never does geometry math
- **Raw + derived separation** — source-specific tables store raw API data as-is. The parcels table contains derived/calculated fields populated at score time. This means raw data is never lost even if scoring logic changes.
- **Scale:** SQLite is sufficient. Target geography (~20-25k parcels, ~100-150 MB) is trivial. All of Chicago (~600k) is manageable. Upgrade to PostgreSQL + PostGIS only if expanding to all of Cook County with complex spatial queries.

---

## Section 4 — Review UI (Flask)

**Runs at:** `localhost:5000`

### Layout
Three-column interface:
- **Left:** Ranked list — shows top N parcels (default 20, adjustable) sorted by score. Score badge, key signal tags, outreach status per row. Ability to browse beyond top N. Batch "Draft Outreach" button at bottom. Filter by stage (scored / outreach / responded / introduced).
- **Center:** Leaflet.js map (OpenStreetMap tiles, no API key). Color-coded pins: top-N (green), consolidated (purple), outreach sent (blue), all others (gray). Layer toggles for score tiers and outreach status.
- **Right:** Detail panel for selected site — property facts (PIN, address, lot size, ward, building classification), zoning context (current zone class, what's allowed by-right vs. what would require rezoning, built FAR vs. max FAR), score breakdown showing which signals fired and by how much (config-driven, not hardcoded), estimated annual property taxes, hold duration, "Open in Google Maps" link, outreach action buttons (Draft / Skip / Notes). Score version stamp shown so you know if site was scored under old weights.

**Feedback Report:** Lives as a section in the UI. Shows per-signal stats (response rates, handoff rates) from completed outreach waves and wave notes history. See Section 6.

### Score breakdown
Single unified score (0-100) blending development potential and motivation signals. Pulls signal names and weights dynamically from `config/scoring.yaml`. If you add a new signal or change a weight, the breakdown reflects it automatically. Each parcel records `score_version` so old scores remain interpretable after config changes. The breakdown shows each signal's contribution so you can see *why* a parcel scored the way it did.

---

## Section 5 — Outreach Draft & Approval Flow

**Status:** Overview captured — needs dedicated brainstorm session for full spec (`YYYY-MM-DD-pipeline-outreach-design.md`)

### Overview
After you select sites from the ranked list, the system drafts personalized outreach messages for each site, you review and approve them, then they send automatically via the appropriate channel.

### Channels
| Channel | Automation | API | Notes |
|---|---|---|---|
| Physical mail | Automated | Lob API | Handwritten-style font, ~$1.20/letter |
| Email | Automated | Gmail API | OAuth, sent from your Gmail account |
| Phone | Manual | — | UI surfaces the number and a suggested script |
| In-person | Manual | — | UI surfaces address and talking points |
| Broker route | Manual | — | If property is listed, flag in UI and pass listing info to developer partner. You don't contact the broker directly (license constraint). |

### Outreach Sequence
```
Week 1  → Physical mail (letter)
Week 3  → Follow-up mail or postcard
Week 5  → Phone call (manual — UI prompts you)
Week 6  → Email (if available)
```
Sequence is tracked per parcel in the `outreach` table. UI surfaces what's due. Exact cadence and strategy TBD in outreach brainstorm.

### Draft & Approval Flow (high level)
1. Select one or more sites in the ranked list
2. Click "Draft Outreach" → system generates a personalized draft per site per channel using templates from `config/templates/`, merging in property-specific data (address, lot size, owner name, specific characteristics that scored well)
3. Review each draft in the UI — edit inline if needed
4. Approve → triggers send via Gmail API / Lob API
5. Sent date logged in `outreach` table; stage updated to `outreach`

### Open questions for dedicated session
- Exact template language and tone
- How much personalization per site (which fields get merged)
- Broker route: when a property is listed, does the lead just get flagged for your developer partner, or do you still mail the owner directly?
- Follow-up: same template or different tone on second touch?
- Sequence scheduler: UI prompt or background check on app load?
- Lob letter vs. postcard decision for follow-up touch

---

## Section 6 — Feedback Loop & Score Iteration

### Overview
After each outreach wave closes out, the system shows you what happened so you can learn and adjust. Raw stats plus your own wave notes inform manual weight adjustments in `config/scoring.yaml`.

### Feedback signals
1. **Response rate:** Did the owner respond? Tracked per site, per channel, per scoring signal.
2. **Handoff outcome:** Was the lead introduced to developer partner? Tracked as a boolean per site.
3. **Wave notes:** Free-text field per wave capturing qualitative learning — what you noticed about owner psychology, which neighborhoods respond better, what site characteristics your developer partner actually cares about, why deals didn't work out. This is where the "why" lives.

### Iteration flow
1. You log responses and handoff outcomes in the UI as they come in
2. After a wave closes out, write wave notes capturing what you learned
3. Open the **Feedback Report** section in the UI to see per-signal stats (response rates, handoff rates by signal)
4. Read the stats, read your wave notes, and manually adjust weights in `config/scoring.yaml` based on your judgment
5. Version bump on config change; all existing parcels record which score version they were evaluated under — re-scoring is optional but available

### Minimum data caveat
Don't tune weights until you have at least 2–3 completed waves with meaningful response counts. Early waves are for learning, not optimization.

---


## Future Brainstorm Sessions Required

### ~~Session: Scoring System~~ — COMPLETE (no separate spec needed)
Scoring system fully defined: signals come from data sources spec, initial weights from historical analysis script, scoring math is simple weighted sum (continuous normalized to 0-1, binary as 0/1, all multiplied by weight, summed, normalized to 0-100). Everything scored, filterable exclusions on output. Consolidated parcels scored individually and as combined entity.

### ~~Session: Data Sources~~ — COMPLETE
**Produced:** `2026-04-11-pipeline-data-sources-design.md`

### Session: Outreach Templates & Sequence
**Produces:** `YYYY-MM-DD-pipeline-outreach-design.md`
**Must answer:**
- Contact enrichment provider selection (BatchSkipTracing or alternative)
- Physical mail template: tone, length, what property-specific data gets merged in, Lob handwritten font selection
- Email template: subject line strategy, body, signature
- How personalization works: which fields from the parcel/contact record get inserted
- Follow-up templates: are they different in tone from the first touch?
- Broker route: when a property is listed, does the lead just get flagged for your developer partner, or do you still mail the owner directly?
- Gmail API: OAuth setup, sent-from address, how to avoid spam filters
- Lob API: account setup, address validation, return address, letter vs. postcard decision for follow-up
- Sequence scheduler: how does the system know when to surface "week 3 follow-up ready" — is this a UI prompt or automated?
- Outreach positioning: how to frame yourself as prospective buyer (license constraint)
