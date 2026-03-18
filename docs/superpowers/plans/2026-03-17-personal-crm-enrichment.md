# Personal CRM — On-Demand Chrome Enrichment Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When the user asks Claude to enrich a contact (or batch of contacts), Claude navigates to their LinkedIn profile via Chrome MCP, extracts missing or stale field values, and writes the updated data back to the Notion "Professional Contacts" database.
**Architecture:** Chrome MCP drives a logged-in LinkedIn session to extract profile data; Claude parses the page text and maps each extracted value to the correct Notion field; Notion MCP writes the enriched values back to the existing page record. Batch enrichment layers a loop with rate limiting and a summary report on top of the single-contact flow.
**Tech Stack:** Chrome MCP (`mcp__Claude_in_Chrome__*`), Notion MCP (`mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__*`), Notion "Professional Contacts" database (ID stored in `docs/personal-crm/notion-ids.md`).
**Prerequisite:** Plan 1 (Foundation) complete — "Professional Contacts" database exists in Notion and contains at least one test contact with a valid LinkedIn URL.

---

## Chunk 1: Enrichment Workflow Document

### Task 1: Create the Single-Contact Enrichment Instructions Document
**Files:**
- Create: `docs/personal-crm/workflows/enrichment-instructions.md`

- [ ] **Step 1: Create the directory**
Run:
```bash
mkdir -p /Users/hunterheyman/Claude/docs/personal-crm/workflows
```
Verify the directory exists before writing the file.

- [ ] **Step 2: Write `enrichment-instructions.md`**
Create `/Users/hunterheyman/Claude/docs/personal-crm/workflows/enrichment-instructions.md` with the full content below. This document is what Claude reads (or follows from memory) whenever an enrichment trigger is received.

```markdown
# Personal CRM — Enrichment Instructions

These are the step-by-step instructions Claude follows when enriching a single contact
in the "Professional Contacts" Notion database using LinkedIn via Chrome MCP.

---

## Trigger Phrases

Claude begins enrichment when the user says any of:
- "Enrich [contact name]"
- "Enrich [LinkedIn URL]"
- "Refresh [name]'s profile"
- "Update [name]'s LinkedIn info"
- Batch: "Enrich my top N contacts" / "Enrich all contacts"

---

## Step-by-Step: Single Contact Enrichment

### 1. Resolve the Notion Page

If given a name:
- Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search` with `{ "query": "<name>" }`.
- From the results, identify the page in the "Professional Contacts" database.
- Record: `page_id`, `linkedin_url` (from the "LinkedIn URL" property).

