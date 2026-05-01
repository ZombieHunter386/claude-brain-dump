# bike-map-lakeview

A bike-grid map of Lakeview (Chicago) showing how to reach the everyday "life locations" — groceries, schools, hospitals, libraries, transit, parks, pharmacies — by bike, plus the strategic gaps in the network where infrastructure is missing.

## What this is

Two layers on one map:

1. **Destinations layer** — the places people actually need to reach in daily life, categorized.
2. **Bike network layer** — existing infrastructure (protected lanes, buffered lanes, painted lanes, neighborhood greenways, off-street trails) overlaid with the gaps where a rider has to merge into mixed traffic to complete a trip.

The point isn't "here's a route from A to B" (Google Maps does that). The point is **"here's why getting to your kid's school by bike sucks today, and here's the 0.4-mile gap that would fix it."**

## Why this is interesting

- **Advocacy artifact.** A specific, mappable list of "if the city paved/striped/protected *these* segments, X% more destinations become safely bikeable" is more persuasive than a generic "we need more bike lanes" ask at a CDOT meeting or alderman office hours.
- **Personal utility.** A grid view of Lakeview tells you which errands are realistically chainable by bike vs. which require a detour through a stroad.
- **Reusable template.** If the methodology works for Lakeview, it works for Lincoln Park, Logan Square, Uptown, etc.

## Open design questions (brainstorm)

### Scope of "Lakeview"
- East Lakeview vs. West Lakeview vs. Lakeview proper? Suggest: full community area (Diversey → Irving Park, Ravenswood → Lake Michigan) since the bike-relevant edges are arterials at the boundaries (Belmont, Ashland, Western, Lake Shore Trail).

### Destination categories (v1)
- Grocery (full-service vs. corner store)
- Schools (CPS + private, K–12)
- Hospitals + urgent care + clinics
- Pharmacies
- CTA stations (Brown, Red, bus rapid corridors)
- Libraries
- Parks + playgrounds
- Post office / civic
- Maybe: third-places (coffee shops with bike parking, gyms) — risk of scope creep

### "Bikeability" scoring per segment
Options, cheapest to most rigorous:
- **Tier A (fast):** classify each street by CDOT's published bike facility type. Color-code.
- **Tier B (medium):** add a Level of Traffic Stress (LTS 1–4) score using OSM tags + speed limits. Standard methodology, defensible.
- **Tier C (slow):** ride/observe each contested segment and score subjectively.
Recommend **Tier B** — it's the de facto standard advocates use and the data is free.

### "Strategic gap" definition
A gap is a segment where:
- It's on a likely shortest-path route between a destination cluster and residential blocks, AND
- Its LTS is ≥3 (stressful for most riders), AND
- A small intervention (paint, diverter, signal change, ≤0.5 mi of new lane) would drop it to LTS ≤2.
This gives a ranked punch-list rather than a vibe.

### Tech stack (recommendation)
- **Data:** OpenStreetMap (streets + amenities), Chicago Data Portal (bike lanes, schools, CTA), USDOT/HIFLD (hospitals).
- **Build:** Python notebook to pull + clean → GeoJSON.
- **Render:** static site with MapLibre GL JS or Leaflet, hosted on **GitHub Pages** (free, public, fits the "publicly accessible" requirement once the repo is public).
- **Why not a Google My Map:** no version control, no programmatic gap analysis, hard to share methodology.

### Stretch features (defer past v1)
- Isochrones: "what's reachable in a 10-min bike ride from each CTA station?"
- Before/after toggle for proposed CDOT projects.
- Crowd-sourced pin-drops for "I almost got hit here."
- Comparison overlay vs. car/transit travel time.

## Proposed v1 milestones

1. **Data ingest** — pull OSM + Chicago Data Portal layers, clip to Lakeview.
2. **Destination layer** — render categorized POIs.
3. **Network classification** — LTS score per street segment.
4. **Gap detection** — naive version: shortest-path between every residential centroid and nearest destination of each category, flag segments with LTS ≥3.
5. **Static site** — MapLibre + layer toggles, deploy to GitHub Pages.
6. **Writeup** — README with methodology + top 10 ranked gap interventions.

## Open questions for you

- Lakeview boundary: community area, or your personal mental map?
- Any destination categories above you'd cut, or any I'm missing (daycares? veterinarians? bike shops themselves)?
- Are you OK with v1 being a static map + ranked gap list, with crowd-sourcing/isochrones as v2?
- Do you want this to eventually feed into a specific advocacy ask (alderman, CDOT, Active Trans), or is it a personal/portfolio project?
