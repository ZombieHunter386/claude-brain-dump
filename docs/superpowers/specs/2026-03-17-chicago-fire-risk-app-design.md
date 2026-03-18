# Chicago Fire Risk Factors App — Design Spec
**Date:** 2026-03-17
**Project:** Strong Towns Chicago Advocacy Tool
**Status:** Approved for implementation

---

## Purpose

A public-facing web app that visualizes Chicago fire incident data to make a data-driven case for allowing single-stair residential buildings. The primary audience is Chicago aldermen and the public. The core arguments are:

1. Buildings with sprinklers have dramatically better fire outcomes — sprinkler presence matters far more than stairwell count
2. New buildings with sprinklers (single-stair) are safer than old buildings without sprinklers (two-stair)
3. Fire deaths are far fewer than deaths from traffic and housing unaffordability/homelessness — we are over-regulating fire safety at the cost of housing

---

## Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | Next.js (App Router) | Full control, fast, great for public-facing sites |
| Map | React-Leaflet + OpenStreetMap | Free tiles, address-level incident plotting |
| Charts | Recharts | Simple, composable, well-documented |
| Data fetching | Chicago Socrata API + NFPA static data | No backend needed; data fetched at build time or client-side |
| Hosting | Vercel (free tier) | Purpose-built for Next.js, free for this use case |

---

## Data Sources

### Chicago Open Data (Socrata API)
- **CFD Fire Incidents / Emergency Events** — incident locations 2022–present, used for map and geographic analysis
- **Building Violations** (2006–present) — joinable by address to fire incidents; used to show violation density correlating with fire risk
- **Building Permits** (2006–present) — joinable by address/PIN; used as proxy for building age and renovation history
- **Fire Stations** — locations for map context

### NFPA (National Fire Protection Association) — Static Data
- Sprinklered vs. unsprinklered building fire outcomes (deaths, injuries, property damage)
- Used as authoritative source for the sprinkler effectiveness argument since Chicago-specific sprinkler data requires FOIA

### Chicago / Illinois Public Sources — Static Data
- **Traffic fatalities:** CDOT annual report (specific figure: CDOT publishes annual traffic fatality counts for Chicago)
- **Unsheltered/homelessness-related deaths:** Chicago DFSS (Dept of Family & Support Services) Annual Homeless Point-in-Time Count report, which includes deaths among unsheltered individuals. Use "deaths among persons experiencing homelessness" — a defined count, not a derived estimate. Alternatively, Cook County Medical Examiner annual report, filtering for "exposure/environmental" and "undetermined" causes with no fixed address. Nail down the exact source and year before implementation; do not use an estimated or methodologically derived number.
- **Fire fatalities:** CFD Annual Report (published by Chicago Fire Dept, available on city website)
- All three figures must be from the same year for a valid comparison. Use the most recent year where all three are available.
- Used for the Hero stat cards and Section 6 comparative chart

### FOIA Requests (Future Enrichment — not blocking v1)
- CFD sprinkler permit database by address — will replace NFPA data with Chicago-specific data when available
- Fire incident fatalities and injuries at incident level — will enrich map and outcome analysis
- Construction type per incident — will enable segmentation by building type
- Complete CFD response time records — will enable density vs. response time analysis

---

## Page Architecture

Single scrollable page with sticky navigation. Narrative arc builds the advocacy argument section by section.

### Section 1: Hero
- **Headline:** Bold, advocacy-forward (e.g., "Chicago's Fire Rules Are Costing Lives — Just Not the Ones You Think")
- **3 stat cards:** Annual deaths in Chicago from (a) fires, (b) traffic, (c) homelessness/unsheltered deaths — all from the same year, all static hardcoded values with source citations shown inline
- **Purpose:** Comparative deaths reframing is the first thing a visitor sees, before any fire data

### Section 2: The Map
- Interactive map of Chicago fire incidents (2022–present)
- **Marker severity coloring:** Derived from the `incident_type_desc` or equivalent field in the CFD dataset. Three buckets: `fatality` (red) = incident type includes "fatal" or "death"; `injury` (orange) = incident type includes "injury"; `property-only` (yellow) = all others. Exact field name and values to be confirmed against live dataset schema before implementation.
- **Ward filter:** Chicago ward boundaries GeoJSON is available from the Chicago Data Portal (`/Facilities-Geographic-Boundaries/Ward-Offices/`). Perform a client-side point-in-polygon join (use `@turf/boolean-point-in-polygon` from the Turf.js library) to assign each incident to a ward at load time. Ward selector dropdown lists wards 1–50.
- **Data volume / performance:** Fetch incidents from Socrata API with a `$limit=5000` cap and `$order=date DESC` to get the most recent incidents. Use React-Leaflet's `MarkerCluster` plugin to cluster markers at low zoom levels. Fire station locations and ward boundary GeoJSON are fetched separately.
- Overlay toggle: building violations density (heatmap layer, violations fetched with same address-level approach)
- **Filter controls:** Year range slider, ward selector (dropdown), incident type
- **Purpose:** Geographic grounding; ward filter makes it personal in aldermanic meetings

### Section 3: Building Violations & Fire Risk
- Bar chart: fire incident count at addresses with prior violations vs. addresses with no violation history
- **Address join strategy:** Normalize all addresses to uppercase, strip unit numbers, standardize directionals (N/S/E/W), and abbreviate street types (ST/AVE/BLVD) before joining. Use a dedicated `normalizeAddress()` utility in `lib/utils.ts`. Match on normalized address string. Accept that some joins will fail; note match rate in a dev comment.
- Scatter plot (optional, secondary): violation count vs. fire incident count per address
- **Purpose:** Shows that building maintenance and code compliance — not stairwell count — predicts fire risk