If given a LinkedIn URL directly:
- Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search` with `{ "query": "<name or partial URL>" }` to find the matching page.
- Record `page_id`.

### 2. Navigate to the LinkedIn Profile

```
mcp__Claude_in_Chrome__navigate({ "url": "<linkedin_url>" })
```

- Wait for the page to load (the tool will return when navigation completes).
- If redirected to a login page: STOP and tell the user "LinkedIn is not logged in in Chrome. Please log in and retry."
- If the URL returns a 404 or "Profile not found": mark contact as SKIPPED and note "Profile not found."

### 3. Capture Full Profile Text

```
mcp__Claude_in_Chrome__get_page_text({})
```

Store the returned text as `profile_text`. This is the primary source for extraction.

Optionally use `mcp__Claude_in_Chrome__read_page({})` if structured DOM data is needed for a specific section.

### 4. Extract Fields — Mapping Guide

Work through `profile_text` top-to-bottom. Extract each field as described:

#### 4a. Current Title
- **Where:** The bolded headline directly below the person's name at the top of the profile.
- **What to look for:** A line like "Senior Software Engineer at Acme Corp" or "Founder & CEO | Acme Corp".
- **Fallback:** The most recent entry in the "Experience" section — look for the first job entry after the heading "Experience".
- **Field:** `Current Title` (rich_text)

#### 4b. Company
- **Where:** Same headline, or the company name on the most recent Experience entry.
- **What to look for:** After "at " in the headline, or the company name on the first Experience item.
- **Field:** `Company` (rich_text)

#### 4c. Industry
- **Where:** The "About" section sometimes lists industry. Alternatively, look for a line like "Industry:" in the profile sidebar, or infer from the company's listed industry in the Experience entry (LinkedIn sometimes shows it as a sub-line under the company name).
- **Inference rule:** If not explicit, infer from the job title and company — e.g., "Software Engineer at Stripe" → "FinTech / Software". Produce 1–3 tags as a list.
- **Field:** `Industry` (multi_select) — provide as an array of strings, e.g., `["FinTech", "Software"]`

#### 4d. Professional Expertise
- **Where:** The "Skills" section — appears as a list of skill names (e.g., "Python", "Product Management", "Leadership").
- **What to look for:** Text following the "Skills" heading. LinkedIn shows the top endorsed skills first.
- **How many:** Extract the top 5–10 skills. If fewer than 5 are listed, take all of them.
- **Field:** `Professional Expertise` (multi_select) — array of strings

#### 4e. Personal Interests
- **Where:** The "Interests" section (near the bottom of the profile) and/or "Volunteer Experience" / "Causes" sections.
- **What to look for:**
  - Under "Interests": company/newsletter pages the person follows (e.g., "TED", "Harvard Business Review").
  - Under "Volunteer Experience": organizations and roles.
  - Under "Causes": listed causes they care about.
- **How many:** Extract up to 5–8 tags. Use short descriptive labels (e.g., "Volunteering", "Climate Tech", "TED Talks").
- **Field:** `Personal Interests` (multi_select) — array of strings

#### 4f. Connection Detail
- **Where:** The "How you're connected" section — usually a small card near the top of the profile (below the headline) that says something like:
  - "Both attended University of Illinois"
  - "Both worked at Coldwell Banker"
  - "Connected through Jane Doe"
  - "1st degree connection"
- **What to look for:** Any line containing "Both attended", "Both worked at", "Connected through", or a shared institution/company name in that section.
- **Fallback:** If only "1st degree" is shown with no shared context, leave the field unchanged (do not overwrite existing value).
- **Field:** `Connection Detail` (rich_text)

#### 4g. Latest Notable Event
- **Where:** Look for recent changes in the profile:
  1. New job: An Experience entry with a start date within the past 6 months.
  2. Promotion: An Experience entry at the same company with a new title and a recent start date.
  3. Recent "About" section text mentioning a new role, launch, or milestone.
  4. A recent post highlighted on the profile (if visible in page text).
- **Format:** Write a brief human-readable note, e.g., "Started new role as VP of Engineering at Acme Corp" or "Promoted to Senior Director at Stripe".
- **Field:** `Latest Notable Event` (rich_text)

#### 4h. Latest Event Date
- **Where:** The start date on the Experience entry identified in 4g.
- **Format:** Parse to ISO 8601 date string: `YYYY-MM-DD`. If only month/year visible (e.g., "Jan 2026"), use the first of the month: `2026-01-01`.
- **Field:** `Latest Event Date` (date) — pass as `{ "start": "YYYY-MM-DD" }`

#### 4i. Open Job Posting URL
- **Where:** Navigate to the company's LinkedIn jobs page.
  1. From the Experience entry, identify the company name.
  2. Use `mcp__Claude_in_Chrome__find({ "query": "<company name> LinkedIn jobs" })` or construct the URL manually: `https://www.linkedin.com/company/<company-slug>/jobs/`.
  3. Navigate there and check if any job postings appear.
- **If postings found:** Copy the URL of the jobs page (e.g., `https://www.linkedin.com/company/acme-corp/jobs/`).
- **If no postings:** Leave the field empty / do not update.
- **Field:** `Open Job Posting URL` (url)

