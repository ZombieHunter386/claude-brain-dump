# Personal CRM — Foundation Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a fully-structured Professional Contacts Notion database with all 15 fields, 6 views, and a Job Target profile page — ready to receive LinkedIn connections via CSV import.

**Architecture:** Single Notion database ("Professional Contacts") with properties covering identity, professional info, interests, and relationship tracking. Six curated views optimize for browsing, follow-up, birthdays, industry grouping, recent events, and job search. A separate Notion page ("My Job Target Profile") stores job search context Claude reads during queries. A local reference file tracks all Notion IDs for use in future automation plans (Plans 2–5).

**Tech Stack:** Notion MCP (notion-create-database, notion-create-view, notion-create-pages, notion-fetch, notion-search), LinkedIn CSV export

---

## Chunk 1: Locate Parent Page + Create Database

### Task 1: Find or Confirm Parent Page in Notion

**Files:**
- Create: `docs/personal-crm/notion-ids.md` — reference file for all Notion IDs created across all CRM plans

- [ ] **Step 1: Search for existing workspace pages**

Use `notion-search` with query "" (empty) to list top-level pages and find a suitable parent (e.g., a "Personal", "Tools", or workspace root page).

- [ ] **Step 2: Confirm parent location with user**

Ask: "I found these top-level pages in your Notion: [list]. Which should be the parent for the CRM, or should I create a new top-level page?"

- [ ] **Step 3: Record parent page ID**

Create `docs/personal-crm/notion-ids.md`:

```markdown
# Personal CRM — Notion IDs

_Reference file for all Notion IDs used across CRM automation plans._

## Parent Page
- **ID:** [parent-page-id]
- **Name:** [page name]

## Contacts Database
- **ID:** (fill after Task 2)
- **Name:** Professional Contacts

## Job Target Profile Page
- **ID:** (fill after Task 9)
- **Name:** My Job Target Profile

## Views (fill during Tasks 3–8)
- **All Contacts:** [view-id]
- **Follow-up Queue:** [view-id]
- **Birthdays:** [view-id]
- **By Industry:** [view-id]
- **Recent Events:** [view-id]
- **Job Search:** [view-id]
```

---

### Task 2: Create the Contacts Database

**Files:**
- Modify: `docs/personal-crm/notion-ids.md` — add database ID

- [ ] **Step 1: Define the expected schema**

| Property | Notion Type | Notes |
|---|---|---|
| Name | title | Required, auto-present |
| LinkedIn URL | url | |
| Connection Detail | rich_text | e.g., "University of Illinois 2019" |
| Current Title | rich_text | |
| Company | rich_text | |
| Industry | multi_select | Seeds: Tech, Real Estate, Finance, Healthcare, Legal, Marketing, Sales, Consulting, Other |
| Professional Expertise | multi_select | Empty — user/Claude populated |
| Personal Interests | multi_select | Empty — user/Claude populated |
| Birthday | date | |
| Last Contacted | date | |
| Latest Notable Event | rich_text | e.g., "Promoted to VP Sales at Acme" |
| Latest Event Date | date | |
| Open Job Posting URL | url | LinkedIn job posting link |
| Follow-up Frequency | select | Options: Monthly, Quarterly, Annually, As-Needed |
| Notes | rich_text | Free text |

- [ ] **Step 2: Create the database via MCP**

Call `notion-create-database` with:
- Parent: [parent-page-id from Task 1]
- Title: "Professional Contacts"
- All 15 properties with correct types and seed options as defined above

- [ ] **Step 3: Verify database was created correctly**

Call `notion-fetch` on the returned database ID. Confirm:
- Title is "Professional Contacts"
- All 15 properties exist with correct types
- `Industry` multi_select has all 9 seed options
- `Follow-up Frequency` select has all 4 options

- [ ] **Step 4: Record database ID**

Update `docs/personal-crm/notion-ids.md` — fill in the Contacts Database ID.

- [ ] **Step 5: Commit**

```bash
git add docs/personal-crm/notion-ids.md
git commit -m "feat: create Professional Contacts Notion database with 15 fields"
```

---

## Chunk 2: Views

### Task 3: All Contacts View (Default Table)

- [ ] **Step 1: Create view via MCP**

Call `notion-create-view` with:
- Database: [database-id]
- Type: table
- Name: "All Contacts"
- Visible properties: Name, Current Title, Company, Industry, Last Contacted, Follow-up Frequency

- [ ] **Step 2: Verify**

Call `notion-fetch` on the database. Confirm "All Contacts" table view is present.

---

### Task 4: Follow-up Queue View

- [ ] **Step 1: Create view via MCP**

