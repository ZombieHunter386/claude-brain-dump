# Chicago Off-Market Multifamily Pipeline — System Design

**Date:** 2026-04-09
**Status:** Architecture, pipeline stages, data model, Review UI, outreach overview, and feedback loop overview complete. Three subsystem specs still required before coding — see Spec Suite table below.

---

## Spec Suite — Full File Plan

This system requires multiple spec files before it is ready to hand off for implementation. Each file must be complete before coding begins. This file is the **master architecture spec** — all other specs are subsystem specs that plug into it.

| File | Status | What it covers |
|---|---|---|
| `2026-04-09-chicago-multifamily-pipeline-design.md` | **This file** — in progress | System architecture, pipeline stages, data model, UI layout, outreach flow, feedback loop |
| `YYYY-MM-DD-pipeline-gates-design.md` | Not started — needs brainstorm session | Exact pre-filter criteria; Gate 1 financial viability formula, thresholds, and data fields; Gate 2 motivation signals, starting weights, and score tier cutoffs; adjacent consolidation logic |
| `YYYY-MM-DD-pipeline-data-sources-design.md` | Not started — needs brainstorm session | Full list of free APIs by stage; Cook County Assessor API field mapping; Chicago Data Portal endpoints; rate limits and pagination handling; how new sources get added incrementally |
| `YYYY-MM-DD-pipeline-outreach-design.md` | Not started — needs brainstorm session | Message templates per channel (mail, email); sequence timing and scheduling logic; Gmail API setup; Lob API setup and handwritten-style configuration; broker route workflow; how drafts are personalized per site |

**Coding should not begin until all four files exist and are approved.**

---

## Goals

1. Automatically screen LP/Lakeview parcels through configurable gates with minimal manual input
2. Understand why a site is good — explainable scores, not just a number
3. Make outreach feel personal at scale — drafted messages, multi-channel, human approves before anything sends
4. Iterate both gates based on response rate AND whether the deal penciled
5. Laptop-first, local tool — no hosting required; SQLite file shareable by email with partner

---

## Constraints & Decisions

- **Tooling:** Python pipeline + Flask web UI + SQLite — single language, single portable file, no hosting
- **Vibe coding:** Code will be rewritten by Claude as needed; reuse of existing CRE app is not a priority
- **Gate parameters:** Exact signals and weights are TBD — to be defined in a future brainstorm session. Both gates are fully configurable via YAML; changing a signal may require a code change, changing a weight does not.
- **Free databases only initially** — paid enrichment (BatchSkipTracing) only after both gates pass
- **No Street View** — replaced with "Open in Google Maps" link; avoids Google API key dependency
- **Geography is configurable** — defined as a polygon in YAML config; can be narrowed or expanded without code changes. Initial target: Lakeview + Lincoln Park (Chicago community areas 6 & 7)

---

## Architecture Overview

Five layers, each with a single responsibility:

```
CONFIG (YAML)
  Geography polygon · gate thresholds · scoring weights · outreach templates
  └─ Updated by approved Claude API suggestions after each outreach wave
        ↓
PIPELINE (Python scripts — run on demand)
  Pre-filter → Gate 1 → Gate 2 → Enrichment
  └─ Writes results to SQLite at each stage
        ↓
DATABASE (SQLite — single portable file)
  All pre-filter survivors tagged by stage reached
  └─ Email .sqlite to partner; re-score without re-fetching
        ↓
REVIEW UI (Flask — localhost:5000)
  Map + ranked list + score breakdown + outreach drafting + feedback report
        ↓
OUTREACH (APIs — gated by human approval)
  Gmail API · Lob API · phone (manual) · in-person (manual) · broker route
```

**Key principles:**
- Pipeline and UI are decoupled — pipeline runs as a script, not triggered from the browser
- Config drives parameter tuning; new signals require a code change + new config entry
- SQLite is the handoff point between pipeline and UI
- Nothing sends without explicit approval in the UI

---

## Section 1 — Config (YAML)

Two config files:

**`config/geography.yaml`** — search area polygon (GeoJSON or bounding coordinates). Change this to expand/narrow search without touching code.

**`config/gates.yaml`** — all gate parameters, scoring weights, and thresholds. Versioned so each pipeline run records which version was used. Changing weights = edit this file. Adding a new signal = edit code + add entry here.

Outreach message templates also stored as separate files (`config/templates/`) referenced from gates.yaml.

---

## Section 2 — Pipeline (4-Stage Cascade)

### Pre-filter (Stage 0)
- **Source:** Cook County Assessor API (Socrata) — single database
- **Pulls:** PIN, lot size, zoning code, land use code, owner name, owner mailing address, assessed value, year built
- **Filters:** Basic lot size threshold, zoning codes worth investigating, land use exclusions — exact parameters TBD in gate brainstorm session
- **Adjacent consolidation:** Group parcels by owner name + mailing address. For groups with 2+ parcels, check proximity (lat/lng within ~50ft or sequential Cook County block/lot numbers). Flag adjacent same-owner parcels as consolidation candidates with combined lot size. This runs at pre-filter because it uses only Cook County data.
- **Output:** ~500–2,000 parcels from ~10,000–15,000 total