### 5. Write Back to Notion

Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page` with the `page_id` and a `properties` object containing only the fields that were successfully extracted (do not send null/empty values for fields you couldn't extract — omit them entirely).

Example call structure:
```json
{
  "page_id": "<page_id>",
  "properties": {
    "Current Title": { "rich_text": [{ "text": { "content": "VP of Engineering" } }] },
    "Company": { "rich_text": [{ "text": { "content": "Acme Corp" } }] },
    "Industry": { "multi_select": [{ "name": "Software" }, { "name": "SaaS" }] },
    "Professional Expertise": { "multi_select": [{ "name": "Python" }, { "name": "System Design" }] },
    "Personal Interests": { "multi_select": [{ "name": "Open Source" }, { "name": "Mentorship" }] },
    "Connection Detail": { "rich_text": [{ "text": { "content": "Both attended University of Illinois" } }] },
    "Latest Notable Event": { "rich_text": [{ "text": { "content": "Started as VP of Engineering at Acme Corp" } }] },
    "Latest Event Date": { "date": { "start": "2026-01-01" } },
    "Open Job Posting URL": { "url": "https://www.linkedin.com/company/acme-corp/jobs/" }
  }
}
```

### 6. Verify the Update

Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` with `{ "url": "https://www.notion.so/<page_id_without_dashes>" }` (or use `notion-search` again by name) to retrieve the updated page. Confirm that the newly written fields appear with the expected values.

### 7. Report to User

Return a brief confirmation:
```
Enriched: <Name>
  - Current Title: <value>
  - Company: <value>
  - Industry: <values>
  - Professional Expertise: <values>
  - Personal Interests: <values>
  - Connection Detail: <value>
  - Latest Notable Event: <value>
  - Latest Event Date: <value>
  - Open Job Posting URL: <value or "none found">
```

---

## Field Omission Rules

- Never overwrite a field that already has a value UNLESS the new value is clearly more current (e.g., a more recent job title).
- Exception: `Latest Notable Event` and `Latest Event Date` should always be refreshed if a newer event is found.
- `Notes` is never touched by enrichment — it is user-managed only.
- `Follow-up Frequency`, `Birthday`, and `Last Contacted` are never set during enrichment — they require user input.

---

## Error Handling

| Situation | Action |
|---|---|
| LinkedIn not logged in | Stop, tell user to log in |
| Profile URL returns 404 | Skip contact, note "Profile not found" |
| Profile is private / limited view | Extract what's visible; note "Partial data — private profile" |
| Skills section not visible | Skip `Professional Expertise`; do not error |
| Company slug for jobs page unknown | Use `mcp__Claude_in_Chrome__find` to locate company page first |
| Notion update fails | Retry once; if it fails again, note the error and continue |
```

- [ ] **Step 3: Verify the file was written**
Read back `/Users/hunterheyman/Claude/docs/personal-crm/workflows/enrichment-instructions.md` and confirm it contains the "Step-by-Step: Single Contact Enrichment" heading and all 9 field extraction sub-sections (4a through 4i).

- [ ] **Step 4: Commit**
```bash
git add docs/personal-crm/workflows/enrichment-instructions.md
git commit -m "feat: add LinkedIn enrichment instructions for Personal CRM"
```

---

## Chunk 2: Single Contact Enrichment

### Task 2: Implement and Test Single-Contact Enrichment
**Files:**
- Modify/Reference: `docs/personal-crm/notion-ids.md` (read the database ID from here)
- Reference: `docs/personal-crm/workflows/enrichment-instructions.md`

This task walks through executing the enrichment workflow end-to-end for one contact.

- [ ] **Step 1: Read the Notion database ID**
Read `/Users/hunterheyman/Claude/docs/personal-crm/notion-ids.md` to get the `Professional Contacts` database ID. Store it as `CONTACTS_DB_ID`.

- [ ] **Step 2: Identify the target contact in Notion**
Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search` with:
```json
{ "query": "<contact name>" }
```
From the response, find the page whose `parent.database_id` matches `CONTACTS_DB_ID`. Record:
- `page_id` — the Notion page ID
- `linkedin_url` — the value of the "LinkedIn URL" property
- Current values of all 15 fields (note which are empty/null)