Call `notion-create-view` with:
- Type: table
- Name: "Follow-up Queue"
- Visible properties: Name, Current Title, Company, Last Contacted, Follow-up Frequency, Latest Notable Event
- Sort: Last Contacted ascending (longest overdue first)

- [ ] **Step 2: Document manual filter step**

The dynamic filter (overdue based on Follow-up Frequency) cannot be expressed via MCP alone. After creation, document in `docs/personal-crm/notion-ids.md`:

```markdown
## Manual Setup Required
### Follow-up Queue view filter
Open the "Follow-up Queue" view in Notion UI and add this filter group:
- OR group 1: Follow-up Frequency = "Monthly" AND Last Contacted < 30 days ago
- OR group 2: Follow-up Frequency = "Quarterly" AND Last Contacted < 90 days ago
- OR group 3: Follow-up Frequency = "Annually" AND Last Contacted < 365 days ago
```

---

### Task 5: Birthdays Calendar View

- [ ] **Step 1: Create view via MCP**

Call `notion-create-view` with:
- Type: calendar
- Name: "Birthdays"
- Calendar by: Birthday

- [ ] **Step 2: Verify**

Confirm "Birthdays" calendar view appears in the database.

---

### Task 6: By Industry Board View

- [ ] **Step 1: Create view via MCP**

Call `notion-create-view` with:
- Type: board
- Name: "By Industry"
- Group by: Industry

- [ ] **Step 2: Verify**

Confirm "By Industry" board view appears.

---

### Task 7: Recent Events View

- [ ] **Step 1: Create view via MCP**

Call `notion-create-view` with:
- Type: table
- Name: "Recent Events"
- Visible properties: Name, Current Title, Company, Latest Notable Event, Latest Event Date, Last Contacted
- Sort: Latest Event Date descending

- [ ] **Step 2: Verify**

Confirm "Recent Events" view appears with correct sort order.

---

### Task 8: Job Search View

- [ ] **Step 1: Create view via MCP**

Call `notion-create-view` with:
- Type: table
- Name: "Job Search"
- Visible properties: Name, Current Title, Company, Industry, Professional Expertise, Open Job Posting URL, Connection Detail

- [ ] **Step 2: Verify**

Confirm "Job Search" view appears with correct columns.

- [ ] **Step 3: Commit all views**

```bash
git add docs/personal-crm/notion-ids.md
git commit -m "feat: create all 6 views for Professional Contacts CRM"
```

---

## Chunk 3: Job Target Profile Page

### Task 9: Create My Job Target Profile Page

**Files:**
- Modify: `docs/personal-crm/notion-ids.md` — add profile page ID

This is a regular Notion page (not in the database). Claude reads it to contextualize job search queries in Plan 5.

- [ ] **Step 1: Create page via MCP**

Call `notion-create-pages` with:
- Parent: [parent-page-id]
- Title: "My Job Target Profile"
- Content (template):

```
## Target Role
[Job title(s) you're targeting — e.g., "Real Estate Analyst, Asset Manager"]

## Industries of Interest
[e.g., Commercial Real Estate, PropTech, Investment Management]

## Key Skills / Keywords
[e.g., financial modeling, deal underwriting, Argus, Excel, asset management]

## Resume Summary
[2–3 sentence summary of your background and what you bring]

## What I'm Looking For
[Type of company, size, culture, location preferences, deal with remote/hybrid]
```

- [ ] **Step 2: Verify page was created**

Call `notion-fetch` on the returned page ID. Confirm page exists with the template content.

- [ ] **Step 3: Record profile page ID**

Update `docs/personal-crm/notion-ids.md` — fill in the Job Target Profile Page ID.

- [ ] **Step 4: Prompt user to fill it in**

Tell the user: "Your Job Target Profile page is live in Notion. Fill in your actual target role, skills, and resume summary — Claude will read this when you run job search queries in Plan 5."

- [ ] **Step 5: Commit**

```bash
git add docs/personal-crm/notion-ids.md
git commit -m "feat: create Job Target Profile page in Notion"
```

---

## Chunk 4: CSV Import

### Task 10: Smoke Test — Add One Contact Manually

Before building the CSV workflow, verify the database accepts data correctly.

- [ ] **Step 1: Pick one LinkedIn connection to test with**

Choose someone you know well — you'll verify their data looks right.

- [ ] **Step 2: Create the contact via MCP**

Call `notion-create-pages` with parent = [database-id] and properties:
- Name: full name
- LinkedIn URL: their profile URL
- Connection Detail: how you know them (e.g., "University of Illinois 2019")
- Current Title: their current role
- Company: their current company
- Industry: select the best match
- Follow-up Frequency: "Annually" (default)

