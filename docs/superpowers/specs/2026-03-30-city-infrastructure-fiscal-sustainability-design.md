# City Infrastructure Fiscal Sustainability Tool — Design Spec
**Date:** 2026-03-30
**Project:** Infrastructure Fiscal Sustainability Analysis
**Status:** Approved for implementation
**First application:** Oak Forest, IL

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

### Relationship to ST Financial Decoder
Run the Decoder first to get the macro fiscal health picture. Then use this tool for the infrastructure-specific deep dive. The two tools are complementary, not redundant.

---

## The Report Structure

The report is produced **section by section** in conversation — not as a single document at the end. Each section is validated before the next one is built on top of it. This prevents compounding errors from misread documents or wrong assumptions.

At the end of the full analysis, Claude produces a **single consolidated report** the user can share, copy into a presentation, or hand to a council member. Two versions are always produced:
- **Technical version** — numbers, methodology, data sources, assumptions stated explicitly
- **Plain-language version** — same findings written for a council member or resident, using concrete analogies

---

### Part 1: Infrastructure Inventory & Asset Health

Claude reads the capital assets schedule (typically found in the CAFR's notes to financial statements) and any available CIP, asset management plans, or inspection reports to build a picture of what the city owns.

**For each asset class** (roads, water/sewer, bridges, parks & recreation facilities, municipal buildings, fleet/equipment, stormwater):
- Replacement value (what it would cost to rebuild today)
- Remaining useful life (years before replacement is needed)
- Condition rating (if available from inspections; otherwise estimated from age and depreciation data)

**Output:** A table showing the city's full infrastructure portfolio, sorted by remaining useful life. Example:

> *"Oak Forest owns an estimated $X in road infrastructure with an average remaining useful life of 12 years, $Y in water/sewer infrastructure with 22 years remaining..."*

**Validation checkpoint:** User confirms the asset picture looks accurate before Part 2 runs.

---

### Part 2: Annual Maintenance Obligation & Funding Coverage

Using industry-standard maintenance rates for each asset class, Claude calculates what the city *should* be spending annually versus what it's actually spending — and maps all dedicated funding sources against those obligations.

**Maintenance rate benchmarks by asset class:**
- Roads: 2–4% of replacement value per year
- Water/sewer systems: 1–2% per year
- Bridges: 1–2% per year
- Buildings: 2–3% per year
- Parks & recreation: 1–2% per year
- Fleet/equipment: 5–10% per year (shorter useful life)

**Funding source accounting:**
All state and federal pass-through funding is identified, mapped to the asset class it's designated for, and subtracted from the gross obligation. In Illinois this includes Motor Fuel Tax (MFT) revenue, IDOT allocations, CDBG, and federal transportation grants. The output shows:

- Gross annual maintenance obligation (what should be spent)
- State/federal funding covering that obligation
- Net annual maintenance obligation (what the city itself must fund)
- Actual maintenance spending (what is being spent)
- **Annual maintenance gap** (net obligation minus actual spending)

**30-year replacement schedule:**
A year-by-year projection of when major assets need replacement and what that costs, based on remaining useful life from Part 1.

**Deferred maintenance backlog:**
If the city has been underspending for years, this is quantified and compounded. Deferred maintenance doesn't disappear — it accelerates deterioration and increases future replacement costs.

**Output example:**

> *"Oak Forest should be spending $X/year on road maintenance. It receives $Y/year in MFT revenue, covering Z% of that obligation. The remaining $W has no dedicated funding source. Actual spending is $V — leaving an annual gap of $U that is being quietly added to the deferred maintenance backlog."*

**Validation checkpoint:** User confirms the funding source mapping and gap calculation before Part 3.

---

### Part 3: Fiscal Capacity

Claude reads the CAFR's revenue history, debt schedules, and tax base trend data to establish what the city can realistically spend on infrastructure over the next 30 years.

**Key inputs:**
- General fund revenue trend (last 5–10 years)
- Property tax base (assessed value trend)
- Existing debt obligations and their payoff schedule
- Any dedicated infrastructure funds (TIF districts, special service areas, etc.)

**Output:**
- Revenue trajectory — is the tax base growing, flat, or shrinking?
- Debt capacity — how much of the city's borrowing capacity is already committed?
- Available infrastructure capacity — dollars realistically available for maintenance and capital after existing obligations

**Validation checkpoint:** User confirms the fiscal capacity picture before Part 4.

---

### Part 4: The Gap — The Headline Numbers

This section combines Parts 1–3 into the core findings. It produces the numbers that go on the cover page of the report.

**The 30-year gap:**
> *"Oak Forest has $X in infrastructure obligations over the next 30 years. Based on current revenue trends, it has capacity to spend $Y on infrastructure. That leaves a gap of $Z — infrastructure the city has committed to maintain but has no plan to fund."*

**The tax increase number:**
The single most politically legible metric in the report:
> *"To fully fund maintenance of Oak Forest's existing infrastructure — not one new project, just keeping what exists — property taxes would need to increase by X% starting today."*

**The three-number dashboard:**

| Metric | Definition | What It Signals |
|---|---|---|
| **Maintenance Coverage Rate** | Actual maintenance spending ÷ required maintenance spending | Are you keeping up? (100% = fully funded; below 80% = deteriorating) |
| **Deferred Maintenance Backlog** | Cumulative underspending, compounded | What's already overdue and getting more expensive |
| **Tax Increase to Break Even** | % property tax increase needed to fully fund existing obligations | The honest political number |

**Plain-language version includes:**
- A concrete analogy scaled to the numbers (e.g., *"This is like owning a house and only budgeting for half the maintenance it needs. The house doesn't fall apart immediately — but every year you skip, the eventual repair gets more expensive."*)
- A red/yellow/green fiscal health rating for the infrastructure portfolio overall
- A clear statement of what "doing nothing" means in dollar terms over 10, 20, and 30 years

---

## Input Framework

### Guided, Flexible Intake

The Claude Project does not require a specific set of documents. When a user starts a conversation, Claude:

1. Explains what the analysis will produce
2. Asks what documents the user has available
3. Suggests the most useful document types (see below) but adapts to what's actually provided
4. Flags gaps and explains what can and can't be estimated without certain data

### Most Useful Document Types

**Core (produces the most complete analysis):**
- Comprehensive Annual Financial Report (CAFR) or Annual Financial Report — especially the capital assets schedule and notes to financial statements
- Capital Improvement Plan (CIP) — proposed projects, cost estimates, funding sources
- Any asset management or infrastructure condition reports

**Helpful additions:**
- Motor Fuel Tax reports or state funding allocation letters
- Recent property tax levy and EAV (equalized assessed value) history
- Debt service schedules
- Budget documents (operating and capital)
- Any FOIA responses with maintenance spending detail

**If limited data is available:**
Claude can produce a partial analysis with clearly stated assumptions and flag which findings are estimates vs. data-derived. A partial analysis is still useful — it surfaces the right questions even when it can't answer all of them precisely.

---

## First Application: Oak Forest, IL

Oak Forest is a south-suburban Chicago municipality and the first city this tool will be applied to. It is a typical aging inner-ring suburb with a stable but not growing tax base, aging road and utility infrastructure, and the same structural fiscal pressures facing most Midwest municipalities.

The Oak Forest application will:
1. Test the intake process against real, publicly available Illinois municipal documents
2. Identify gaps in the methodology where document formats don't fit expectations
3. Produce a real Fiscal Sustainability Report that can be presented to the Oak Forest city council

Documents to gather for Oak Forest:
- Oak Forest CAFR (available from the city or GFOA award database)
- Oak Forest CIP (public or FOIA)
- Illinois MFT allotment history (available from IDOT)
- Cook County/Oak Forest EAV and levy history (available from Cook County Assessor)

---

## Out of Scope (v1)

**New project evaluation (Part 5)** — evaluating proposed new infrastructure projects against the baseline gap is a natural v2. The value threshold metric (how much new assessed value a project must generate to cover its maintenance obligation) will be designed after Parts 1–4 are proven on Oak Forest.

---

## Success Criteria

- A non-technical user (no financial background) can run the full analysis using publicly available documents in a single Claude conversation
- The plain-language report can be read and understood by a city council member with no preparation
- The "tax increase to break even" number is defensible, clearly sourced, and can withstand pushback
- The Oak Forest report is accurate enough to present publicly and stand behind