Expected result: one matching page with at minimum `Name` and `LinkedIn URL` populated.

- [ ] **Step 3: Navigate Chrome to the LinkedIn profile**
Call:
```json
mcp__Claude_in_Chrome__navigate({ "url": "<linkedin_url>" })
```
Confirm the page title returned contains the contact's name (not "Sign In" or "Page not found"). If it contains "Sign In", stop and prompt the user to log in to LinkedIn in Chrome.

- [ ] **Step 4: Capture profile text**
Call:
```json
mcp__Claude_in_Chrome__get_page_text({})
```
Inspect the returned text:
- Confirm the contact's name appears near the top.
- Identify which sections are present: look for the headings "Experience", "Skills", "Interests", "Volunteer Experience", "About".
- Note any sections that are absent (they will be skipped).

- [ ] **Step 5: Extract Current Title and Company**
In the captured text:
- Find the headline line directly below the name. It typically appears as: `<Name>\n<Headline>\n<Location>`.
- Parse `Current Title` as the full headline string.
- Parse `Company` as the substring after " at " in the headline, or from the first company name listed under "Experience".
- If the headline contains " | ", split on " | " and use the first segment as title and look for company in Experience.

Expected result: two non-empty strings.

- [ ] **Step 6: Extract Industry**
- Search the text for a line labeled "Industry:" or an industry tag near the top of the profile.
- If not found explicitly, infer 1–3 industry tags from the job title + company name using your knowledge (e.g., "Account Executive at Salesforce" → `["CRM", "SaaS", "Enterprise Software"]`).
- Format as an array of strings for the Notion `multi_select` field.

- [ ] **Step 7: Extract Professional Expertise (Skills)**
- Find the "Skills" section in the text. It appears after the heading "Skills" and lists skill names, often with endorsement counts (e.g., "Python · 45 endorsements").
- Strip endorsement counts: extract just the skill name from each line.
- Take the first 5–10 skills listed.
- Format as an array of strings.

If "Skills" section is absent: skip this field.

- [ ] **Step 8: Extract Personal Interests**
- Find the "Interests" section (near the bottom). It lists company/page names the person follows.
- Find the "Volunteer Experience" section if present. Extract organization names and roles.
- Find the "Causes" section if present. Extract cause names.
- Combine into a deduplicated list of up to 8 short descriptive tags.
- Format as an array of strings.

If none of these sections appear: skip this field.

- [ ] **Step 9: Extract Connection Detail**
- Search the text for phrases: "Both attended", "Both worked at", "Connected through", "You both know".
- If found, extract the full phrase (e.g., "Both attended University of Illinois at Urbana-Champaign").
- Shorten if needed (remove city/state suffixes to keep it readable): "Both attended University of Illinois".
- If only "1st" or "2nd degree" is present with no shared context: skip this field (leave existing value untouched).

- [ ] **Step 10: Extract Latest Notable Event and Date**
- Find the "Experience" section. Look at the first (most recent) entry.
- Check the start date. If the start date is within the last 6 months (relative to today, 2026-03-17): this is the notable event.
  - If the company matches the second entry's company: classify as "Promoted to <new title> at <company>".
  - Otherwise: classify as "Started new role as <title> at <company>".
- Parse the start date into ISO 8601 format (`YYYY-MM-DD`). If only `MMM YYYY` is given, use the first of the month.
- If the most recent entry is older than 6 months: check if any other recent signal exists (e.g., post about a milestone). If nothing recent is found: skip `Latest Notable Event` and `Latest Event Date`.

- [ ] **Step 11: Extract Open Job Posting URL**
- From the Company extracted in Step 6, construct the LinkedIn company jobs URL:
  - Navigate to the contact's current company page by clicking the company name link in the text, or by constructing: `https://www.linkedin.com/company/<slug>/jobs/`
  - To find the slug: call `mcp__Claude_in_Chrome__find({ "query": "<company name> linkedin company" })` and navigate to the result.
