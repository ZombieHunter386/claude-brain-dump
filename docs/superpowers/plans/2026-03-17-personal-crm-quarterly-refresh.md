# Personal CRM — Quarterly Scheduled Refresh Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automate quarterly enrichment of the Professional Contacts database by having Claude visit each contact's LinkedIn profile via Chrome MCP, detect changes to dynamic fields, update only changed values in Notion, and append a dated summary to a Refresh Log page.

**Architecture:** A scheduled task fires quarterly and reads its self-contained instructions, which walk Claude through fetching all contacts with LinkedIn URLs from the Notion database, iterating with rate-limited scraping via Chrome MCP, diffing scraped values against current Notion field values, writing only detected changes back to Notion via the update-page API, and appending a structured run summary to a dedicated CRM Refresh Log Notion page. All Notion IDs (database, log page) are stored in `docs/personal-crm/notion-ids.md` and referenced inside the scheduled task's instruction payload so Claude has full context at fire time.

**Tech Stack:** `mcp__scheduled-tasks__create_scheduled_task`, `mcp__scheduled-tasks__list_scheduled_tasks`, Chrome MCP (`mcp__Claude_in_Chrome__navigate`, `mcp__Claude_in_Chrome__get_page_text`, `mcp__Claude_in_Chrome__read_page`), Notion MCP (`mcp__notion__API-query-data-source`, `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page`, `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch`, `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search`, `mcp__notion__API-patch-block-children`)

**Prerequisites:** Plans 1 and 2 complete.

---

## Chunk 1: Refresh Instructions Document

### Task 1: Create Quarterly Refresh Instructions
**Files:**
- Create: `docs/personal-crm/workflows/quarterly-refresh-instructions.md`

- [ ] **Step 1: Create the workflows directory and instructions file**

Create `docs/personal-crm/workflows/quarterly-refresh-instructions.md` with the following content. This document is the canonical reference Claude follows when the scheduled task fires. It must be fully self-contained.

```markdown
# Quarterly CRM Refresh — Claude Instructions

_These instructions are executed by Claude when the quarterly scheduled task fires.
All Notion IDs are in `docs/personal-crm/notion-ids.md`._

## Overview

Refresh dynamic fields for every contact in the Professional Contacts database
who has a LinkedIn URL. Update only fields that have changed. Log results to the
CRM Refresh Log page.

## Fields to Refresh (per contact)

| Field | Notion Property | Update if... |
|---|---|---|
| Current Title | `Current Title` (rich_text) | scraped title differs from stored value |
| Company | `Company` (rich_text) | scraped company differs from stored value |
| Industry | `Industry` (multi_select) | scraped industry differs; merge, don't wipe |
| Latest Notable Event | `Latest Notable Event` (rich_text) | new job, promotion, or role change detected |
| Latest Event Date | `Latest Event Date` (date) | updated alongside Latest Notable Event |
| Open Job Posting URL | `Open Job Posting URL` (url) | found a new posting, or clear if prior URL is gone |

## Fields NOT Touched

Birthday, Last Contacted, Follow-up Frequency, Notes, Connection Detail,
Professional Expertise, Personal Interests — never overwrite these.

## Step-by-Step Execution

### Phase 1: Fetch All Contacts

1. Read `docs/personal-crm/notion-ids.md` to get the contacts database ID.
2. Call `mcp__notion__API-query-data-source` with the database ID and a filter:
   - `LinkedIn URL` is not empty
3. Collect the full result set (handle pagination if more than one page of results).
4. For each contact record, note: page ID, Name, LinkedIn URL, and current values
   of all six refreshable fields.

### Phase 2: Scrape Each Profile

For each contact in the list:

1. Call `mcp__Claude_in_Chrome__navigate` with the contact's LinkedIn URL.
2. Wait 3–5 seconds (use a brief pause between requests to avoid rate limiting).
3. Call `mcp__Claude_in_Chrome__get_page_text` to extract visible page text.
4. If the page returns a login wall, captcha, or "profile not found" message:
   - Mark the contact as SKIPPED in the run log.
   - Do not modify any Notion fields.
   - Continue to the next contact.

#### Parsing Rules

- **Current Title:** Look for the headline text directly below the contact's name
  (e.g., "Senior Software Engineer at Acme Corp"). Extract only the job title portion.
- **Company:** Extract the company name from the headline or the top experience entry.
- **Industry:** Look for the "Industry" field on the profile, or infer from company.
  If the existing `Industry` multi_select values are still accurate, keep them.
  Only add newly detected values; do not remove existing ones.
- **Latest Notable Event:** Look for the most recent experience entry.
  If it differs from the stored `Latest Notable Event` (new company, new title,
  promotion indicator), construct a short summary string, e.g.:
  "Promoted to VP of Engineering at Acme Corp".
- **Latest Event Date:** Use the start date of the most recent experience entry,
  formatted as YYYY-MM-DD.
- **Open Job Posting URL:** Navigate to `https://www.linkedin.com/company/[company-slug]/jobs/`
  (derive slug from the company LinkedIn URL if visible on profile).
  If an open posting relevant to the contact's domain is found, record the job URL.
  If no relevant posting exists, clear the field (set to null).

