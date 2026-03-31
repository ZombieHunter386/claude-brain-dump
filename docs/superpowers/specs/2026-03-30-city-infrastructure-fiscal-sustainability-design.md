# City Infrastructure Fiscal Sustainability Tool — Design Spec
**Date:** 2026-03-30
**Project:** Infrastructure Fiscal Sustainability Analysis
**Status:** Approved for implementation

---

## Purpose

A Claude Project that enables anyone — city staff, outside advocates, Strong Towns members, or citizens — to upload publicly available (or FOIA'd) municipal financial documents and receive a structured **City Infrastructure Fiscal Sustainability Report**.

The report answers one core question: *What has this city already committed to maintain, what does that cost over the next 30 years, and does the city have the fiscal capacity to pay for it?*

This tool is designed as a companion to the [Strong Towns Financial Decoder](https://www.strongtowns.org/decoder). The Decoder shows overall fiscal health trajectory. This tool shows specifically what the infrastructure obligation looks like underneath that trajectory and what the long-term maintenance gap means in concrete, political terms.

### Intellectual Foundations
- [New Zealand Infrastructure Priorities Programme](https://tewaihanga.govt.nz/our-work/infrastructure-priorities-programme) — standardized independent assessment methodology
- [Te Waihanga: Paying It Forward](https://tewaihanga.govt.nz/our-work/research-insights/paying-it-forward-understanding-our-long-term-infrastructure-needs) — ~60% of infrastructure spending is just renewal/replacement of what already exists
- [Te Waihanga: Foundations for Growth](https://tewaihanga.govt.nz/our-work/research-insights/foundations-for-growth) — after depreciation and maintenance, many infrastructure investments produce negative fiscal returns
- Strong Towns fiscal analysis framework — infrastructure must generate enough new taxable value to cover its own lifecycle costs

---

## Architecture

### Delivery Format
A **Claude Project** in Claude.ai (Pro plan). No code, no hosting, no API required. The system prompt encodes the full methodology. Users upload documents to a conversation and receive a structured report.

### Who Uses It
Both insiders (city staff, administrators) and outsiders (advocates, Strong Towns members, researchers, citizens). Outsiders can FOIA documents that aren't publicly posted. The intake process is flexible — it works with whatever the user has.

---

## The Report Structure

The report is produced **section by section** in conversation — not as a single document at the end. Each section is validated before the next one is built on top of it. This prevents compounding errors from misread documents or wrong assumptions.

At the end of the full analysis, Claude produces a **single consolidated report** (described in detail below).

---

## The Consolidated Report

The consolidated report is the final, shareable output produced after all four parts have been validated. It is a single formatted document intended to be copied, pasted, or exported — usable in a council presentation, a public meeting, or a citizen communication without further editing.

### Structure of the Consolidated Report

**Cover Page**
- City name
- Date of analysis
- Three headline numbers (from Part 4): 30-year gap, maintenance coverage rate, tax increase to break even
- A single red/yellow/green infrastructure health rating
- One-paragraph plain-language summary of what the report found

**Section 1 — Infrastructure Inventory** *(from Part 1)*
The full asset table. Two columns per row: the technical data (replacement value, useful life) and a plain-language annotation (e.g., "Roads are approximately halfway through their useful life — major reconstruction costs will begin accelerating within 10 years").

**Section 2 — Maintenance Obligations & Funding Coverage** *(from Part 2)*
The maintenance gap table by asset class, plus the 30-year replacement schedule (see Excel model description below), plus a written narrative explaining where state/federal funding applies, how much of the obligation it covers, and what is left unfunded.

**Section 3 — Fiscal Capacity** *(from Part 3)*
A written summary of the city's revenue trajectory, existing debt load, and realistic infrastructure spending capacity. Includes a table of projected available infrastructure dollars per year for the next 10/20/30 years.

**Section 4 — The Gap** *(from Part 4)*
The headline numbers with full explanation. Includes:
- The 30-year obligation vs. capacity comparison
- The "doing nothing" cost: what the deferred backlog becomes in 10, 20, and 30 years if the gap is not addressed
- The tax increase calculation with methodology shown

**Assumptions & Limitations**
A plain-language disclosure of every estimate, benchmark rate, and gap in the data. This section makes the analysis defensible: it shows the work and flags where numbers are data-derived vs. industry-standard estimates.

### Two Versions

Both versions contain the same findings. The difference is tone and language:

- **Technical version** — precise numbers, stated assumptions, benchmark sources cited (APWA, ASCE, GASB 34), methodology explained. Intended for city finance staff, auditors, or anyone who will scrutinize the numbers.
- **Plain-language version** — same findings written for a council member or resident who will not read footnotes. Uses concrete analogies (e.g., "This is like owning a house and budgeting for only half the maintenance it needs — the house doesn't fall apart immediately, but every skipped year makes the eventual repair more expensive"). Avoids jargon. Every number is paired with what it means in practical terms.

Claude produces both in the same response, clearly labeled.

---

### Part 1: Infrastructure Inventory & Asset Health

Claude reads the capital assets schedule (typically in the CAFR's notes to financial statements) and any available CIP, asset management plans, or inspection reports.

**Analysis performed:**
For each asset class (roads, water/sewer, bridges, parks & recreation facilities, municipal buildings, fleet/equipment, stormwater), Claude extracts or estimates:
- Original cost (from CAFR capital assets schedule)
- Accumulated depreciation (from CAFR)
- Net book value (original cost minus accumulated depreciation)
- Estimated useful life (from CAFR notes — e.g., roads = 20–40 years)
- Estimated age (derived from useful life and depreciation percentage)
- Remaining useful life in years
- Current replacement cost (net book value adjusted upward for construction cost inflation — typically 3–5% annually since original construction)
- Condition rating: if a formal inspection report exists, use it; if not, use accumulated depreciation as a proxy (0–33% depreciated = Good, 34–66% = Fair, 67–100% = Poor)

**Output — Part 1 Table:**

| Asset Class | Original Cost | Accum. Depreciation | % Depreciated | Est. Remaining Life | Current Replacement Value | Condition |
|---|---|---|---|---|---|---|
| Roads & Streets | $X | $X | X% | X yrs | $X | Fair |
| Water System | $X | $X | X% | X yrs | $X | Good |
| Sewer System | $X | $X | X% | X yrs | $X | Fair |
| Stormwater | $X | $X | X% | X yrs | $X | Poor |
| Bridges | $X | $X | X% | X yrs | $X | Good |
| Municipal Buildings | $X | $X | X% | X yrs | $X | Fair |
| Parks & Recreation | $X | $X | X% | X yrs | $X | Fair |
| Fleet & Equipment | $X | $X | X% | X yrs | $X | Good |
| **TOTAL** | **$X** | **$X** | **X%** | — | **$X** | — |

A written narrative accompanies the table, flagging which asset classes are entering the critical replacement window (remaining life under 10 years) and which appear adequately maintained.

**Data limitations for this part:** See Feasibility section below.

**Validation checkpoint:** User confirms the asset inventory looks accurate and complete before Part 2 runs.

---

### Part 2: Annual Maintenance Obligation & Funding Coverage

Using industry-standard maintenance rate benchmarks, Claude calculates what the city *should* be spending annually versus what it's actually spending, accounts for all dedicated funding sources, and builds the 30-year replacement schedule.

**Maintenance rate benchmarks by asset class** (source: APWA, ASCE infrastructure report cards):
- Roads & streets: 2–4% of replacement value per year (Claude uses midpoint of available range and states assumption)
- Water systems: 1–2% per year
- Sewer systems: 1–2% per year
- Stormwater: 1–2% per year
- Bridges: 1–2% per year
- Municipal buildings: 2–3% per year
- Parks & recreation: 1–2% per year
- Fleet & equipment: 10–15% per year (shorter useful life, high annual replacement need)

**Funding source accounting:**
Claude identifies all state and federal pass-through revenues from the documents provided, maps each to the asset class it is designated for, and calculates net obligation. Common sources include:
- Motor Fuel Tax (MFT) / state road fund allotments → Roads
- State/federal water & sewer grants → Water/Sewer
- Community Development Block Grant (CDBG) → Variable
- Federal transportation grants (CMAQ, STP, TAP) → Roads/Bridges
- State capital programs → Variable

**Output — Part 2 Maintenance Gap Table:**

| Asset Class | Gross Annual Obligation | State/Federal Funding | Net City Obligation | Actual Spending | Annual Gap |
|---|---|---|---|---|---|
| Roads & Streets | $X | $X | $X | $X | **($X)** |
| Water System | $X | $X | $X | $X | **($X)** |
| Sewer System | $X | $X | $X | $X | **($X)** |
| Stormwater | $X | $X | $X | $X | **($X)** |
| Bridges | $X | $X | $X | $X | **($X)** |
| Municipal Buildings | $X | $X | $X | $X | **($X)** |
| Parks & Recreation | $X | $X | $X | $X | **($X)** |
| Fleet & Equipment | $X | $X | $X | $X | **($X)** |
| **TOTAL** | **$X** | **$X** | **$X** | **$X** | **($X)** |

**The 30-Year Replacement Schedule (Excel Model)**

The 30-year replacement schedule is the most complex output in the report. It is produced as a structured table in the conversation — formatted so it can be copied directly into Excel or Google Sheets with no reformatting required.

The model is structured as follows:
- **Rows:** One row per asset class, plus a totals row, plus a "cumulative deferred backlog" row at the bottom
- **Columns:** One column per year (Year 1 through Year 30), plus a "Total" column
- **Cell values:** The scheduled replacement cost for that asset class in that year, calculated from remaining useful life in Part 1. When an asset class hits the end of its useful life, its full replacement value appears in that year's cell.
- **Annual maintenance row:** Below the replacement schedule, a row for annual maintenance obligation (from the gap table above) is added for each year, allowing the user to see the combined annual infrastructure cost (maintenance + scheduled replacement) for any given year
- **Revenue row:** The city's projected annual infrastructure spending capacity (from Part 3) is shown as a comparison row
- **Net position row:** Annual obligation minus projected capacity — positive means funded, negative means gap

This model allows the user to answer: *"In year 7, we have $X in scheduled road reconstruction plus $Y in annual maintenance across all assets. Our projected revenue covers $Z. The shortfall is $W."*

The model uses two clearly stated assumptions that Claude discloses: (1) replacement costs are held constant in today's dollars unless an inflation rate is specified; (2) assets are replaced exactly at end of useful life, which is conservative — deferred replacement means higher costs.

**Deferred maintenance backlog:**
If the city has been underspending on annual maintenance, Claude calculates the compounding backlog. For each year of underspending, the deferred amount is assumed to increase future replacement costs by a penalty factor (industry standard: 4–5x — a dollar of deferred maintenance typically costs $4–5 to fix later when it becomes structural failure rather than preventive maintenance). This produces the total deferred backlog figure.

**Validation checkpoint:** User confirms the maintenance gap table and replacement schedule before Part 3.

---

### Part 3: Fiscal Capacity

Claude reads the CAFR's revenue history, debt schedules, and tax base trend data to establish what the city can realistically spend on infrastructure over the next 30 years.

**Analysis performed:**
- **Revenue trend:** Last 5–10 years of total general fund revenue and any dedicated infrastructure fund revenue. Claude calculates the compound annual growth rate (CAGR) and projects forward using that rate — if the trend is flat or negative, this is flagged prominently.
- **Infrastructure spending share:** What percentage of the city's total spending has gone to infrastructure (maintenance + capital) historically. This is compared to the obligation calculated in Part 2.
- **Existing debt load:** Total outstanding debt principal, annual debt service payments, and when existing debt retires. Debt service consumes revenue that cannot go to pay-as-you-go maintenance.
- **Borrowing capacity:** Estimated remaining debt capacity (most states cap municipal debt as a percentage of assessed value). This is the city's ability to finance large capital projects.
- **Dedicated infrastructure funds:** TIF districts, special service areas, water/sewer enterprise funds, and any other revenue streams legally restricted to infrastructure. These are separated from general fund capacity.

**Output — Part 3 Summary Table:**

| Metric | Current | 10-Year Projection | 20-Year Projection | 30-Year Projection |
|---|---|---|---|---|
| Total annual revenue | $X | $X | $X | $X |
| Annual debt service | $X | $X | $X | $X |
| Available for infrastructure | $X | $X | $X | $X |
| Infrastructure as % of revenue | X% | X% | X% | X% |

A written narrative accompanies the table explaining the revenue trend, any significant debt retirements that free up capacity in future years, and what the projections assume.

**Validation checkpoint:** User confirms the fiscal capacity picture before Part 4.

---

### Part 4: The Gap — The Headline Numbers

This section combines Parts 1–3 into the core findings of the report.

**The 30-year gap:**
Total obligation (from the replacement schedule + annual maintenance from Part 2) minus total projected capacity (from Part 3). Expressed three ways:
- Total dollar gap over 30 years
- Average annual gap
- Gap as a percentage of current annual revenue

**The "doing nothing" cost:**
What the deferred maintenance backlog becomes if the annual gap is left unaddressed, at the 4–5x penalty factor. Shown at 10, 20, and 30 years. This is often the most striking number in the report.

**The tax increase number:**
The single most politically legible metric:
> *"To fully fund maintenance of [City]'s existing infrastructure — not one new project, just keeping what exists — property taxes would need to increase by X% starting today."*

This is calculated from the annual gap divided by current property tax revenue, expressed as a percentage increase. Methodology is shown explicitly.

**The three-number dashboard:**

| Metric | Value | Benchmark | Rating |
|---|---|---|---|
| **Maintenance Coverage Rate** | X% | 90–100% = Healthy; 70–89% = Warning; <70% = Critical | 🔴/🟡/🟢 |
| **Deferred Maintenance Backlog** | $X | Anything > 0 signals underfunding | — |
| **Tax Increase to Break Even** | X% | Context-dependent | — |

**Red/Yellow/Green rating:**
- 🟢 **Green:** Maintenance coverage rate above 90%, deferred backlog below 10% of total asset value, tax increase to break even below 5%
- 🟡 **Yellow:** Coverage rate 70–89%, or deferred backlog 10–25% of asset value, or tax increase 5–15%
- 🔴 **Red:** Coverage rate below 70%, or deferred backlog above 25% of asset value, or tax increase above 15%

**Plain-language version of Part 4 includes:**
- A concrete household analogy calibrated to the actual numbers
- A clear statement of what "doing nothing" costs at 10, 20, and 30 years
- A one-paragraph statement suitable for reading aloud at a public meeting

---

## Input Framework

### Guided, Flexible Intake

The Claude Project does not require a specific set of documents. When a user starts a conversation, Claude:

1. Explains what the analysis will produce and approximately how long it will take
2. Asks what documents the user has available
3. Suggests the most useful document types (see below) but adapts to what's actually provided
4. Flags gaps and explains clearly what can and cannot be estimated without certain data
5. Before beginning each Part, confirms what data will be used and what will be estimated

### Most Useful Document Types

**Core (produces the most complete analysis):**
- Comprehensive Annual Financial Report (CAFR) or Annual Financial Report — especially the capital assets schedule and notes to financial statements
- Capital Improvement Plan (CIP) — proposed projects, cost estimates, funding sources
- Any asset management or infrastructure condition reports

**Helpful additions:**
- State road fund / Motor Fuel Tax allotment letters or reports
- Recent property tax levy and EAV (equalized assessed value) history
- Debt service schedules (if not in the CAFR)
- Annual operating and capital budgets
- Any FOIA responses with maintenance spending detail by department or asset class

**If limited data is available:**
Claude produces a partial analysis with clearly stated assumptions and flags which findings are data-derived vs. estimated. A partial analysis is still useful — it surfaces the right questions even when it cannot answer all of them precisely.

---

## Feasibility & Data Limitations

This section documents where the methodology is on solid ground and where significant estimation is required. These limitations are always disclosed in the Assumptions section of the consolidated report.

### High Feasibility — Data Is Reliably Available

| Data Point | Source | Notes |
|---|---|---|
| Asset replacement value (category-level) | CAFR capital assets schedule | Every CAFR includes this under GASB 34 |
| Accumulated depreciation | CAFR capital assets schedule | Required by GASB 34 |
| Estimated useful life by asset class | CAFR notes | Always disclosed alongside depreciation policy |
| Total revenue & revenue history | CAFR | Consistently formatted across municipalities |
| Existing debt & debt service | CAFR | Required disclosure |
| Property tax levy & EAV | CAFR or county assessor records | Public record in all states |
| General fiscal health trends | CAFR (5–10 years) | Foundation for Part 3 |

### Medium Feasibility — Available But Inconsistent

| Data Point | Challenge | Workaround |
|---|---|---|
| Actual maintenance spending by asset class | Many cities lump all public works spending together; road maintenance and building maintenance are not separated in the budget | Request budget detail by department/fund via FOIA; use total public works operating expenditures as a proxy with noted limitation |
| State pass-through funding by asset class | MFT allotments are published by state DOTs; federal grants are harder to track and earmark to specific asset classes | Use state DOT allotment letters for road funding; flag other grants as unverified |
| Capital Improvement Plan | Many small cities don't have a formal CIP, or have one that's a wish list without confirmed funding | If no CIP exists, note that planned projects are unknown and the analysis reflects existing obligations only |
| Construction cost inflation factors | Varies by region and asset type | Use national CPI for construction (ENR Construction Cost Index) as a conservative default; flag as an estimate |

### Low Feasibility — Significant Estimation Required

| Data Point | Challenge | How the Tool Handles It |
|---|---|---|
| Physical condition of infrastructure | Most small cities don't conduct formal engineering condition assessments. The CAFR's depreciation schedule is an accounting measure, not an engineering one — a road can be 80% depreciated on paper but physically adequate, or 40% depreciated but failing | Use depreciation percentage as a proxy with explicit caveat: "This is an accounting estimate, not an engineering assessment. A formal pavement condition index or facility condition assessment would produce more accurate results." |
| Underground utility inventory (water/sewer pipe-level detail) | Pipe-level asset data (age, material, diameter, condition by segment) is rarely public. Cities with GIS-based asset management systems have this; many do not | Work from CAFR category-level data only. Flag stormwater and underground utilities as the highest uncertainty items in the analysis. |
| Stormwater asset inventory | Often the least-documented asset class. Stormwater infrastructure is frequently omitted from CAFR capital assets schedules entirely, especially in older cities | If stormwater is absent from the CAFR, note this explicitly. The actual stormwater obligation is likely larger than reported. |
| Asset-level replacement timing | The CAFR gives category-level remaining useful life, not segment-level. "Roads" might have an average of 12 years remaining, but some segments are fine and others need replacement now | The 30-year replacement schedule uses category averages. This is appropriate for a macro fiscal analysis but understates near-term urgency for specific failing assets. |
| Fleet replacement schedule | Fleet is often reported as a single category in the CAFR. Per-vehicle replacement timing requires access to the fleet inventory | Use total fleet replacement value and average useful life (typically 10–15 years for municipal vehicles) to estimate annual replacement cost. Flag as a category estimate. |

### What This Means for the Analysis

The tool produces a **fiscal order-of-magnitude analysis**, not an engineering assessment. The right framing for every output is: *"Based on publicly available financial data, this city's infrastructure obligations are in the range of $X — likely conservative, because several asset classes are incompletely documented."*

This is still enormously valuable. Most cities have never seen their total infrastructure obligations laid out this way at all. An order-of-magnitude estimate that surfaces a $40–60 million unfunded gap is more useful for public accountability than no analysis, even if a detailed engineering assessment would put the number at $48 million or $55 million.

The limitations section of every report ends with: *"This analysis likely understates the true obligation. Underground utilities and stormwater infrastructure are the most commonly under-documented asset classes. A formal asset management study would produce a more precise figure."*

---

## Out of Scope (v1)

**New project evaluation (Part 5)** — evaluating proposed new infrastructure projects against the baseline gap is a natural v2. The value threshold metric (how much new assessed value a project must generate to cover its own maintenance obligation) will be designed after Parts 1–4 are proven in practice.

---

## Success Criteria

- A non-technical user (no financial background) can run the full analysis using publicly available documents in a single Claude conversation
- The plain-language report can be read and understood by a city council member with no preparation
- The "tax increase to break even" number is defensible, clearly sourced, and can withstand pushback
- The consolidated report can be shared publicly and the methodology can be explained and defended
- Every estimate in the report is clearly labeled as an estimate, with the source or assumption stated