- Call:
  ```json
  mcp__Claude_in_Chrome__navigate({ "url": "https://www.linkedin.com/company/<slug>/jobs/" })
  ```
- Call `mcp__Claude_in_Chrome__get_page_text({})` and check if the text contains job posting titles (look for "Full-time", "Part-time", "Remote", or the word "jobs" followed by a count like "47 jobs").
- If postings found: store the URL `https://www.linkedin.com/company/<slug>/jobs/` as `Open Job Posting URL`.
- If none found: skip this field.

- [ ] **Step 12: Write enriched data back to Notion**
Build the `properties` object including only fields that were successfully extracted (omit null/empty). Call:
```json
mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page({
  "page_id": "<page_id>",
  "properties": {
    "Current Title": { "rich_text": [{ "text": { "content": "<extracted>" } }] },
    "Company": { "rich_text": [{ "text": { "content": "<extracted>" } }] },
    "Industry": { "multi_select": [{ "name": "<tag1>" }, { "name": "<tag2>" }] },
    "Professional Expertise": { "multi_select": [{ "name": "<skill1>" }, ...] },
    "Personal Interests": { "multi_select": [{ "name": "<interest1>" }, ...] },
    "Connection Detail": { "rich_text": [{ "text": { "content": "<extracted>" } }] },
    "Latest Notable Event": { "rich_text": [{ "text": { "content": "<extracted>" } }] },
    "Latest Event Date": { "date": { "start": "YYYY-MM-DD" } },
    "Open Job Posting URL": { "url": "<extracted>" }
  }
})
```

Expected result: HTTP 200 response with the updated page object.

- [ ] **Step 13: Verify the Notion update**
Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` with the Notion page URL to retrieve the page. Confirm:
- `Current Title` is no longer empty.
- `Company` is no longer empty.
- At least one `Industry` tag is present.
- `Professional Expertise` contains at least one tag (if the Skills section was present).

Expected result: all written fields appear in the fetched page properties.

- [ ] **Step 14: Report result**
Output a summary to the user:
```
Enriched: <Name>
  LinkedIn URL: <url>
  Current Title: <value>
  Company: <value>
  Industry: <tags>
  Professional Expertise: <skills>
  Personal Interests: <interests>
  Connection Detail: <value or "unchanged">
  Latest Notable Event: <value or "none found">
  Latest Event Date: <value or "none found">
  Open Job Posting URL: <url or "none found">
```

---

## Chunk 3: Batch Enrichment

### Task 3: Implement Batch Enrichment with Rate Limiting and Summary Report
**Files:**
- Reference: `docs/personal-crm/workflows/enrichment-instructions.md`
- Reference: `docs/personal-crm/notion-ids.md`

- [ ] **Step 1: Resolve the contact list**
When the user says "Enrich my top N contacts" or "Enrich all contacts":

1. Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` on the database using the `CONTACTS_DB_ID` to query the database. Use filter/sort parameters:
   - Sort by `Follow-up Frequency` in priority order: Monthly > Quarterly > Annually > As-Needed.
   - Use `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search` or the Notion query API with:
     ```json
     {
       "database_id": "<CONTACTS_DB_ID>",
       "sorts": [{ "property": "Follow-up Frequency", "direction": "ascending" }],
       "page_size": <N>
     }
     ```
   - Note: Notion sorts select fields alphabetically. "Monthly" < "Quarterly" happens to match priority order, but "As-Needed" sorts before "Monthly" (A < M), placing those contacts incorrectly at the top. Use a two-pass approach: first query with filter `Follow-up Frequency is not As-Needed` (ascending), then append As-Needed contacts at the end if filling remaining slots.

2. From the results, build a list of objects: `[{ "page_id": "...", "name": "...", "linkedin_url": "..." }, ...]`.
3. Filter out any contacts where `LinkedIn URL` is empty — add them to a `SKIPPED` list with reason "No LinkedIn URL".