### Gate 1 — Financial Viability
- **Sources:** Multiple free databases (Chicago Data Portal — zoning, permits, building footprints; Cook County Recorder — sale history). Exact sources TBD in gate brainstorm session.
- **Logic:** FAR-based unit potential calculation, land cost per unit estimate from nearby comps. Exact parameters TBD.
- **Output:** ~50–150 survivors

### Gate 2 — Motivation Scoring
- **Sources:** Cook County Treasurer (tax delinquency), hold duration from sale history, LLC detection, other signals TBD
- **Logic:** Weighted score 0–100. Score tiers (active / watch / hold) TBD in gate brainstorm session.
- **Output:** ~20–50 active queue

### Enrichment (Stage 3 — paid)
- **Sources:** BatchSkipTracing (~$0.10–0.15/record) for phone + email; Illinois SOS for LLC registered agent (free, may require scraping)
- **Broker check:** Check if property is actively listed before outreach — if listed, route to broker instead of direct owner contact
- **Expected cost:** $3–8 per pipeline run
- **Output:** Final active queue with contact info appended

---

## Section 3 — Database (SQLite)

### Schema

**`parcels` table** — PIN is unique key (Cook County format: `14-21-101-001-0000`)

| Column group | Columns |
|---|---|
| Identity | address, lat, lng, owner_name, owner_address |
| Pre-filter | lot_size_sf, zoning_code, land_use_code, assessed_value, year_built, consolidation_group_id |
| Gate 1 | max_far, built_far, permit_count, land_cost_per_unit, achievable_units, gate1_score, gate1_pass |
| Gate 2 | hold_years, tax_delinquent, absentee_owner, single_asset_llc, use_discontinuity, gate2_score *(columns illustrative — finalized in gate brainstorm session)* |
| Status | stage, score_version, first_seen_date, last_updated_date |

Stage values: `pre_filter` / `gate1_fail` / `gate2_fail` / `active` / `outreach` / `responded` / `dead`

**`consolidation_groups` table** — synthetic group_id for adjacent same-owner parcels

| Column | Notes |
|---|---|
| group_id | auto-increment PK |
| pins | JSON array of PINs |
| combined_lot_size_sf | sum of constituent lots |
| combined_units_potential | Gate 1 calc on combined footprint |
| owner_name | shared owner |
| detected_at_prefilter | date flagged |

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
| source | `assessor` / `skip_trace` / `broker_check` |

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
| deal_penciled | bool — logged manually after underwriting |
| notes | free text |

### Key design decisions

- **Store everything that passes pre-filter** — tagged by stage reached. Allows re-scoring with new weights without re-fetching API data.
- **Upsert on PIN** — `INSERT ... ON CONFLICT(pin) DO UPDATE SET ...` — new API pulls refresh existing rows, add new parcels, never touch `first_seen_date`
- **Spatial logic in Python** — geopandas handles geographic filtering and adjacency checks before writing to SQLite; SQLite never does geometry math
- **Scale:** SQLite is sufficient. LP/Lakeview (~15k parcels) is trivial; all of Chicago (~600k) is manageable. Upgrade to PostgreSQL + PostGIS only if expanding to all of Cook County with complex spatial queries.

---

## Section 4 — Review UI (Flask)

**Runs at:** `localhost:5000`

### Layout
Three-column interface:
- **Left:** Ranked list — sorted by combined score, filterable by stage. Score badge, motivation signal tags, outreach status per row. Batch "Draft Outreach" button at bottom.
- **Center:** Leaflet.js map (OpenStreetMap tiles, no API key). Color-coded pins by stage: active (green), consolidated (purple), outreach sent (blue), watch list (yellow). Layer toggles for Gate 1 / Gate 2 / outreach layers.
- **Right:** Detail panel for selected site — property facts (PIN, address, lot size, zoning), score breakdown showing which signals fired and by how much (config-driven, not hardcoded), Gate 1 financial summary, "Open in Google Maps" link, outreach action buttons (Draft / Watch / Skip / Notes). Score version stamp shown so you know if site was scored under old weights.

**Gate suggestions:** Not a persistent banner. Lives in a "Feedback Report" section, plus a prompt when starting a new pipeline run if wave data exists. See Section 6.

### Score breakdown
Pulls signal names and weights dynamically from the current YAML config. If you add a new signal or change a weight, the breakdown reflects it automatically. Each parcel records `score_version` so old scores remain interpretable after config changes.

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
| Broker route | Automated/Manual | TBD | Triggered when broker detected at enrichment stage; different message |

### Outreach Sequence
```
Week 1  → Physical mail (letter)
Week 3  → Follow-up mail or postcard
Week 5  → Phone call (manual — UI prompts you)
Week 6  → Email (if available)
```
Sequence is tracked per parcel in the `outreach` table. UI surfaces what's due.

### Draft & Approval Flow (high level)
1. Select one or more sites in the ranked list
2. Click "Draft Outreach" → system generates a personalized draft per site per channel using templates from `config/templates/`, merging in property-specific data (address, lot size, owner name, specific characteristics that scored well)
3. Review each draft in the UI — edit inline if needed
4. Approve → triggers send via Gmail API / Lob API
5. Sent date logged in `outreach` table; stage updated to `outreach`