### Section 4: Sprinklers Save Lives
- **Source: NFPA national data** (to be replaced with Chicago FOIA data when available)
- Side-by-side bar chart: sprinklered vs. unsprinklered buildings — deaths per 1,000 fires, injuries per 1,000 fires, property damage per fire
- Callout stat: "Sprinklers reduce fire deaths by X%"
- **Purpose:** Core argument — sprinkler presence is the variable that matters, not stairwell count

### Section 5: New vs. Old Buildings
- **Source: NFPA national data only** (not joined with Chicago permit data — that join is not data-feasible since Chicago incident data lacks per-building outcome fields)
- Grouped bar chart using NFPA data: compare fire outcomes (deaths/1000 fires, injuries/1000 fires, avg property damage) for (a) older unsprinklered buildings vs. (b) newer sprinklered buildings, using NFPA era/construction-type breakdowns
- Headline framing: "A new building with one stairwell and sprinklers is safer than an old building with two stairs and no sprinklers"
- Include a callout box with the explicit caveat: "National data — Chicago-specific building-era breakdown requires FOIA (in progress)"
- Chicago building permit data is NOT used in this section for v1
- **Purpose:** Directly addresses the "but two stairs are safer" objection using authoritative national data

### Section 6: The Real Risk — Putting Fire in Context
- Horizontal bar chart: annual deaths in Chicago from (a) fires, (b) traffic, (c) homelessness/unsheltered — same static data as Hero stat cards, displayed as a chart for visual comparison
- Sub-note on housing cost context: how single-stair allowance enables more housing units per lot
- **Note:** Hero (Section 1) and this section both show comparative deaths; Hero shows the numbers first as a hook, Section 6 shows the full chart as the climactic argument. This is intentional — not duplication.
- **Purpose:** Closes the argument — we're trading many lives lost to housing unaffordability to prevent far fewer fire deaths

### Section 7: The Ask
- Short advocacy copy (3-5 sentences)
- CTA button: contact your alderman (link to city alderman directory)
- Optional: link to Strong Towns Chicago resources

---

## Component Structure

```
app/
  page.tsx                  # Main scrollable page, assembles all sections
  layout.tsx                # Shared layout, metadata, nav

components/
  Hero.tsx                  # Stat cards + headline
  FireMap.tsx               # React-Leaflet map with filters
  ViolationsChart.tsx       # Violations vs fire risk bar chart
  SprinklersChart.tsx       # NFPA sprinkler effectiveness charts
  BuildingEraChart.tsx      # New vs old building outcomes
  RiskComparisonChart.tsx   # Fire vs traffic vs homelessness deaths
  TheAsk.tsx                # Advocacy copy + CTA
  NavBar.tsx                # Sticky section navigation

lib/
  fetchFireIncidents.ts     # Socrata API calls for fire incidents
  fetchBuildingViolations.ts
  fetchBuildingPermits.ts
  nfpaData.ts               # Static NFPA data constants
  comparativeDeathsData.ts  # Static Chicago death comparison data
  utils.ts                  # Shared helpers (formatters, color scales)
```

---

## Data Flow

- **Map data:** Fetched client-side from Chicago Socrata API on page load. Capped at `$limit=5000` with `$order=date DESC` to control volume. Marker clustering via React-Leaflet MarkerCluster. Ward boundary GeoJSON fetched separately from Chicago Data Portal for point-in-polygon ward assignment (Turf.js).
- **Chart data (Sections 3 violations):** Fetched client-side from Socrata API; address-normalized before join.
- **Chart data (Sections 4, 5, 6, Hero):** Static — hardcoded constants in `nfpaData.ts` and `comparativeDeathsData.ts`. No API calls.
- **No backend:** All data comes directly from public APIs or is hardcoded from published reports.
- **Future:** When FOIA data arrives, replace static NFPA constants with Chicago-specific fetched data.

---

## MVP Scope (v1)

**In scope:**
- All 7 page sections
- Map with ward filter and severity coloring
- NFPA-based sprinkler charts (static data)
- Comparative deaths section (static data from published reports)
- Building violations correlation chart
- Vercel deployment

**Out of scope for v1 (post-FOIA):**
- Chicago-specific sprinkler data
- Incident-level fatality/injury data
- Construction type segmentation
- Response time vs. density analysis

---

## Success Criteria

- An alderman can filter the map to their ward in under 10 seconds
- The sprinkler argument is clear and citable (NFPA source shown)
- Comparative deaths stat cards are visible in the Hero (first section) — the visitor sees the reframing before any fire map
- The site loads in under 3 seconds on a typical connection (enforced by 5,000-row map cap and static chart data)
- Deployable and publicly accessible via Vercel with a custom URL

---

## FOIA Enrichment Plan

File immediately with CFD:
1. Sprinkler presence / sprinkler permit records by address
2. Fire incident outcomes (fatalities, injuries) at incident level

File second (lower urgency):
3. Construction type per fire incident
4. Complete response time records (turnout + travel, unredacted)

When data arrives: replace static NFPA constants with Chicago-specific data, add construction type chart, add response time by neighborhood density chart.