Expected result: an ordered list of contacts to enrich, with a preliminary skip list.

- [ ] **Step 2: Initialize tracking state**
Before starting the loop, initialize:
```
enriched_count = 0
skipped = []   # [{ name, reason }]
errors = []    # [{ name, error_message }]
total = len(contact_list)
```

- [ ] **Step 3: Enrich each contact with rate limiting**
For each contact in the list (in order):

1. Log to the user: `"[N/total] Enriching <Name>..."`
2. Execute the full single-contact enrichment workflow from Task 2 (Steps 3–14) for this contact.
3. **Rate limiting:** After each profile (success or skip), wait 3–5 seconds before proceeding to the next.
   - Use a randomized delay in the 3–5 second range to appear more human-like.
   - Implementation: after completing the Notion write-back, pause before navigating to the next LinkedIn URL. (In practice, Claude will take a brief pause before issuing the next `mcp__Claude_in_Chrome__navigate` call.)
4. Handle errors without stopping the batch:
   - If LinkedIn returns a login wall: stop the entire batch and tell the user "LinkedIn session expired. Please log in and re-run enrichment."
   - If a specific profile returns 404 or "Profile not found": add to `skipped` with reason "Profile not found". Continue to next contact.
   - If a profile shows only a partial view (private): extract what's available, note "Partial data" in the report, and continue.
   - If the Notion update fails for a contact: add to `errors` with the error message. Continue to next contact.

- [ ] **Step 4: Generate the summary report**
After all contacts have been processed, output:

```
Batch Enrichment Complete
─────────────────────────
Total contacts targeted: <total>
Successfully enriched:   <enriched_count>
Skipped:                 <len(skipped)>
Errors:                  <len(errors)>

Skipped contacts:
  - <Name>: <reason>
  - <Name>: <reason>

Errors:
  - <Name>: <error_message>

Fields updated across all contacts:
  - Current Title:        <N> contacts
  - Company:              <N> contacts
  - Industry:             <N> contacts
  - Professional Expertise: <N> contacts
  - Personal Interests:   <N> contacts
  - Connection Detail:    <N> contacts
  - Latest Notable Event: <N> contacts
  - Open Job Posting URL: <N> contacts
```

Track per-field update counts as you go through the loop (increment a counter each time a field is successfully written).

- [ ] **Step 5: Verify batch state in Notion**
After the batch completes, call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search` with `{ "query": "" }` against the database (or use a fetch on 3–5 specific contacts from the batch) and confirm that `Current Title` and `Company` are now populated for those contacts.

Expected result: at least 80% of targeted contacts (excluding those with inaccessible profiles) have `Current Title` and `Company` filled in.

---

## Chunk 4: Smoke Test

### Task 4: Run Enrichment on the Plan 1 Test Contact and Commit
**Files:**
- Reference: `docs/personal-crm/notion-ids.md`
- Reference: `docs/personal-crm/workflows/enrichment-instructions.md`

This task validates the entire enrichment pipeline end-to-end using the test contact created during Plan 1.

- [ ] **Step 1: Identify the test contact**
Read `docs/personal-crm/notion-ids.md` and locate the `page_id` for the test contact created in Plan 1 (it should be noted there). If not noted, call:
```json
mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search({ "query": "<test contact name>" })
```
to find it. Record `page_id` and `linkedin_url`.

- [ ] **Step 2: Capture pre-enrichment state**
Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` on the test contact's Notion page URL and record which of the 9 enrichable fields are currently empty:
- `Current Title`, `Company`, `Industry`, `Professional Expertise`, `Personal Interests`, `Connection Detail`, `Latest Notable Event`, `Latest Event Date`, `Open Job Posting URL`

Expected result: most or all of these fields are empty (post-CSV-import state).

- [ ] **Step 3: Run single-contact enrichment**
Execute the full enrichment workflow (Task 2, Steps 3–14) on the test contact.

