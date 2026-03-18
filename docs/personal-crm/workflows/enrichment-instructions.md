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

- Wait for the page to load.
- If redirected to a login page: STOP and tell the user "LinkedIn is not logged in in Chrome. Please log in and retry."
- If the URL returns a 404 or "Profile not found": mark contact as SKIPPED and note "Profile not found."

### 3. Capture Full Profile Text

Use `mcp__Claude_in_Chrome__javascript_tool` with:
```javascript
document.body.innerText
```
This is more reliable than `get_page_text` for LinkedIn. Store as `profile_text`.

### 4. Extract Fields — Mapping Guide

Work through `profile_text` top-to-bottom. Extract each field as described:

#### 4a. Current Title
- **Where:** The headline directly below the person's name (e.g., "Analyst, Real Estate Credit").
- **Fallback:** First job title in the "Experience" section.
- **Field:** `Current Title` (rich_text)

#### 4b. Company
- **Where:** Same headline area, or company name in the first Experience entry.
- **Field:** `Company` (rich_text)

#### 4c. Industry
- **Where:** Infer from job title + company (e.g., "Analyst at Apollo Global Management" → `["Alternative Asset Management", "Private Credit"]`).
- **Field:** `Industry` (multi_select) — array of strings

#### 4d. Professional Expertise
- **Where:** The "Skills" section — list of skill names.
- **How many:** Top 5–10 skills.
- **Field:** `Professional Expertise` (multi_select) — array of strings

#### 4e. Personal Interests
- **Where:** "Interests" section (companies/pages they follow), "Volunteer Experience", "Organizations", "Causes".
- **How many:** Up to 8 short descriptive tags.
- **Field:** `Personal Interests` (multi_select) — array of strings

#### 4f. Connection Detail
- **Where:** "How you're connected" card near top of profile — look for "Both attended", "Both worked at", "Connected through", or mutual group membership.
- **Fallback:** If only "1st degree" with no shared context, leave existing value unchanged.
- **Field:** `Connection Detail` (rich_text)

#### 4g. Latest Notable Event
- **Where:** Most recent Experience entry. If start date is within 6 months of today:
  - Same company, new title → "Promoted to <title> at <company>"
  - New company → "Started new role as <title> at <company>"
- **Field:** `Latest Notable Event` (rich_text)

#### 4h. Latest Event Date
- **Where:** Start date of the most recent Experience entry (ISO 8601: `YYYY-MM-DD`).
- **Field:** `Latest Event Date` (date)

#### 4i. Open Job Posting URL
- **How:** Navigate to company jobs page. Find the company's LinkedIn ID by looking at company links on the profile (`/company/<slug>/` or `/company/<numeric-id>/`), then navigate to `/jobs/`.
- If the page shows active job openings: store the URL.
- If no openings: skip.
- **Field:** `Open Job Posting URL` (url)

### 5. Write Back to Notion

Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page` with `command: "update_properties"` and a `properties` object.

**Property shapes:**
| Field | Shape |
|---|---|
| Current Title | plain string value |
| Company | plain string value |
| Connection Detail | plain string value |
| Latest Notable Event | plain string value |
| Industry | JSON-encoded array string e.g. `"[\"Real Estate\", \"Finance\"]"` — must use only existing schema options |
| Professional Expertise | JSON-encoded array string e.g. `"[\"Excel Modeling\", \"Financial Analysis\"]"` |
| Personal Interests | JSON-encoded array string e.g. `"[\"University of Michigan Alumni\"]"` |
| Latest Event Date | use key `"date:Latest Event Date:start"` = `"YYYY-MM-DD"` and `"date:Latest Event Date:is_datetime"` = 0 |
| Open Job Posting URL | plain string — use key `"Open Job Posting URL"` (no prefix needed) |

**Important:** For multi_select fields, if a new option value is needed that isn't in the schema, first call `notion-update-data-source` to add it with `ALTER COLUMN "Field Name" SET MULTI_SELECT(...)` before setting the page property.

Only include fields that were successfully extracted — omit any that couldn't be parsed.

### 6. Verify the Update

Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch` on the page and confirm key fields appear correctly.

### 7. Report to User

```
Enriched: <Name>
  Current Title: <value>
  Company: <value>
  Industry: <values>
  Professional Expertise: <values>
  Personal Interests: <values>
  Connection Detail: <value>
  Latest Notable Event: <value>
  Latest Event Date: <value>
  Open Job Posting URL: <value or "none found">
```

---

## Field Omission Rules

- Never overwrite a field that already has a value UNLESS the new value is clearly more current.
- Exception: `Latest Notable Event` and `Latest Event Date` should always be refreshed if a newer event is found.
- `Notes`, `Follow-up Frequency`, `Birthday`, and `Last Contacted` are never touched during enrichment.

---

## Error Handling

| Situation | Action |
|---|---|
| LinkedIn not logged in | Stop, tell user to log in |
| Profile URL returns 404 | Skip contact, note "Profile not found" |
| Profile is private / limited view | Extract what's visible; note "Partial data — private profile" |
| Skills section not visible | Skip `Professional Expertise`; do not error |
| Company jobs page slug unknown | Use company numeric ID from profile links (e.g. `/company/48414/jobs/`) |
| Notion update fails | Retry once; if it fails again, note the error and continue |

---

## Known Limitations

- `get_page_text` often fails on LinkedIn due to slow JS rendering. Use `javascript_tool` with `document.body.innerText` instead.
- LinkedIn only shows 2 skills on profile page without clicking "Show all X skills" — accept top visible skills.
- Company jobs page sometimes uses numeric ID instead of slug — get it from the company link in the Experience section.