Leave Birthday, Latest Notable Event, Open Job Posting URL, Professional Expertise, Personal Interests empty — those get filled during enrichment (Plan 2).

- [ ] **Step 3: Verify in Notion**

Open Notion and confirm:
- Contact appears in "All Contacts" view with all filled fields
- Contact appears on the correct column in "By Industry" board
- No errors or missing properties

_No commit needed for this task — no file changes are produced._

---

### Task 11: Document LinkedIn CSV Export + Import Process

**Files:**
- Create: `docs/personal-crm/csv-import-guide.md`

- [ ] **Step 1: Write the export guide**

```markdown
# LinkedIn CSV Export Steps

1. Go to LinkedIn → Me → Settings & Privacy
2. Click "Data Privacy" in the left nav
3. Click "Get a copy of your data"
4. Select "Connections" only (faster than full archive)
5. Click "Request archive"
6. LinkedIn emails you a download link (usually within 10 minutes)
7. Download and unzip — the file is named `Connections.csv`

## What LinkedIn Provides in the CSV
| Field | Maps to Notion |
|---|---|
| First Name + Last Name | Name |
| URL | LinkedIn URL |
| Position | Current Title |
| Company | Company |
| Connected On | Connection Detail (e.g., "LinkedIn since 2021-03-15") |
| Email Address | NOT stored (privacy) |
```

- [ ] **Step 2: Write the import workflow instructions**

```markdown
# CSV Import Workflow

## How to run it
Tell Claude: "Import my LinkedIn connections from [file path to Connections.csv]"

## What Claude does
For each row in the CSV, Claude calls notion-create-pages with:
- Name = First Name + " " + Last Name
- LinkedIn URL = URL field
- Current Title = Position field
- Company = Company field
- Connection Detail = "LinkedIn connection since [Connected On date]"
- Follow-up Frequency = "Annually" (default — update key contacts after import)

## What Claude skips
- Rows where URL is empty
- Duplicate contacts (same LinkedIn URL already exists in the database)

## LinkedIn URL normalization
LinkedIn URLs can vary in format (e.g., trailing slashes, `https://www.linkedin.com/in/username/` vs `https://linkedin.com/in/username`). Claude normalizes all URLs to lowercase with no trailing slash before duplicate checking.

## After import
- Run enrichment on your top 20–30 contacts (Plan 2) to add:
  Industry, Professional Expertise, Personal Interests, Birthday
- Update Follow-up Frequency for contacts you want to stay close to
- Fill in your Job Target Profile page (Plan 5 prerequisite)

## Note on volume
LinkedIn allows up to 30,000 connections. For large imports, Claude processes
in batches of 50 to avoid rate limits.
```

- [ ] **Step 3: Commit**

```bash
git add docs/personal-crm/csv-import-guide.md
git commit -m "docs: add LinkedIn CSV export and import guide"
```

---

## Final Verification Checklist

- [ ] "Professional Contacts" database exists in Notion with all 15 properties
- [ ] All 6 views exist: All Contacts, Follow-up Queue, Birthdays, By Industry, Recent Events, Job Search
- [ ] "My Job Target Profile" page exists with template content
- [ ] Smoke test contact appears correctly in All Contacts and By Industry views
- [ ] `docs/personal-crm/notion-ids.md` contains: parent page ID, database ID, profile page ID
- [ ] `docs/personal-crm/csv-import-guide.md` exists
- [ ] User has been prompted to fill in Job Target Profile
- [ ] Manual filter steps for Follow-up Queue view are documented

---

## Reference: docs/personal-crm/notion-ids.md Final State

```markdown
# Personal CRM — Notion IDs

## Parent Page
- **ID:** [filled]
- **Name:** [filled]

## Contacts Database
- **ID:** [filled]
- **Name:** Professional Contacts

## Job Target Profile Page
- **ID:** [filled]
- **Name:** My Job Target Profile

## Views
- **All Contacts:** [filled]
- **Follow-up Queue:** [filled]
- **Birthdays:** [filled]
- **By Industry:** [filled]
- **Recent Events:** [filled]
- **Job Search:** [filled]

## Manual Setup Required
### Follow-up Queue view filter
Open the "Follow-up Queue" view in Notion UI and add this filter group:
- OR group 1: Follow-up Frequency = "Monthly" AND Last Contacted < 30 days ago
- OR group 2: Follow-up Frequency = "Quarterly" AND Last Contacted < 90 days ago
- OR group 3: Follow-up Frequency = "Annually" AND Last Contacted < 365 days ago
```
