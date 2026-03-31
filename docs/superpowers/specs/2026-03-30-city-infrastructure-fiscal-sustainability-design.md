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

The consolidated report is the final, shareable output produced after all four parts have been validated. It is a single formatted document intended to be copied, pasted, or exported — usable in a council presentation, a public meeting, or a citizen communication without further editing. The report is written in plain language throughout — technical data (tables, methodology, sources) is included but always accompanied by a clear explanation of what it means. There is one report, not separate technical and non-technical versions.

### Structure of the Consolidated Report

**Cover Page**
- City name
- Date of analysis
- Three headline numbers (from Part 4): 30-year gap, maintenance coverage rate, tax increase to break even

**Executive Summary**
Plain-language summary of what the report found. Includes:
- A 2–3 sentence description of the city's infrastructure portfolio and overall fiscal condition
- The three headline numbers
- The red/yellow/green rating with a one-sentence explanation
- **A brief Key Data Limitations notice** — 3–5 bullet points flagging the most significant gaps or estimates in this specific analysis (e.g., "Stormwater infrastructure was not separately reported in the CAFR and is excluded from this analysis"; "Maintenance spending by asset class was estimated from total public works expenditures — a detailed breakdown was not available"). This notice is in plain language, not buried in a footnotes section, so any reader immediately understands the confidence level of the numbers they are looking at.

**Section 1 — Infrastructure Inventory** *(from Part 1)*
The full asset excel table. After, a plain language summary of what the full asset excel table is showing. 

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
A plain-language disclosure of every estimate, benchmark rate, and gap in the data. This section makes the analysis defensible: it shows the work and flags where numbers are data-derived vs. industry-standard estimates. Any estimates must include a link to the source from which they were derived. Additionally, this section provides suggestions for further studies to better understand the maintenaince obligations presented. 

---

### Part 1: Infrastructure Inventory & Asset Health

