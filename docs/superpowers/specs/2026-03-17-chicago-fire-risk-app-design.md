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
- **Traffic fatalities:** CDOT annual report
- **Homelessness/housing-related deaths:** IDPH or Cook County Medical Examiner data
- **Fire fatalities:** CFD annual report
- Used for comparative deaths "reframing the risk" section

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
- **3 stat cards:** Annual deaths in Chicago from (a) fires, (b) traffic, (c) housing unaffordability/homelessness
- **Purpose:** Immediately reframes the risk conversation before any fire data is shown

### Section 2: The Map
- Interactive map of Chicago fire incidents (2022–present)
- Markers colored by severity (fatality / injury / property damage only)
- Overlay toggle: building violations density
- **Filter controls:**
  - Year range slider
  - Ward selector (dropdown) — so an alderman can filter to their own ward
  - Incident type
- **Purpose:** Geographic grounding; ward filter makes it personal in aldermanic meetings

### Section 3: Building Violations & Fire Risk
- Bar chart: fire incident rate by violation history (buildings with prior violations vs. clean record)
- Possibly a scatter plot: violation count vs. fire incidents at address level
- **Purpose:** Shows that building maintenance and code compliance — not stairwell count — predicts fire risk

### Section 4: Sprinklers Save Lives
- **Source: NFPA national data** (to be replaced with Chicago FOIA data when available)
- Side-by-side bar chart: sprinklered vs. unsprinklered buildings — deaths per 1,000 fires, injuries per 1,000 fires, property damage per fire
- Callout stat: "Sprinklers reduce fire deaths by X%"
- **Purpose:** Core argument — sprinkler presence is the variable that matters, not stairwell count

### Section 5: New vs. Old Buildings
- Chart combining NFPA sprinkler data + Chicago building permit dates
- Frame: "A new sprinklered building with one stairwell is safer than an old unsprinklered building with two"
- Show fire outcomes by building era (pre-1970s vs. post-2000s construction permitted)
- **Purpose:** Directly addresses the "but two stairs are safer" objection

### Section 6: The Real Risk — Putting Fire in Context
- Horizontal bar chart (or bold stat blocks): annual deaths in Chicago from fires vs. traffic vs. homelessness/housing unaffordability
- Sub-note on housing cost context: how single-stair allowance enables more housing units
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

- **Map data:** Fetched client-side from Chicago Socrata API on page load (incidents + violations), with loading states
- **Chart data:** Mix of static (NFPA, comparative deaths) and Socrata API fetches
- **No backend:** All data comes directly from public APIs or is hardcoded from published reports
- **Future:** When FOIA data arrives, replace NFPA static constants with Chicago-specific fetched data

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
- The comparative deaths section is the first thing a visitor sees
- The site loads in under 3 seconds on a typical connection
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
