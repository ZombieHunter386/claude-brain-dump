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

## Critical Guardrails

These instructions take priority over the methodology details that follow. They prevent the most damaging errors in the analysis.

### Common Misreads from CAFRs
Claude must watch for these common errors when reading municipal financial documents:
- **Governmental vs. business-type activities:** The CAFR's government-wide statements combine both, but the capital assets schedule often separates them. Claude must use the correct column — governmental activities for general fund assets, business-type for enterprise funds — not the combined total, which would double-count
- **Capital assets vs. infrastructure assets:** Some CAFRs report "capital assets" (which includes land, equipment, and construction in progress) separately from "infrastructure" (roads, bridges, water systems). Claude should use infrastructure-specific line items when available, not total capital assets, which inflates the number with land (which doesn't depreciate or require maintenance)
- **Construction in progress:** CIP line items on the capital assets schedule represent projects under construction, not completed infrastructure. These should be excluded from the replacement value and remaining useful life calculations until they are reclassified as completed assets
- **Net pension and OPEB liabilities:** These appear on the government-wide statements and can dwarf infrastructure obligations. They are a separate fiscal issue and should not be included in the infrastructure analysis, but Claude should note their existence as context for the city's overall fiscal position

### Expenditure Classification
When calculating "available for infrastructure" in Part 3, operating expenditures include all CAFR expenditure line items EXCEPT capital outlay and debt service. This is a conservative approach — it slightly overstates operating costs (and therefore understates available infrastructure capacity) because some expenditures classified as "public works" or "culture and recreation" are actually infrastructure maintenance. Claude notes this conservative bias in the report.

### Source Tracing
Every number in the report is tagged as one of three types:
- **Extracted** — pulled directly from a specific document (cited by document name and page/section)
- **Calculated** — derived from extracted numbers using a stated formula
- **Estimated** — based on an industry benchmark or assumption (cited with source and stated rationale)

### Reasonableness Checks
Before producing the consolidated report, Claude runs the following sanity checks and flags any that fail:
- Total replacement value should fall within a reasonable range per capita (most US municipalities: $15,000–$60,000 per capita in infrastructure replacement value)
- Annual maintenance obligation should be 2–5% of total replacement value (if it falls outside this range, an input is likely wrong)
- The tax increase percentage should be checked against the city's current tax rate — an increase that would more than double the tax rate may indicate an error in inputs rather than an actual gap of that magnitude
- Revenue projections at year 30 should not exceed 3x current revenue (if they do, the growth rate assumption is likely too aggressive)

### Arithmetic Verification
Claude shows its work for every key calculation — not just the result but the formula and inputs. At each validation checkpoint, Claude presents a brief "calculation check" for the most consequential numbers: total replacement value, annual maintenance obligation, and the 30-year gap. If any number seems implausible (e.g., annual maintenance obligation exceeds total annual revenue), Claude flags it and asks the user to verify the source data before proceeding.

### Report Tone
The target reader is a city council member who is not a finance professional — someone who understands concepts like "property tax" and "debt" but not "CAGR" or "net book value." Technical terms are used where necessary but always followed by a parenthetical explanation on first use. Numbers are always accompanied by a concrete comparison (e.g., "$2.1 million per year — roughly $180 per household").

---

## The Report Structure

The report is produced **section by section** in conversation — not as a single document at the end. Each section is validated before the next one is built on top of it. This prevents compounding errors from misread documents or wrong assumptions.

At the end of the full analysis, Claude produces a **single consolidated report** (described in detail below).

---

## The Consolidated Report

The consolidated report is the final, shareable output produced after all four parts have been validated. It is a single formatted document intended to be copied, pasted, or exported — usable in a council presentation, a public meeting, or a citizen communication without further editing. The report is written in plain language throughout — technical data (tables, methodology, sources) is included but always accompanied by a clear explanation of what it means. There is one report, not separate technical and non-technical versions.

**Note on length:** The consolidated report will likely exceed a single Claude message due to the detail involved. Claude will produce it across multiple consecutive messages, clearly indicating when the report continues and when it is complete.

### Structure of the Consolidated Report

**Cover Page**
- City name
- Date of analysis
- Two or three headline numbers (from Part 4, depending on data availability): 30-year gap, Maintenance Coverage Rate (if actual spending data is available), tax increase to break even

**Executive Summary**
Plain-language summary of what the report found. Includes:
- A 2–3 sentence description of the city's infrastructure portfolio and overall fiscal condition
- The three headline numbers
- **A brief Key Data Limitations notice** — 3–5 bullet points flagging the most significant gaps or estimates in this specific analysis (e.g., "Stormwater infrastructure was not separately reported in the CAFR and is excluded from this analysis"; "Maintenance spending by asset class was estimated from total public works expenditures — a detailed breakdown was not available"). This notice is in plain language, not buried in a footnotes section, so any reader immediately understands the confidence level of the numbers they are looking at.

**Section 1 — Infrastructure Inventory** *(from Part 1)*
The full asset table. After, a plain language summary of what the full asset table is showing. 

**Section 2 — Obligations, Reserves & Deferred Impact** *(from Part 2)*
Three sub-sections:
- 2A: The annual operating maintenance table (what must be spent each year to keep infrastructure functional) with funding sources and gap
- 2B: The replacement reserve table (what must be set aside each year to fund eventual reconstruction) plus the 30-year replacement schedule
- 2C: The deferred impact comparison — for asset classes where maintenance is underfunded, a side-by-side showing what replacement costs under proper maintenance vs. continued deferral
Plus a written narrative explaining where state/federal funding applies, how much of the obligation it covers, and what is left unfunded.

**Section 3 — Fiscal Capacity** *(from Part 3)*
A written summary of the city's revenue trajectory, existing debt load, and realistic infrastructure spending capacity. Includes a table of projected available infrastructure dollars per year for the next 10/20/30 years.

**Section 4 — The Gap** *(from Part 4)*
The headline numbers with full explanation. Includes:
- The 30-year obligation vs. capacity comparison
- The "doing nothing" cost: what the deferred backlog becomes in 10, 20, and 30 years if the gap is not addressed
- The tax increase calculation with methodology shown

**Assumptions & Limitations**
A plain-language disclosure of every estimate, benchmark rate, and gap in the data. This section makes the analysis defensible: it shows the work and flags where numbers are data-derived vs. industry-standard estimates. Any estimates must include a link to the source from which they were derived. Additionally, this section provides suggestions for further studies to better understand the maintenance obligations presented.

**Internal Consistency Check**
After completing the consolidated report, Claude performs a self-check before delivering the final message: a cross-reference table listing every key figure, where it first appeared, and everywhere it is used. Claude verifies that every reference matches the original and flags any discrepancies. The cross-reference table is included at the end of the report so the user can also verify:

| Figure | Value | First Appears | Used In |
|---|---|---|---|
| Total replacement value | $X | Part 1 | Parts 2B, 4 |
| Annual operating maintenance gap | $X | Part 2A | Parts 4, sensitivity |
| Annual replacement reserve gap | $X | Part 2B | Parts 4, sensitivity |
| Available for infrastructure | $X | Part 3 | Part 4 |
| 30-year total gap | $X | Part 4 | Sensitivity, tax increase |

If any figure does not match across sections, Claude corrects the inconsistency before delivering the report.

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
- Current replacement cost (**original cost** adjusted upward for construction cost inflation from date of acquisition to present, using 3% per year default or ENR CCI if available). Note: the inflation base is original cost, not net book value — net book value subtracts depreciation, which would systematically understate what it costs to replace the asset today. For example, a road built for $5M that is 80% depreciated has a net book value of $1M, but its replacement cost is closer to $8–10M in today's dollars. **Date anchor for inflation:** The CAFR reports aggregate original cost per asset class, not individual acquisition dates. Claude uses the estimated average age (derived from depreciation percentage and useful life) as the acquisition date for the inflation adjustment. For example, if roads have a 30-year useful life and are 60% depreciated, the average age is ~18 years, so inflate from 18 years ago to present. This is an approximation — the actual portfolio contains assets of many different ages — and Claude states this assumption explicitly.
- Condition rating: if a formal inspection report exists (e.g., pavement condition index, facility condition assessment), use it. If not, use accumulated depreciation as a proxy (0–33% depreciated = Good, 34–66% = Fair, 67–100% = Poor) with the following guardrails:
  - **Check for recent capital additions:** If the CAFR shows significant recent capital spending on an asset class (additions in the last 3–5 years), the condition may be better than the depreciation percentage implies — the old asset is heavily depreciated but newly added portions are not. Claude notes when recent additions likely improve actual condition beyond what the aggregate depreciation shows.
  - **Flag where depreciation and reality are likely to diverge:** Depreciation is an accounting measure, not an engineering one. A road can be 40% depreciated on paper but physically failing due to poor subgrade, or 80% depreciated but still serviceable because it was over-engineered. Claude includes a caveat on every condition rating derived from depreciation: "This rating is based on accounting depreciation and may not reflect physical condition. A formal condition assessment would produce a more accurate rating."

**Data extraction verification:**
Before performing any calculations, Claude presents the raw numbers it extracted from the CAFR in a simple list format — asset class, original cost, accumulated depreciation, and useful life — and asks: *"Do these numbers match what you see on [page/section]?"* This catches extraction errors (misread columns, wrong table, governmental vs. business-type confusion) before they propagate through the entire analysis. This is separate from the validation checkpoint, which checks the *analysis*; this step checks the *raw data pull*.

**Output — Part 1 Asset Table:**

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

**Data carry-forward:** At the end of Part 1, Claude produces a compact DATA CARRY-FORWARD block listing every number that subsequent Parts depend on (total replacement value by asset class, remaining useful life, condition ratings, enterprise fund identification). This block is restated at the start of Part 2 so calculations don't depend on earlier conversation context being intact. The same pattern repeats at every Part transition — each Part opens by restating the carry-forward numbers from all prior Parts.

---

### Part 2: Obligations, Reserves & Deferred Impact

Part 2 answers three distinct questions:
- **2A:** What does it cost each year to keep this infrastructure functional? *(operating maintenance)*
- **2B:** How much should the city set aside each year to pay for eventual replacement? *(replacement reserve)*
- **2C:** What happens to replacement costs and timing if maintenance continues to be underfunded? *(deferred impact)*

These are different kinds of costs — 2A is an annual operating expense (money spent and gone), 2B is a capital reserve (money banked for the future), and 2C is the penalty for not doing either one properly.

#### Shared Context for All of Part 2

**Enterprise fund separation:**
Many municipalities operate water, sewer, and sometimes stormwater systems as **enterprise funds** — self-supporting operations with their own revenue streams (utility rates) separate from the general fund. Claude identifies enterprise funds by checking the CAFR's fund structure section (typically in the first 10–15 pages or the table of contents), which lists governmental funds vs. proprietary/enterprise funds. The capital assets schedule sometimes lists assets by fund type; if it doesn't, Claude maps asset classes to their fund based on the fund structure. If the fund type for a specific asset class is ambiguous, Claude asks the user.

Enterprise fund assets are separated throughout 2A, 2B, and 2C:
- Obligations are shown in all tables but clearly labeled as funded by utility rates, not property taxes
- The "tax increase to break even" number in Part 4 excludes enterprise fund obligations — those would require a utility rate increase, not a property tax increase
- If both a general fund gap and an enterprise fund gap exist, both are reported separately

**Funding source accounting:**
Claude identifies all state and federal pass-through revenues from the documents provided, maps each to the asset class it is designated for, and subtracts them from the gross obligation in 2A. Common sources include:
- Motor Fuel Tax (MFT) / state road fund allotments → Roads
- State/federal water & sewer grants → Water/Sewer
- Community Development Block Grant (CDBG) → Variable
- Federal transportation grants (CMAQ, STP, TAP) → Roads/Bridges
- State capital programs → Variable

**When actual maintenance spending is unknown:**
Many small cities cannot produce per-asset-class maintenance spending. When this data is unavailable, Claude follows this hierarchy:
1. **If a detailed operating budget is available:** Use the public works / streets / utilities operating expenditure lines as a proxy, clearly noting this includes non-maintenance costs (administration, personnel) and therefore overstates actual maintenance spending
2. **If only a summary budget is available:** Use total public works operating expenditure as an upper bound for all asset classes combined, note this in the limitations, and calculate the gap using only the total (not per-asset-class)
3. **If no maintenance spending data is available at all:** Leave the "Actual Spending" and "Gap" columns as "Unknown — insufficient data" and calculate the 30-year gap in Part 4 using gross obligation vs. fiscal capacity only
Claude always asks the user whether they can obtain this data via FOIA before falling back to estimation.

---

#### Part 2A: Annual Operating Maintenance

This is the year-to-year cost of keeping infrastructure functional right now. Pothole patching, crack sealing, snow plowing, street sweeping, pipe inspections, valve exercising, HVAC servicing, roof repairs, mowing. This is an operating expense — money spent and gone each year. If the city stops spending it, assets deteriorate faster and replacement comes sooner.

**Operating maintenance rate benchmarks by asset class:**

These rates represent practitioner estimates widely used in municipal asset management. They are not derived from a single authoritative table — no such universal standard exists — but are consistent with guidance from [APWA's asset management resources](https://www.apwa.net/My_Apwa/Resources/UserFiles/file/Utility/IntroductiontoAssetMgmt.pdf), the [ASCE 2025 Infrastructure Report Card](https://infrastructurereportcard.org/), and [AWWA's water system benchmarking program](https://www.awwa.org/programs/benchmarking/). Claude uses the midpoint of each range by default and explicitly states this assumption.

- Roads & streets: 2–4% of replacement value per year
- Water systems: 1–2% per year
- Sewer systems: 1–2% per year
- Stormwater: 1–2% per year
- Bridges: 1–2% per year
- Municipal buildings: 2–3% per year
- Parks & recreation: 1–2% per year
- Fleet & equipment: 10–15% per year (shorter useful life, high annual replacement need)

If the CIP or budget documents contain actual operating cost data for specific asset classes, Claude uses those figures instead and notes where actual data replaced the estimate.

**Output — Part 2A Operating Maintenance Table:**

| Asset Class | Annual Operating Need | State/Federal Funding | Net City Need | Actual Spending | Gap |
|---|---|---|---|---|---|
| Roads & Streets | $X | $X | $X | $X | ($X) |
| Water System *(enterprise)* | $X | $X | $X | $X | ($X) |
| Sewer System *(enterprise)* | $X | $X | $X | $X | ($X) |
| Stormwater | $X | $X | $X | $X | ($X) |
| Bridges | $X | $X | $X | $X | ($X) |
| Municipal Buildings | $X | $X | $X | $X | ($X) |
| Parks & Recreation | $X | $X | $X | $X | ($X) |
| Fleet & Equipment | $X | $X | $X | $X | ($X) |
| **TOTAL (General Fund)** | **$X** | **$X** | **$X** | **$X** | **($X)** |
| **TOTAL (Enterprise Fund)** | **$X** | **$X** | **$X** | **$X** | **($X)** |

A written narrative accompanies this table explaining: what operating maintenance means in concrete terms for each major asset class, where state/federal funding applies, how much of the obligation it covers, and what is left unfunded.

**Validation checkpoint:** User confirms the operating maintenance numbers look accurate before 2B.

---

#### Part 2B: Replacement Reserve & 30-Year Schedule

This is how much the city should be setting aside each year so that when an asset reaches end of life and needs full reconstruction, the money is there. This is a capital reserve, not an operating expense — the city doesn't spend it today, it banks it for the future.

**Calculation:** Replacement cost (from Part 1), inflated at 3% per year over remaining useful life to get the projected replacement cost at time of replacement, then divided by remaining useful life = annual reserve needed per asset class. This accounts for construction cost inflation — reserving based on today's cost would systematically underfund the eventual replacement. Claude shows this calculation explicitly (e.g., "$5M today × 1.03^15 = $7.8M at replacement → $520K/year reserve needed").

**Output — Part 2B Replacement Reserve Table:**

| Asset Class | Replacement Cost | Remaining Life | Annual Reserve Needed | Currently Budgeted (from CIP) | Reserve Gap |
|---|---|---|---|---|---|
| Roads & Streets | $X | X yrs | $X/yr | $X/yr | ($X/yr) |
| Water System *(enterprise)* | $X | X yrs | $X/yr | $X/yr | ($X/yr) |
| Sewer System *(enterprise)* | $X | X yrs | $X/yr | $X/yr | ($X/yr) |
| Stormwater | $X | X yrs | $X/yr | $X/yr | ($X/yr) |
| Bridges | $X | X yrs | $X/yr | $X/yr | ($X/yr) |
| Municipal Buildings | $X | X yrs | $X/yr | $X/yr | ($X/yr) |
| Parks & Recreation | $X | X yrs | $X/yr | $X/yr | ($X/yr) |
| Fleet & Equipment | $X | X yrs | $X/yr | $X/yr | ($X/yr) |
| **TOTAL (General Fund)** | **$X** | — | **$X/yr** | **$X/yr** | **($X/yr)** |
| **TOTAL (Enterprise Fund)** | **$X** | — | **$X/yr** | **$X/yr** | **($X/yr)** |

The "Currently Budgeted" column comes from the CIP or capital budget if available. Many small cities do not have a dedicated replacement reserve for most asset classes — if no capital set-aside can be identified, the column shows "$0 — no dedicated reserve identified," which is itself a significant finding.

**The 30-Year Replacement Schedule**

The 30-year replacement schedule shows when each asset class is projected to need full replacement and what it will cost. It is produced as a structured table formatted so it can be copied directly into Excel or Google Sheets.

The model is structured as follows:
- **Rows:** One row per asset class, plus totals rows for operating maintenance (from 2A), replacement reserve (from 2B), combined annual obligation, projected revenue (from Part 3), and net position
- **Columns:** Presented in **5-year summary buckets** (Years 1–5, 6–10, 11–15, 16–20, 21–25, 26–30, plus a Total column) so the table renders legibly in conversation. Year-by-year detail for any 5-year window is available on request. Note: the projected revenue and net position rows are populated after Part 3 is complete — the 2B validation checkpoint covers the replacement cost rows only, and the full schedule with revenue is finalized in Part 4.
- **Cell values:** The scheduled replacement cost for that asset class in that period. Because the CAFR gives *average* remaining useful life for an entire asset class (not segment-level), the replacement cost is spread across a **replacement window** centered on the average remaining life rather than placed in a single year. For example, if "Roads" has 12 years average remaining life, the replacement cost is distributed across years 8–16 (±4 years) rather than appearing as a single spike in year 12. This reflects reality — some road segments need replacement sooner and others later — and prevents a misleading spike year.
- **Net position row:** Combined obligation (operating + replacement) minus projected capacity — positive means funded, negative means gap

This model allows the user to answer: *"In years 6–10, we have $X in scheduled road reconstruction plus $Y in annual operating maintenance across all assets. Our projected revenue covers $Z. The shortfall is $W."*

Replacement costs are inflated at **3% per year by default** (consistent with long-run construction cost inflation per the [ENR Construction Cost Index](https://www.enr.com/economics)). If the CIP or provided project data contains actual cost estimates for specific asset classes, Claude uses those figures instead and notes where actual data replaced the default.

**Validation checkpoint:** User confirms the replacement reserve and 30-year schedule before 2C.

---

#### Part 2C: What Happens If Maintenance Is Deferred

For asset classes where 2A shows an operating maintenance gap, this section shows the consequences: how deferring maintenance shortens the asset's useful life and increases its eventual replacement cost. This is shown as a side-by-side comparison — not a multiplier applied to a backlog number.

**The comparison table:**

| Asset Class | Replacement Year (maintained) | Replacement Cost (maintained) | Replacement Year (deferred) | Replacement Cost (deferred) | Annual Reserve Needed (deferred) |
|---|---|---|---|---|---|
| Roads & Streets | 2046 | $X | 2038 | $X (+Y%) | $X/yr |
| Bridges | 2051 | $X | 2045 | $X (+Y%) | $X/yr |
| Water System | 2048 | $X | 2042 | $X (+Y%) | $X/yr |
| Sewer System | 2052 | $X | 2046 | $X (+Y%) | $X/yr |
| Municipal Buildings | — | — | — | — | — |
| Stormwater | — | — | — | — | — |
| Parks & Recreation | — | — | — | — | — |
| Fleet & Equipment | — | — | — | — | — |

The "maintained" columns use the scheduled replacement year and cost from 2B — this is what happens if the city starts fully funding operating maintenance today. The "deferred" columns show what happens if the current maintenance gap continues: the asset's life shortens and the replacement cost increases because the damage is more extensive (e.g., a road that needed resurfacing now needs full reconstruction including subgrade).

**Data sources and confidence levels by asset class:**

**Deferred impact formula:**
To make the 2C calculations reproducible (two users with the same data get the same answer), Claude uses a standardized formula rather than judgment-based ranges:

- **Life reduction:** For each 10% of maintenance underfunding (gap ÷ required spending from 2A), reduce remaining useful life by the asset class factor below. **Cap: life reduction is capped at 50% of remaining useful life** — even total neglect doesn't make an asset fail instantly; it accelerates failure to roughly half the designed life. If the formula produces a life reduction exceeding 50%, use 50% and note that the asset class is severely neglected.
- **Cost premium:** The replacement cost increases by the asset class factor below because deferred damage requires more extensive reconstruction (e.g., a road that needed resurfacing now needs full subgrade reconstruction). **Cap: cost premium is capped at 50%.** If the formula produces a higher premium, use 50% and note the cap.

| Asset Class | Life Reduction per 10% Underfunding | Cost Premium per 10% Underfunding | Source Confidence |
|---|---|---|---|
| Roads & Streets | 8% of remaining life | 6% cost increase | High |
| Bridges | 5% of remaining life | 4% cost increase | High |
| Water Systems | 5% of remaining life | 4% cost increase | Moderate |
| Sewer Systems | 4% of remaining life | 3% cost increase | Moderate |

*Example: If roads are 40% underfunded (spending $600K vs. $1M needed), the life reduction is 4 × 8% = 32% of remaining life, and the cost premium is 4 × 6% = 24%. If the maintained replacement year is 2046 at $5M, the deferred replacement year is ~2040 at ~$6.2M.*

These factors are calibrated from the following sources. If actual condition data is available (e.g., NBI bridge ratings), Claude uses the actual data and notes where it departs from the formula.

**Roads — High confidence.** [FHWA's pavement preservation program](https://www.fhwa.dot.gov/preservation/) provides well-documented pavement deterioration curves showing how maintenance timing affects useful life and replacement cost. [FHWA Pavement Preservation: Preserving Our Investment in Highways](https://highways.dot.gov/public-roads/januaryfebruary-2000/pavement-preservation-preserving-our-investment-highways) documents that preventive treatments are 4–5x more cost-effective than waiting for reconstruction, and that neglected roads typically lose 25–40% of their remaining useful life. The formula factors (8% life / 6% cost per 10% underfunding) are calibrated to the midpoint of these FHWA ranges.

**Bridges — High confidence.** The [FHWA National Bridge Inventory (NBI)](https://www.fhwa.dot.gov/bridge/nbi/condition.cfm) provides public, per-bridge condition ratings on a 0–9 scale, inspected on a 2-year cycle. The [NBIAS deterioration models](https://www.fhwa.dot.gov/policy/24cpr/pdf/AppendixB.pdf) provide probabilistic deterioration rates by element and climate zone. If the city's bridges are in the NBI (most are), Claude can use actual condition ratings rather than the formula. The formula factors (5% life / 4% cost) reflect that bridge deterioration is slower but rehabilitation-to-replacement escalation is costly.

**Water systems — Moderate confidence.** [EPA's 7th Drinking Water Infrastructure Needs Survey](https://www.epa.gov/dwsrf/epas-7th-drinking-water-infrastructure-needs-survey-and-assessment) documents $625 billion in national needs, two-thirds for pipe replacement. [AWWA's useful life research](https://awwa.onlinelibrary.wiley.com/doi/full/10.1002/awwa.1317) provides expected useful life by pipe material (cast iron, ductile iron, PVC) and documents that the national replacement rate (0.8%/year) is slower than the deterioration rate. The formula factors (5% life / 4% cost) are approximations — stated as such in the report.

**Sewer systems — Moderate confidence.** [EPA's sewer asset management guidance](https://www.epa.gov/sites/default/files/2015-10/documents/assetmanagement.pdf) and [EPA's residual life methodology](https://www.epa.gov/sites/default/files/2016-01/documents/determine-residual-life.pdf) provide frameworks for estimating remaining life by pipe material (concrete ~79 years, vitrified clay ~48 years). The formula factors (4% life / 3% cost) reflect that underground pipe deterioration is more gradual than surface infrastructure.

**Municipal buildings — Low confidence.** [BOMA's Building Systems Useful Life](https://www.boma.org/) data provides expected service life for major building systems (HVAC, roofing, electrical). [IFMA's Facility Condition Index (FCI)](https://knowledgelibrary.ifma.org/tag/facility-condition-assessment-fca/) provides a standard metric (deferred maintenance ÷ replacement value; above 0.10 = poor condition). However, building-specific deterioration data connecting maintenance deferral to shortened life is weak. **Claude does not populate the 2C comparison table for buildings** unless the user provides a facility condition assessment. Instead, the report states: *"Operating maintenance for municipal buildings is underfunded by $X/year. This is expected to shorten building system life and increase replacement costs, but the magnitude cannot be estimated from available data. A facility condition assessment would quantify this risk."*

**Stormwater — Low confidence.** No national inventory or condition database exists for stormwater infrastructure. EPA provides general asset management guidance but no deterioration curves. **Claude does not populate 2C for stormwater.** The report flags stormwater as the highest-uncertainty item in the analysis.

**Parks & recreation, Fleet & equipment — Low confidence.** No standard national lifecycle data sources exist for these asset classes. **Claude does not populate 2C for parks or fleet.** The report notes the gap.

**Validation checkpoint:** User confirms the deferred impact comparison before Part 3.

---

### Part 3: Fiscal Capacity

Claude reads the CAFR's revenue history, debt schedules, and tax base trend data to establish what the city can realistically spend on infrastructure over the next 30 years.

**Analysis performed:**
- **Revenue trend:** Last 5–10 years of total general fund revenue and any dedicated infrastructure fund revenue. Claude calculates the compound annual growth rate (CAGR) and projects forward using that rate — if the trend is flat or negative, this is flagged prominently. Projections assume no structural changes (no annexation, no major employer leaving or arriving, no reassessment) and longer projections carry increasing uncertainty — Claude notes this explicitly.
- **Infrastructure spending share:** What percentage of the city's total spending has gone to infrastructure (maintenance + capital) historically. This is compared to the obligation calculated in Part 2.
- **Existing debt load:** Total outstanding debt principal, annual debt service payments, and when existing debt retires. Debt service consumes revenue that cannot go to pay-as-you-go maintenance.
- **Borrowing capacity:** Estimated remaining debt capacity (most states cap municipal debt as a percentage of assessed value). This is the city's ability to finance large capital projects.
- **Dedicated infrastructure funds:** TIF districts, special service areas, water/sewer enterprise funds, and any other revenue streams legally restricted to infrastructure. These are separated from general fund capacity.
- **Operating expenditure growth:** Claude calculates the historical growth rate of operating expenditures and projects those forward alongside revenue. If operating expenditure growth exceeds revenue growth, the "available for infrastructure" residual shrinks over the projection period — this is flagged prominently because it means the infrastructure gap is widening even without any change in infrastructure costs.
- **Available for infrastructure:** This is calculated as total revenue minus total operating expenditures (personnel, administration, contractual services, commodities) minus debt service = residual available for infrastructure capital and maintenance. This is NOT revenue minus debt service alone — omitting operating expenditures would dramatically overstate the amount available. Claude uses the CAFR's expenditure summaries to determine operating costs and states the calculation explicitly.

**Output — Part 3 Summary Table:**

| Metric | Current | 10-Year Projection | 20-Year Projection | 30-Year Projection |
|---|---|---|---|---|
| Total annual revenue | $X | $X | $X | $X |
| Total operating expenditures | $X | $X | $X | $X |
| Operating expenditure growth rate | — | X% CAGR | X% CAGR | X% CAGR |
| Annual debt service | $X | $X | $X | $X |
| Available for infrastructure | $X | $X | $X | $X |
| Infrastructure as % of revenue | X% | X% | X% | X% |

A written narrative accompanies the table explaining the revenue trend, any significant debt retirements that free up capacity in future years, and what the projections assume.

**Validation checkpoint:** User confirms the fiscal capacity picture before Part 4.

---

### Part 4: The Gap — The Headline Numbers

This section combines Parts 1–3 into the core findings of the report.

**The 30-year gap:**
Total obligation (annual operating maintenance from Part 2A + scheduled replacement reserve from Part 2B) minus total projected capacity (from Part 3). Expressed three ways:
- Total dollar gap over 30 years
- Average annual gap
- Gap as a percentage of current annual revenue

**The "doing nothing" cost:**
What happens if the city continues its current maintenance spending pattern. This draws directly from the Part 2C comparison table: replacement arrives sooner and costs more for each asset class where maintenance is being deferred. Shown at 10, 20, and 30 years — the cumulative additional cost of deferred replacement across all asset classes compared to the "properly maintained" scenario. For asset classes where 2C data is unavailable (buildings, stormwater, parks, fleet), the report notes that the true "doing nothing" cost is higher than shown. This is often the most striking number in the report.

**The tax increase number:**
The single most politically legible metric:
> *"To fully fund maintenance of [City]'s existing infrastructure — not one new project, just keeping what exists — property taxes would need to increase by X% starting today."*

This is calculated from the **average annual total gap** (maintenance + scheduled replacement, averaged over 30 years) divided by current property tax revenue, expressed as a percentage increase. The average annual total gap is used rather than just the maintenance gap because replacement costs are real obligations, and rather than a single year's gap because replacement costs are lumpy (a spike in year 8 doesn't mean taxes need to spike in year 8 — it means the city needs to be setting aside that amount annually). Enterprise fund obligations are excluded — those would require a utility rate increase, reported separately. Methodology is shown explicitly.

**The three-number dashboard:**

| Metric | Value | Benchmark | Rating |
|---|---|---|---|
| **Maintenance Coverage Rate** | X% | 90–100+% = Healthy; 70–89% = Warning; <70% = Critical | — |
| **Annual Maintenance Gap** | $X/yr | $0 = fully funded | — |
| **Tax Increase to Break Even** | X% | Context-dependent | — |

**Maintenance Coverage Rate** = actual infrastructure spending (from 2A "Actual Spending" column) ÷ required infrastructure spending (from 2A "Annual Operating Need" + 2B "Annual Reserve Needed"), expressed as a percentage. This measures how much of the total obligation the city is currently covering. **Fallback:** If actual maintenance spending data is unavailable, the Maintenance Coverage Rate is reported as "Unknown — actual spending data not available" and the dashboard uses the Annual Maintenance Gap and Tax Increase as the two headline numbers instead. The cover page adapts to show only metrics that can be calculated from available data.

**Annual Maintenance Gap** = total required spending (from 2A "Annual Operating Need" + 2B "Annual Reserve Needed") minus the city's current infrastructure allocation. If actual per-asset-class spending data is available, use the sum of 2A and 2B gaps. If actual spending is unknown, use Part 3's "available for infrastructure" as the current allocation instead — this is always computable from the CAFR. This is the dollar amount the city is falling short each year.

**Sensitivity analysis:**
The headline gap number depends on assumptions — maintenance rates, inflation, and revenue growth projections. To make the report defensible under scrutiny, Claude produces a simple sensitivity table showing what happens when key assumptions move:

| Scenario | Maintenance Rate | Inflation | Revenue Growth | 30-Year Gap |
|---|---|---|---|---|
| Conservative (low estimate) | Low end of range | 2% | High end of historical CAGR | $X |
| **Base case (report default)** | **Midpoint** | **3%** | **Historical CAGR** | **$X** |
| Aggressive (high estimate) | High end of range | 4% | Flat (0%) | $X |

This prevents the report from being dismissed over any single assumption. The framing is: *"Even under the most optimistic assumptions, the gap is $X. Under more realistic assumptions, it's $Y."*

**Data completeness note:** The sensitivity table captures uncertainty in assumptions (rates, inflation, growth). It does NOT capture uncertainty in the underlying data. If major asset classes are missing or incompletely reported (see Assumptions & Limitations), the true obligation could be significantly higher than even the aggressive scenario shows. Claude adds a note below the sensitivity table stating which asset classes are incompletely documented and what direction that biases the estimate (always upward — missing data means missing obligations).

**Plain-language summary of Part 4 includes:**
- A per-household cost: the annual gap divided by the number of households. Claude asks the user for the number of households in the city (or number of taxable residential parcels if they have it). If the user doesn't have this, Claude notes the per-household figure is omitted and the user can calculate it later
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

- A non-technical user (no financial background) can run the full analysis using publicly available documents. For smaller cities with compact CAFRs, this may complete in a single conversation. For larger document sets, the data carry-forward pattern and handoff summary enable seamless continuation across conversations — Claude summarizes what has been established so far and provides a carry-forward block the user can paste into a new conversation to continue
- The plain-language report can be read and understood by a city council member with no preparation
- The "tax increase to break even" number is defensible, clearly sourced, and can withstand pushback
- The consolidated report can be shared publicly and the methodology can be explained and defended
- Every estimate in the report is clearly labeled as an estimate, with the source or assumption stated