Claude reads the capital assets schedule (typically in the CAFR's notes to financial statements) and any available CIP, asset management plans, or inspection reports.

**Analysis performed:**
For each asset class (roads, water/sewer, bridges, parks & recreation facilities, municipal buildings, fleet/equipment, stormwater), Claude extracts or estimates:
- Original cost (from CAFR capital assets schedule)
- Accumulated depreciation (from CAFR)
- Net book value (original cost minus accumulated depreciation)
- Estimated useful life (from CAFR notes — e.g., roads = 20–40 years)
- Estimated age (derived from useful life and depreciation percentage). Claude first confirms the depreciation method from the CAFR notes — most municipalities use straight-line, but if the method is unclear or different, the age estimate is flagged as less reliable. Claude should describe how it makes all estimates
- Remaining useful life in years
- Current replacement cost (**original cost** adjusted upward for construction cost inflation from date of acquisition to present, using 3% per year default or ENR CCI if available). Note: the inflation base is original cost, not net book value — net book value subtracts depreciation, which would systematically understate what it costs to replace the asset today. For example, a road built for $5M that is 80% depreciated has a net book value of $1M, but its replacement cost is closer to $8–10M in today's dollars.
- Condition rating: if a formal inspection report exists (e.g., pavement condition index, facility condition assessment), use it. If not, use accumulated depreciation as a proxy (0–33% depreciated = Good, 34–66% = Fair, 67–100% = Poor) with the following guardrails:
  - **Check for recent capital additions:** If the CAFR shows significant recent capital spending on an asset class (additions in the last 3–5 years), the condition may be better than the depreciation percentage implies — the old asset is heavily depreciated but newly added portions are not. Claude notes when recent additions likely improve actual condition beyond what the aggregate depreciation shows.
  - **Flag where depreciation and reality are likely to diverge:** Depreciation is an accounting measure, not an engineering one. A road can be 40% depreciated on paper but physically failing due to poor subgrade, or 80% depreciated but still serviceable because it was over-engineered. Claude includes a caveat on every condition rating derived from depreciation: "This rating is based on accounting depreciation and may not reflect physical condition. A formal condition assessment would produce a more accurate rating."

**Output — Part 1 Excel Table:**

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

Using industry-standard maintenance rate benchmarks which are cited, Claude calculates what the city *should* be spending annually versus what it's actually spending, accounts for all dedicated funding sources, and builds the 30-year replacement schedule.

**Maintenance rate benchmarks by asset class:**

These rates represent practitioner estimates widely used in municipal asset management. They are not derived from a single authoritative table — no such universal standard exists — but are consistent with guidance from [APWA's asset management resources](https://www.apwa.net/My_Apwa/Resources/UserFiles/file/Utility/IntroductiontoAssetMgmt.pdf), the [ASCE 2025 Infrastructure Report Card](https://infrastructurereportcard.org/), and [AWWA's water system benchmarking program](https://www.awwa.org/programs/benchmarking/). Claude uses the midpoint of each range by default and explicitly states this assumption.

- Roads & streets: 2–4% of replacement value per year
- Water systems: 1–2% per year
- Sewer systems: 1–2% per year
- Stormwater: 1–2% per year
- Bridges: 1–2% per year
- Municipal buildings: 2–3% per year
- Parks & recreation: 1–2% per year
- Fleet & equipment: 10–15% per year (shorter useful life, high annual replacement need)

If the CIP or budget documents contain actual unit cost data for specific asset classes, Claude uses those figures instead of the benchmark rates and notes where actual data replaced the estimate.

**When actual maintenance spending is unknown:**
Many small cities cannot produce per-asset-class maintenance spending without a forensic audit of their public works budget. When this data is unavailable, Claude follows this hierarchy:
1. **If a detailed operating budget is available:** Use the public works / streets / utilities operating expenditure lines as a proxy, clearly noting this includes non-maintenance costs (administration, personnel) and therefore overstates actual maintenance spending
2. **If only a summary budget is available:** Use total public works operating expenditure as an upper bound for all asset classes combined, note this in the limitations, and calculate the gap using only the total (not per-asset-class)
3. **If no maintenance spending data is available at all:** Leave the "Actual Spending" and "Annual Gap" columns as "Unknown — insufficient data" in the maintenance gap table, and calculate the 30-year gap in Part 4 using gross obligation vs. fiscal capacity only. The report notes that actual spending may partially offset the gap but cannot be verified from available documents
Claude always asks the user whether they can obtain this data via FOIA before falling back to estimation.

**Enterprise fund separation:**
Many municipalities operate water, sewer, and sometimes stormwater systems as **enterprise funds** — self-supporting operations with their own revenue streams (utility rates) separate from the general fund. Claude identifies enterprise funds by checking the CAFR's fund structure section (typically in the first 10–15 pages or the table of contents), which lists governmental funds vs. proprietary/enterprise funds. The capital assets schedule sometimes lists assets by fund type; if it doesn't, Claude maps asset classes to their fund based on the fund structure. If the fund type for a specific asset class is ambiguous, Claude asks the user. When enterprise fund assets are identified, Claude separates them throughout the analysis:
- Enterprise fund obligations are shown in the maintenance gap table but clearly labeled as funded by utility rates, not property taxes
- The "tax increase to break even" number in Part 4 excludes enterprise fund obligations — those would require a utility rate increase, not a property tax increase
- If both a general fund gap and an enterprise fund gap exist, both are reported separately with their respective funding mechanisms
- This prevents the report from inflating the property tax number with obligations that are (or should be) covered by a different revenue stream

**Funding source accounting:**
Claude identifies all state and federal pass-through revenues from the documents provided, maps each to the asset class it is designated for, and calculates net obligation. Common sources include:
- Motor Fuel Tax (MFT) / state road fund allotments → Roads
- State/federal water & sewer grants → Water/Sewer
- Community Development Block Grant (CDBG) → Variable
- Federal transportation grants (CMAQ, STP, TAP) → Roads/Bridges
- State capital programs → Variable

**Output — Part 2 Maintenance Gap Excel Table:**

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
- **Columns:** In the conversation, the schedule is presented in **5-year summary buckets** (6 columns: Years 1–5, 6–10, 11–15, 16–20, 21–25, 26–30, plus a Total column) so the table renders legibly. Year-by-year detail for any 5-year window is available on request. When the user copies the table to Excel or Google Sheets, Claude can produce the full 30-column year-by-year version on request.
- **Cell values:** The scheduled replacement cost for that asset class in that year, calculated from remaining useful life in Part 1. When an asset class hits the end of its useful life, its full replacement value appears in that year's cell.
- **Annual maintenance row:** Below the replacement schedule, a row for annual maintenance obligation (from the gap table above) is added for each year, allowing the user to see the combined annual infrastructure cost (maintenance + scheduled replacement) for any given year
- **Revenue row:** The city's projected annual infrastructure spending capacity (from Part 3) is shown as a comparison row
- **Net position row:** Annual obligation minus projected capacity — positive means funded, negative means gap

This model allows the user to answer: *"In year 7, we have $X in scheduled road reconstruction plus $Y in annual maintenance across all assets. Our projected revenue covers $Z. The shortfall is $W."*

The model uses two clearly stated assumptions that Claude discloses: (1) replacement costs are inflated at **3% per year by default** (consistent with long-run construction cost inflation as tracked by the [ENR Construction Cost Index](https://www.enr.com/economics)); however, if the CIP or provided project data contains actual cost estimates for specific asset classes, Claude uses those figures instead and notes where actual data replaced the default rate; (2) assets are replaced exactly at end of useful life, which is conservative — deferred replacement means higher costs.

**Deferred maintenance backlog:**
If the city has been underspending on annual maintenance, Claude calculates the compounding backlog. Deferred maintenance doesn't disappear — when preventive work is skipped, assets deteriorate faster and repairs become more expensive.

The cost multiplier varies by asset type and the evidence is strongest for roads:
- **Roads:** [FHWA's pavement preservation guidance](https://www.fhwa.dot.gov/preservation/) documents that every $1 spent on preventive pavement maintenance saves up to $6–$14 in future reconstruction costs, depending on when in the pavement life cycle the maintenance is applied. The [FHWA Pavement Preservation: Preserving Our Investment in Highways](https://highways.dot.gov/public-roads/januaryfebruary-2000/pavement-preservation-preserving-our-investment-highways) article states preventive treatments are 4–5x more cost-effective than waiting for reconstruction.
- **Buildings and other assets:** A commonly cited figure in facilities management literature is a 4:1 ratio (James Piper, P.E., writing in [FacilitiesNet](https://www.facilitiesnet.com/maintenanceoperations/article/5-Hidden-Costs-of-Deferring-Maintenance--19388)), though this figure lacks a traceable primary source and should be treated as a practitioner estimate rather than a research finding.

**Default assumption:** Claude uses a **3x multiplier** for non-road assets (conservative) and a **5x multiplier** for roads (mid-range of the FHWA-documented range). Both are stated explicitly in the report with sources. If the user has reason to believe a different multiplier is more appropriate for their city, they can override it and Claude will restate the assumption.

**Compounding methodology:** The deferred backlog is calculated as: cumulative underspending (sum of each year's maintenance gap) × the asset-specific multiplier, calculated once as a total liability. It is NOT compounded year-over-year (applying the multiplier repeatedly to a growing balance) — that would produce absurdly large numbers that undermine credibility. The multiplier represents the eventual cost of repair when deferred maintenance leads to asset failure, not an annual compounding rate.

**Double-counting caveat:** The 30-year total obligation (Part 4) adds annual maintenance obligations to scheduled replacement costs. In practice, these partially overlap: maintenance spending is intended to *prevent* premature replacement, and underfunded maintenance accelerates replacement timelines. The deferred maintenance multiplier captures this relationship approximately, but the total should be understood as an upper-bound estimate. The report frames it as: *"If the city maintains its infrastructure on schedule AND replaces it at end of useful life, the total cost is $X. If maintenance is deferred, some replacement costs arrive sooner and cost more — captured by the deferred backlog estimate."*

**Validation checkpoint:** User confirms the maintenance gap table and replacement schedule before Part 3.

---

### Part 3: Fiscal Capacity

Claude reads the CAFR's revenue history, debt schedules, and tax base trend data to establish what the city can realistically spend on infrastructure over the next 30 years.

**Analysis performed:**
- **Revenue trend:** Last 5–10 years of total general fund revenue and any dedicated infrastructure fund revenue. Claude calculates the compound annual growth rate (CAGR) and projects forward using that rate — if the trend is flat or negative, this is flagged prominently. Projections assume no structural changes (no annexation, no major employer leaving or arriving, no reassessment) and longer projections carry increasing uncertainty — Claude notes this explicitly.
- **Infrastructure spending share:** What percentage of the city's total spending has gone to infrastructure (maintenance + capital) historically. This is compared to the obligation calculated in Part 2.
- **Existing debt load:** Total outstanding debt principal, annual debt service payments, and when existing debt retires. Debt service consumes revenue that cannot go to pay-as-you-go maintenance.
- **Borrowing capacity:** Estimated remaining debt capacity (most states cap municipal debt as a percentage of assessed value). This is the city's ability to finance large capital projects.
- **Dedicated infrastructure funds:** TIF districts, special service areas, water/sewer enterprise funds, and any other revenue streams legally restricted to infrastructure. These are separated from general fund capacity.
- **Available for infrastructure:** This is calculated as total revenue minus total operating expenditures (personnel, administration, contractual services, commodities) minus debt service = residual available for infrastructure capital and maintenance. This is NOT revenue minus debt service alone — omitting operating expenditures would dramatically overstate the amount available. Claude uses the CAFR's expenditure summaries to determine operating costs and states the calculation explicitly.

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
What the deferred maintenance backlog becomes if the annual gap is left unaddressed, using the asset-specific multipliers defined in Part 2 (5x for roads per FHWA; 3x for other assets). Shown at 10, 20, and 30 years. This is often the most striking number in the report.

**The tax increase number:**
The single most politically legible metric:
> *"To fully fund maintenance of [City]'s existing infrastructure — not one new project, just keeping what exists — property taxes would need to increase by X% starting today."*

This is calculated from the **average annual total gap** (maintenance + scheduled replacement, averaged over 30 years) divided by current property tax revenue, expressed as a percentage increase. The average annual total gap is used rather than just the maintenance gap because replacement costs are real obligations, and rather than a single year's gap because replacement costs are lumpy (a spike in year 8 doesn't mean taxes need to spike in year 8 — it means the city needs to be setting aside that amount annually). Enterprise fund obligations are excluded — those would require a utility rate increase, reported separately. Methodology is shown explicitly.

**The three-number dashboard:**

| Metric | Value | Benchmark | Rating |
|---|---|---|---|
| **Maintenance Coverage Rate** | X% | 90–100+% = Healthy; 70–89% = Warning; <70% Critical
| **Deferred Maintenance Backlog** | $X | Anything > 0 signals underfunding | — |
| **Tax Increase to Break Even** | X% | Context-dependent | — |

**Sensitivity analysis:**
The headline gap number depends on assumptions — maintenance rates, inflation, deferred maintenance multipliers. To make the report defensible under scrutiny, Claude produces a simple sensitivity table showing what happens when key assumptions move:

| Scenario | Maintenance Rate | Inflation | Revenue Growth | 30-Year Gap |
|---|---|---|---|---|
| Conservative (low estimate) | Low end of range | 2% | High end of historical CAGR | $X |
| **Base case (report default)** | **Midpoint** | **3%** | **Historical CAGR** | **$X** |
| Aggressive (high estimate) | High end of range | 4% | Flat (0%) | $X |

This prevents the report from being dismissed over any single assumption. The framing is: *"Even under the most optimistic assumptions, the gap is $X. Under more realistic assumptions, it's $Y."*

**Plain-language summary of Part 4 includes:**
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

## Error Mitigation

The following safeguards are built into the analysis process to catch errors before they reach the final report.

### Arithmetic Verification
Claude shows its work for every key calculation — not just the result but the formula and inputs. At each validation checkpoint, Claude presents a brief "calculation check" for the most consequential numbers: total replacement value, annual maintenance obligation, and the 30-year gap. If any number seems implausible (e.g., annual maintenance obligation exceeds total annual revenue), Claude flags it and asks the user to verify the source data before proceeding.

### Source Tracing
Every number in the report is tagged as one of three types:
- **Extracted** — pulled directly from a specific document (cited by document name and page/section)
- **Calculated** — derived from extracted numbers using a stated formula
- **Estimated** — based on an industry benchmark or assumption (cited with source and stated rationale)

This makes errors traceable. If the final gap number seems wrong, you can follow each input back to its source and find where the error entered.

### Reasonableness Checks
Before producing the consolidated report, Claude runs the following sanity checks and flags any that fail:
- Total replacement value should fall within a reasonable range per capita (most US municipalities: $15,000–$60,000 per capita in infrastructure replacement value)
- Annual maintenance obligation should be 2–5% of total replacement value (if it falls outside this range, an input is likely wrong)
- The tax increase percentage should be checked against the city's current tax rate — an increase that would more than double the tax rate may indicate an error in inputs rather than an actual gap of that magnitude
- Revenue projections at year 30 should not exceed 3x current revenue (if they do, the growth rate assumption is likely too aggressive)

### Common Misreads from CAFRs
Claude is specifically instructed to watch for these common errors when reading municipal financial documents:
- **Governmental vs. business-type activities:** The CAFR's government-wide statements combine both, but the capital assets schedule often separates them. Claude must use the correct column — governmental activities for general fund assets, business-type for enterprise funds — not the combined total, which would double-count
- **Capital assets vs. infrastructure assets:** Some CAFRs report "capital assets" (which includes land, equipment, and construction in progress) separately from "infrastructure" (roads, bridges, water systems). Claude should use infrastructure-specific line items when available, not total capital assets, which inflates the number with land (which doesn't depreciate or require maintenance)
- **Construction in progress:** CIP line items on the capital assets schedule represent projects under construction, not completed infrastructure. These should be excluded from the replacement value and remaining useful life calculations until they are reclassified as completed assets
- **Net pension and OPEB liabilities:** These appear on the government-wide statements and can dwarf infrastructure obligations. They are a separate fiscal issue and should not be included in the infrastructure analysis, but Claude should note their existence as context for the city's overall fiscal position

---

## Out of Scope (v1)

**New project evaluation (Part 5)** — evaluating proposed new infrastructure projects against the baseline gap is a natural v2. The value threshold metric (how much new assessed value a project must generate to cover its own maintenance obligation) will be designed after Parts 1–4 are proven in practice.

---

## Success Criteria

- A non-technical user (no financial background) can run the full analysis using publicly available documents, ideally in a single Claude conversation. If document volume exceeds context limits (large CAFRs + CIP + budget documents), the system prompt instructs Claude to summarize what has been established so far and provide a handoff summary the user can paste into a new conversation to continue seamlessly
- The plain-language report can be read and understood by a city council member with no preparation
- The "tax increase to break even" number is defensible, clearly sourced, and can withstand pushback
- The consolidated report can be shared publicly and the methodology can be explained and defended
- Every estimate in the report is clearly labeled as an estimate, with the source or assumption stated