### Phase 3: Diff and Update

For each contact processed successfully:

1. Compare each scraped value to the stored Notion value.
2. Build an update payload containing ONLY the fields where the value differs
   and the new scraped value is non-empty.
3. Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page` with
   the contact's Notion page ID and the update payload.
4. Record what changed: contact name + list of changed fields + old→new values.

**Safety rules:**
- Never update a field if the scraped value is blank or could not be parsed.
- For `Industry` (multi_select): merge new values into existing; do not replace.
- For `Latest Notable Event`: only update if the event is newer than the stored
  `Latest Event Date` (or if no date exists).

### Phase 4: Write Refresh Log Entry

After all contacts are processed:

1. Read `docs/personal-crm/notion-ids.md` to get the CRM Refresh Log page ID.
2. Call `mcp__notion__API-patch-block-children` on the Refresh Log page to
   append a new section with this structure:

```
## Quarterly Refresh — [YYYY-MM-DD]

- **Contacts with LinkedIn URL:** [N]
- **Contacts processed:** [N]
- **Contacts skipped (inaccessible):** [N]
- **Contacts updated:** [N]

### Changes

| Contact | Field | Old Value | New Value |
|---|---|---|---|
| [name] | [field] | [old] | [new] |
| ... | | | |

### Skipped Contacts
- [name] — reason (login wall / profile not found / no LinkedIn URL)
```

3. If zero changes were made, still write the log entry noting "No changes detected."

## Error Handling

- If `mcp__notion__API-query-data-source` fails: abort, do not proceed. Log the error.
- If Chrome MCP is unavailable: abort with a note in the Refresh Log.
- If a single contact's Notion update fails: log the error, continue to next contact.
- Never abort the entire run due to a single contact failure.
```

- [ ] **Step 2: Verify file created correctly**

Read back `docs/personal-crm/workflows/quarterly-refresh-instructions.md` and confirm:
- All six refreshable fields are documented with their Notion property names and types.
- Parsing rules exist for each field.
- Safety rules (no blank overwrites, Industry merge-not-replace, date comparison for events) are present.
- Phase 4 log format matches the structure defined in Task 2 below.

- [ ] **Step 3: Commit**

Commit `docs/personal-crm/workflows/quarterly-refresh-instructions.md` with message:
`docs: add quarterly CRM refresh instructions for scheduled task`

---

## Chunk 2: Refresh Log Page

### Task 2: Create CRM Refresh Log Notion Page
**Files:**
- Modify: `docs/personal-crm/notion-ids.md` — add Refresh Log page ID

- [ ] **Step 1: Read notion-ids.md to get parent page ID**

Read `docs/personal-crm/notion-ids.md`. Locate the Parent Page ID — this is where the Refresh Log page will be created as a child.

