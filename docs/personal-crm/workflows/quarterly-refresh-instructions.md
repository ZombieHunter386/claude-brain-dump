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
3. Use `mcp__Claude_in_Chrome__javascript_tool` with `document.body.innerText` to extract visible page text.
4. If the page returns a login wall, captcha, or "profile not found" message:
   - Mark the contact as SKIPPED in the run log.
   - Do not modify any Notion fields.
   - Continue to the next contact.

#### Parsing Rules

- **Current Title:** Look for the headline text directly below the contact's name.
  Extract only the job title portion.
- **Company:** Extract the company name from the headline or the top experience entry.
- **Industry:** Infer from job title and company. If the existing `Industry` multi_select
  values are still accurate, keep them. Only add newly detected values; do not remove existing ones.
  Use only values from the existing schema options.
- **Latest Notable Event:** Look for the most recent experience entry.
  If it differs from the stored `Latest Notable Event` (new company, new title,
  promotion indicator), construct a short summary string, e.g.:
  "Promoted to VP of Engineering at Acme Corp".
- **Latest Event Date:** Use the start date of the most recent experience entry,
  formatted as YYYY-MM-DD.
- **Open Job Posting URL:** Navigate to `https://www.linkedin.com/company/<id>/jobs/`
  (use the numeric company ID from profile links — more reliable than slug).
  If an open posting relevant to the contact's domain is found, record the job URL.
  If no relevant posting exists, clear the field (set to null).

### Phase 3: Diff and Update

For each contact processed successfully:

1. Compare each scraped value to the stored Notion value.
2. Build an update payload containing ONLY the fields where the value differs
   and the new scraped value is non-empty.
3. Call `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-update-page` with
   the contact's Notion page ID and the update payload using `command: "update_properties"`.
4. Record what changed: contact name + list of changed fields + old→new values.

**Safety rules:**
- Never update a field if the scraped value is blank or could not be parsed.
- For `Industry` (multi_select): merge new values into existing; do not replace.
  Use JSON-encoded array format: `"[\"Real Estate\", \"Finance\"]"`.
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

### Skipped Contacts
- [name] — reason (login wall / profile not found / no LinkedIn URL)
```

3. If zero changes were made, still write the log entry noting "No changes detected."

## Error Handling

- If `mcp__notion__API-query-data-source` fails: abort, do not proceed. Log the error.
- If Chrome MCP is unavailable: abort with a note in the Refresh Log.
- If a single contact's Notion update fails: log the error, continue to next contact.
- Never abort the entire run due to a single contact failure.