### Open questions for dedicated session
- Exact template language and tone
- How much personalization per site (which fields get merged)
- Broker route: contact broker only, or also mail owner directly?
- Follow-up: same template or different tone on second touch?
- Sequence scheduler: UI prompt or background check on app load?
- Lob letter vs. postcard decision for follow-up touch

---

## Section 6 — Feedback Loop & Gate Iteration

**Status:** Overview captured — needs dedicated brainstorm session for full spec (part of `YYYY-MM-DD-pipeline-gates-design.md`)

### Overview
After each outreach wave closes out, the system analyzes what happened and suggests updates to both Gate 1 and Gate 2 parameters. The goal is that the pipeline gets smarter over time — both at identifying motivated sellers and at predicting whether a deal will pencil.

### Two feedback signals
1. **Response rate:** Did the owner respond? Tracked per site, per channel, per motivation signal
2. **Pencilability:** Did the deal actually pencil when you dug into it? Logged manually per site in the UI

Both signals feed back into both gates — not just Gate 2 (motivation) but also Gate 1 (financial viability thresholds).

### Iteration flow
1. You log responses and pencilability outcomes in the UI as they come in
2. When ready to analyze, open the **Feedback Report** section in the UI
3. Click "Analyze Wave" → Python compiles per-signal stats and sends to **Claude API** (requires Anthropic API key in env)
4. Claude returns plain-language suggestions — e.g. "Absentee owner correlated with 2× response rate — consider raising weight. Single asset LLC showed no signal — consider reducing weight."
5. Each suggestion is displayed with the underlying data so you can evaluate it
6. You approve or reject each suggestion individually
7. Approved suggestions write back to `config/gates.yaml` with a version bump
8. All existing parcels in the DB record which score version they were evaluated under — re-scoring is optional but available

### Fallback
If Claude API is unavailable or you skip the analysis: raw stats (per-signal response rates, pencil rates) are still shown in the Feedback Report. You can adjust weights manually in the YAML at any time.

### Minimum data caveat
Claude is prompted to flag when sample sizes are too small to draw conclusions. Early runs will return "not enough data yet" for most signals — this is correct. Don't tune weights until you have at least 2–3 completed waves with meaningful response counts.

### Open questions for dedicated session
- Exact structure of the data payload sent to Claude API
- How score versioning works when you partially re-score (some parcels on v3, some on v4)
- Whether the Feedback Report lives as a separate page or a section within the main UI
- Trigger for surfacing "wave analysis available" — on app load, or manual only

---


## Future Brainstorm Sessions Required

### Session: Gate Parameters
**Produces:** `YYYY-MM-DD-pipeline-gates-design.md`
**Must answer:**
- Pre-filter: minimum lot size, which zoning codes to keep/drop, which land use codes to exclude
- Adjacent consolidation: exact proximity threshold (ft), how to handle 3+ adjacent parcels
- Gate 1: FAR calculation method, efficiency ratio assumption, minimum unit threshold, land cost/unit formula, fallback comp logic, rezoning corridor logic (separate bucket or flag?)
- Gate 2: full list of motivation signals, starting weight per signal, score tier cutoffs (active/watch/hold), how "long hold" is defined in years
- Both gates: what triggers a Watch List site moving to Active Queue (nearby sale, tax delinquency event, permit expiration)
- Score versioning: how to handle re-scoring existing parcels when weights change

### Session: Data Sources
**Produces:** `YYYY-MM-DD-pipeline-data-sources-design.md`
**Must answer:**
- Cook County Assessor Socrata API: exact endpoint, field names, pagination, geographic filter syntax
- Chicago Data Portal: which datasets, which endpoints, field mappings for zoning/permits/land use/building footprints
- Cook County Treasurer: tax delinquency API — endpoint, field names, how delinquency is flagged
- Cook County Recorder: sale history — endpoint, how to get $/sf comps within ¼ mile
- Illinois SOS LLC lookup: is there an API or is this a scrape? What fields are available?
- Broker/listing check: which free source to use (LoopNet scrape? CoStar? MLS? None initially?)
- Rate limits for each source and how the pipeline handles them (sleep, retry, cache)
- How to add a new data source without restructuring the pipeline

### Session: Outreach Templates & Sequence
**Produces:** `YYYY-MM-DD-pipeline-outreach-design.md`
**Must answer:**
- Physical mail template: tone, length, what property-specific data gets merged in, Lob handwritten font selection
- Email template: subject line strategy, body, signature
- How personalization works: which fields from the parcel/contact record get inserted
- Follow-up templates: are they different in tone from the first touch?
- Broker route: what changes when a broker is detected — do you still mail the owner or only contact the broker?
- Gmail API: OAuth setup, sent-from address, how to avoid spam filters
- Lob API: account setup, address validation, return address, letter vs. postcard decision for follow-up
- Sequence scheduler: how does the system know when to surface "week 3 follow-up ready" — is this a UI prompt or automated?