- [ ] **Step 2: Create the Refresh Log page**

Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-create-pages` with:
- Parent: the CRM parent page ID from notion-ids.md
- Title: `CRM Refresh Log`
- Initial content (first block): A short intro paragraph:

```
Quarterly refresh run history for the Professional Contacts database.
Each entry records the run date, contacts processed, fields updated,
and a table of individual changes. Entries are appended automatically
by the quarterly scheduled task.
```

- [ ] **Step 3: Record the Refresh Log page ID**

After the page is created, capture the returned page ID. Open `docs/personal-crm/notion-ids.md` and append:

```markdown
## CRM Refresh Log Page
- **ID:** [returned-page-id]
- **Name:** CRM Refresh Log
- **Purpose:** Append-only log of quarterly refresh runs; each run adds a dated section
```

- [ ] **Step 4: Verify page is accessible**

Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` with the new page ID to confirm it loads correctly and the intro text is present.

- [ ] **Step 5: Commit**

Commit the updated `docs/personal-crm/notion-ids.md` with message:
`docs: add CRM Refresh Log Notion page ID to notion-ids.md`

---

## Chunk 3: Create the Scheduled Task

### Task 3: Register the Quarterly Scheduled Task
**Files:**
- No local files created. Scheduled task registered via MCP.

- [ ] **Step 1: Read all required IDs**

Read `docs/personal-crm/notion-ids.md` and collect:
- Contacts database ID (from "Contacts Database" section)
- CRM Refresh Log page ID (from "CRM Refresh Log Page" section)
- Parent page ID (from "Parent Page" section)

These IDs will be embedded directly in the scheduled task's instruction payload so Claude has full context when the task fires without needing to search for them.

- [ ] **Step 2: Create the scheduled task**

Call `mcp__scheduled-tasks__create_scheduled_task` with the following parameters:

**name:** `Personal CRM — Quarterly Contact Refresh`

**schedule:** `0 9 1 */3 *`
_(Runs at 9:00 AM on the 1st of every 3rd month — January, April, July, October)_

**instructions:** The full self-contained instruction payload (embed the actual IDs collected in Step 1 where placeholders appear below):

```
You are running the quarterly Personal CRM refresh. Follow these steps exactly.

## Context

- Contacts Database ID: [CONTACTS_DATABASE_ID]
- CRM Refresh Log Page ID: [REFRESH_LOG_PAGE_ID]
- Instructions reference: docs/personal-crm/workflows/quarterly-refresh-instructions.md

## What to Do

1. Read docs/personal-crm/workflows/quarterly-refresh-instructions.md for the
   complete step-by-step process.

2. Execute Phase 1 (Fetch All Contacts): Query the Contacts database using
   mcp__notion__API-query-data-source with the database ID above. Filter for
   records where "LinkedIn URL" is not empty.

3. Execute Phase 2 (Scrape Each Profile): For each contact, navigate to their
   LinkedIn URL using mcp__Claude_in_Chrome__navigate, then extract text using
   mcp__Claude_in_Chrome__get_page_text. Wait 3–5 seconds between each contact.

4. Execute Phase 3 (Diff and Update): Compare scraped values to stored Notion
   values. Call mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page
   only for contacts where values have changed, and only for the fields that
   changed. Never overwrite fields with blank/unparseable values.
   For Industry: merge new values, do not replace existing ones.
   For Latest Notable Event: only update if the event is newer than the stored
   Latest Event Date.

5. Execute Phase 4 (Write Refresh Log): Append a dated summary section to the
   CRM Refresh Log page (ID above) using mcp__notion__API-patch-block-children.
   Include: run date, # contacts processed, # skipped, # updated, and a table
   of field-level changes (contact name | field | old value | new value).

## Fields to Refresh (never touch others)

- Current Title (rich_text)
- Company (rich_text)
- Industry (multi_select) — merge only
- Latest Notable Event (rich_text)
- Latest Event Date (date)
- Open Job Posting URL (url)

## If Something Goes Wrong

- If Chrome MCP is unavailable: abort, append error note to Refresh Log.
- If a single contact's update fails: log the error, continue to the next contact.
- If the database query fails entirely: abort, do not proceed.
```