During execution, verify each extraction step produces a non-empty result (or note which sections were absent from the profile):
- Confirm Chrome navigated to the correct LinkedIn URL (check page title).
- Confirm `get_page_text` returned text containing the contact's name.
- Confirm at least `Current Title` and `Company` were extracted.

- [ ] **Step 4: Verify post-enrichment state in Notion**
Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` on the test contact again. For each of the 9 enrichable fields:
- If the LinkedIn profile had data for that field: confirm the field is now populated in Notion.
- If the section was absent on LinkedIn: confirm the field is still empty (correct — not filled with garbage).

Expected result: at minimum `Current Title` and `Company` are populated. Ideally 5+ fields are populated if the test profile is a public, fully-filled LinkedIn account.

- [ ] **Step 5: Document findings**
If any fields could not be extracted (e.g., Skills section not visible, Interests section absent), note the reason in a brief comment at the bottom of `docs/personal-crm/workflows/enrichment-instructions.md` under a new section:
```markdown
## Known Limitations (Updated 2026-03-17)
- [Field name]: [reason it may not be extractable]
```

Only add entries for limitations actually encountered during the smoke test. If everything worked, skip this step.

- [ ] **Step 6: Commit**
```bash
# Only include enrichment-instructions.md if it was modified in Step 5 (Known Limitations added).
# It was already committed in Chunk 1 Step 4 if unchanged.
git add docs/superpowers/plans/2026-03-17-personal-crm-enrichment.md
git diff --cached --name-only  # verify what's staged
git commit -m "feat: complete Personal CRM enrichment smoke test"
# If Step 5 added known limitations, also stage the workflow doc:
# git add docs/personal-crm/workflows/enrichment-instructions.md
# git commit --amend --no-edit  (or create a second commit)
```

Verify `git status` shows a clean working tree for these files after the commit.

---

## Reference: Notion Field Type Cheat Sheet

When calling `notion-update-page`, use these property shapes:

| Field | Notion Type | Shape |
|---|---|---|
| Current Title | rich_text | `{ "rich_text": [{ "text": { "content": "..." } }] }` |
| Company | rich_text | `{ "rich_text": [{ "text": { "content": "..." } }] }` |
| Connection Detail | rich_text | `{ "rich_text": [{ "text": { "content": "..." } }] }` |
| Latest Notable Event | rich_text | `{ "rich_text": [{ "text": { "content": "..." } }] }` |
| Notes | rich_text | `{ "rich_text": [{ "text": { "content": "..." } }] }` |
| Industry | multi_select | `{ "multi_select": [{ "name": "..." }, ...] }` |
| Professional Expertise | multi_select | `{ "multi_select": [{ "name": "..." }, ...] }` |
| Personal Interests | multi_select | `{ "multi_select": [{ "name": "..." }, ...] }` |
| Birthday | date | `{ "date": { "start": "YYYY-MM-DD" } }` |
| Last Contacted | date | `{ "date": { "start": "YYYY-MM-DD" } }` |
| Latest Event Date | date | `{ "date": { "start": "YYYY-MM-DD" } }` |
| LinkedIn URL | url | `{ "url": "https://..." }` |
| Open Job Posting URL | url | `{ "url": "https://..." }` |
| Follow-up Frequency | select | `{ "select": { "name": "Monthly" } }` |
| Name | title | `{ "title": [{ "text": { "content": "..." } }] }` |

---

## Reference: MCP Tool Quick Reference

| Tool | Purpose |
|---|---|
| `mcp__Claude_in_Chrome__navigate` | Navigate Chrome to a URL |
| `mcp__Claude_in_Chrome__get_page_text` | Get all visible text from the current page |
| `mcp__Claude_in_Chrome__read_page` | Get structured DOM snapshot |
| `mcp__Claude_in_Chrome__find` | Search for a URL/page (like a browser search) |
| `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search` | Search Notion by text query |
| `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page` | Update properties on an existing Notion page |
| `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` | Fetch a Notion page by URL |