- [ ] **Step 3: Verify the task was registered**

Call `mcp__scheduled-tasks__list_scheduled_tasks` and confirm:
- A task named `Personal CRM — Quarterly Contact Refresh` appears in the list.
- The schedule expression is `0 9 1 */3 *`.
- The instructions are present and contain both embedded Notion IDs.

- [ ] **Step 4: Commit verification note**

Append a note to `docs/personal-crm/notion-ids.md` under a new section:

```markdown
## Scheduled Tasks
- **Quarterly Refresh Task Name:** Personal CRM — Quarterly Contact Refresh
- **Schedule:** `0 9 1 */3 *` (9 AM on the 1st of Jan / Apr / Jul / Oct)
- **Next Expected Run:** 2026-04-01 09:00
```

Commit with message: `docs: record quarterly refresh scheduled task metadata`

---

## Chunk 4: Dry Run Test

### Task 4: Manually Trigger a 3-Contact Dry Run
**Files:**
- No new files. Validates Task 1–3 outputs.

- [ ] **Step 1: Select 3 test contacts**

Query the Professional Contacts database using `mcp__notion__API-query-data-source`. Select 3 contacts that have a LinkedIn URL populated. Note their:
- Notion page ID
- Name
- LinkedIn URL
- Current values of all 6 refreshable fields

These 3 contacts will be the dry run subjects — do not process the full database yet.

- [ ] **Step 2: Run Phase 2 (scrape) for the 3 contacts**

For each of the 3 contacts:
1. Call `mcp__Claude_in_Chrome__navigate` with their LinkedIn URL.
2. Pause 3–5 seconds.
3. Call `mcp__Claude_in_Chrome__get_page_text` and extract:
   - Current Title
   - Company
   - Industry
   - Most recent experience entry (for Latest Notable Event + Latest Event Date)
4. Navigate to the company's LinkedIn jobs page and check for an open posting URL.

If any profile is inaccessible (login wall, not found), note it as SKIPPED and continue.

- [ ] **Step 3: Run Phase 3 (diff and update) for the 3 contacts**

For each successfully scraped contact:
1. Compare scraped values to the values noted in Step 1.
2. Build a minimal update payload (only changed, non-blank fields).
3. Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page` with the payload.
4. After each update, call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch`
   on that page to confirm the values were written correctly in Notion.

- [ ] **Step 4: Write dry run log entry**

Call `mcp__notion__API-patch-block-children` on the CRM Refresh Log page to append a dry run entry:

```
## Dry Run — [today's date]

- **Contacts processed:** 3
- **Contacts skipped:** [N]
- **Contacts updated:** [N]
- **Note:** Manual dry run test — not a full quarterly refresh.

### Changes
[table of any changes, or "No changes detected"]
```

- [ ] **Step 5: Verify Refresh Log in Notion**

Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` on the CRM Refresh Log page. Confirm the dry run entry is visible and formatted correctly.

- [ ] **Step 6: Confirm safety rules held**

Review the dry run output and verify:
- No fields were blanked out (scrape failures did not overwrite good data).
- Industry values were merged, not replaced.
- Latest Notable Event was only updated if the detected event is newer than the stored date.
- Fields not in the refreshable list (Birthday, Last Contacted, etc.) were untouched.

If any safety rule was violated, open `docs/personal-crm/workflows/quarterly-refresh-instructions.md`, tighten the relevant parsing or diff rule, and re-run the affected contact manually to correct the value.

- [ ] **Step 7: Commit**

Commit any updates to `docs/personal-crm/workflows/quarterly-refresh-instructions.md` (if corrections were needed) with message:
`fix: tighten quarterly refresh safety rules based on dry run findings`

If no corrections were needed, note that in the commit:
`docs: dry run complete — no instruction changes required`
