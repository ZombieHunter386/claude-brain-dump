# Personal CRM — Follow-up Reminders + Google Calendar Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a weekly scheduled task that queries the Professional Contacts database for overdue contacts, determines a personalized outreach reason for each one, and creates a 30-minute Google Calendar event per contact via the Chrome MCP.

**Architecture:** A local instructions document defines all reminder logic (overdue thresholds, reason-for-outreach priority, event title format, and Chrome steps for Google Calendar). The scheduled task is fully self-contained — its prompt embeds the Notion database ID, all logic thresholds, and the step-by-step Chrome MCP workflow so it can execute without any external context. Google Calendar events are created by navigating to calendar.google.com via the Chrome MCP, since no Google Calendar MCP exists.

**Tech Stack:** `mcp__scheduled-tasks__create_scheduled_task`, Notion MCP (`mcp__notion__API-query-data-source`, `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch`), Chrome MCP (`mcp__Claude_in_Chrome__navigate`, `mcp__Claude_in_Chrome__find`, `mcp__Claude_in_Chrome__form_input`, `mcp__Claude_in_Chrome__javascript_tool`, `mcp__Claude_in_Chrome__get_page_text`)

**Prerequisite:** Plan 1 (Foundation) complete. `docs/personal-crm/notion-ids.md` exists with the Professional Contacts database ID.

---

## Chunk 1: Reminder Logic Document

### Task 1: Create Follow-up Reminder Instructions Document

**Files:**
- Create: `docs/personal-crm/workflows/followup-reminder-instructions.md`

This document is the canonical reference for all logic the scheduled task follows. It also serves as a manual runbook if you want to trigger reminders outside the scheduled task.

- [ ] **Step 1: Read the Notion database ID**

Open `docs/personal-crm/notion-ids.md` and copy the value for "Contacts Database ID". You will need it in every query below.

- [ ] **Step 2: Write the overdue query logic section**

Create `docs/personal-crm/workflows/followup-reminder-instructions.md` with the following content in a section called "## Querying for Overdue Contacts":

```markdown
## Querying for Overdue Contacts

Use `mcp__notion__API-query-data-source` with the Professional Contacts database ID.

**Database ID:** (fill from docs/personal-crm/notion-ids.md)

Fetch all contacts where Follow-up Frequency is NOT "As-Needed". Filter out any
contact where Follow-up Frequency is empty or null (treat as As-Needed — skip them).

Request these properties for each contact:
- Name (title)
- Last Contacted (date)
- Follow-up Frequency (select)
- Latest Notable Event (rich_text)
- Latest Event Date (date)
- Birthday (date)
- Current Title (rich_text)
- Company (rich_text)
- LinkedIn URL (url)

After fetching, apply the overdue threshold logic in memory (not via Notion filter)
so the logic is transparent and auditable.
```

- [ ] **Step 3: Write the overdue threshold logic section**

Add a section called "## Overdue Threshold Logic":

```markdown
## Overdue Threshold Logic

Today's date is the date the task fires. For each contact:

1. If Follow-up Frequency = "As-Needed" → SKIP. Do not create a calendar event.

2. If Last Contacted is empty (null/blank) → treat as OVERDUE regardless of frequency.
   Reason: we have no record of ever reaching out, so they're always surfaced.

3. If Last Contacted is set, compute days since Last Contacted:
   days_since = (today - Last Contacted) in whole days

   - Monthly   → overdue if days_since > 30
   - Quarterly → overdue if days_since > 90
   - Annually  → overdue if days_since > 365

4. If NOT overdue → SKIP. Do not create a calendar event.

5. If OVERDUE → proceed to Reason-for-Outreach Logic below.
```

- [ ] **Step 4: Write the reason-for-outreach logic section**

Add a section called "## Reason-for-Outreach Logic":

```markdown
## Reason-for-Outreach Logic

For each overdue contact, determine the event title suffix using this priority order.
Evaluate conditions top-to-bottom and use the FIRST match.

**Priority 1 — Birthday coming up:**
- Condition: Birthday is set AND the birthday (month + day) falls within the next
  14 calendar days from today (year-agnostic — compare month/day only).
- Reason text: "Birthday message"
- Event title: "Reach out: [Name] — Birthday message"

**Priority 2 — Recent notable event not yet acknowledged:**
- Condition: Latest Event Date is set AND Latest Event Date is within the last 30 days
  AND (Last Contacted is empty OR Last Contacted < Latest Event Date)
  (meaning we haven't contacted them since the event happened)
- Reason text: "Congratulate: [Latest Notable Event]"
  - If Latest Notable Event is empty, fall back to: "Congratulate on recent news"
- Event title: "Reach out: [Name] — Congratulate: [Latest Notable Event]"

**Priority 3 — Default check-in:**
- Condition: none of the above matched
- Reason text: "Check-in"
- Event title: "Reach out: [Name] — Check-in"
```

- [ ] **Step 5: Write the calendar event format section**

Add a section called "## Google Calendar Event Format":

```markdown
## Google Calendar Event Format

For each overdue contact, create ONE Google Calendar event with:

- **Title:** "Reach out: [Name] — [Reason]"
  (where Reason comes from the Reason-for-Outreach Logic above)

- **Date:** Next available weekday from today.
  - If today is Monday–Thursday → use tomorrow.
  - If today is Friday → use Monday (skip Saturday and Sunday).
  - If today is Saturday → use Monday.
  - If today is Sunday → use Monday.
  - Note: do NOT use today itself. Always schedule for tomorrow or later.

- **Time:** 9:00 AM – 9:30 AM (30-minute block)

- **Description:** Use this template:
  ```
  Contact: [Name]
  Title: [Current Title] at [Company]
  LinkedIn: [LinkedIn URL]

  Reason for outreach: [Reason text]
  [If Birthday] → Their birthday is on [Birthday date]. Send a warm message.
  [If Notable Event] → They [Latest Notable Event] on [Latest Event Date]. Congratulate them.
  [If Check-in] → It has been [days_since] days since last contact. Reach out to stay top of mind.

  Last Contacted: [Last Contacted date or "Never"]
  Follow-up Frequency: [Follow-up Frequency]
  ```

- **Calendar:** Your primary Google Calendar (default).
```

- [ ] **Step 6: Commit**

```bash
git add docs/personal-crm/workflows/followup-reminder-instructions.md
git commit -m "docs: add follow-up reminder instructions with overdue logic and calendar event format"
```

---

## Chunk 2: Google Calendar Event Creation via Chrome MCP

### Task 2: Document and Verify Chrome MCP Steps for Google Calendar

**Files:**
- Modify: `docs/personal-crm/workflows/followup-reminder-instructions.md` — add Chrome MCP steps section

The Chrome MCP is the only way to create Google Calendar events (no Calendar API MCP is available). This section documents the exact steps and verifies them by creating one test event manually.

- [ ] **Step 1: Write the Chrome MCP steps section**

Append a section called "## Chrome MCP Steps for Creating a Google Calendar Event" to `docs/personal-crm/workflows/followup-reminder-instructions.md`:

```markdown
## Chrome MCP Steps for Creating a Google Calendar Event

Repeat these steps for EACH overdue contact. Do not batch — create one event at a time
and wait for confirmation before proceeding to the next.

### Step A: Navigate to Google Calendar

Call `mcp__Claude_in_Chrome__navigate` with URL:
  https://calendar.google.com

Wait for the page to load. Call `mcp__Claude_in_Chrome__get_page_text` and confirm
the calendar grid is visible (text should include day/week/month navigation labels).
If redirected to a Google login page, stop and notify the user: "Google Calendar
requires sign-in. Please open calendar.google.com in Chrome and log in, then re-run
the task."

### Step B: Open the New Event dialog

Call `mcp__Claude_in_Chrome__find` with query: "Create" or "+ New" button
(Google Calendar shows a "+ Create" button in the top-left).

Call `mcp__Claude_in_Chrome__javascript_tool` to click the create button:
```javascript
// Find and click the "+ Create" button
const btn = document.querySelector('[data-view="CREATE"]') ||
            [...document.querySelectorAll('button')].find(b => b.textContent.trim() === '+ Create') ||
            document.querySelector('button[aria-label="Create"]');
if (btn) btn.click();
```

Wait ~1 second for the quick-event dialog to appear. Then click "More options"
to open the full event editor (needed to set description and exact time).

Call `mcp__Claude_in_Chrome__javascript_tool`:
```javascript
// Click "More options" link in the quick event dialog
const moreOpts = [...document.querySelectorAll('a, button')].find(
  el => el.textContent.trim().toLowerCase().includes('more options')
);
if (moreOpts) moreOpts.click();
```

### Step C: Fill in the event title

Wait for the full event editor to load. Call `mcp__Claude_in_Chrome__find` with
query: "Add title" input field.

Call `mcp__Claude_in_Chrome__form_input` with:
- selector: input[placeholder="Add title"] or the title field
- value: "Reach out: [Name] — [Reason]"

### Step D: Set the event date and time

Locate the date field and time fields in the event editor.

Call `mcp__Claude_in_Chrome__javascript_tool` to set the start date:
```javascript
// Set the date input — find the start date field
const dateInputs = document.querySelectorAll('input[type="date"], [data-date]');
// Or use the visible date button — click it to open date picker
const startDateBtn = document.querySelector('[data-testid="start-date"] button') ||
                     document.querySelectorAll('[role="button"]')[1]; // date area
if (startDateBtn) startDateBtn.click();
```

After clicking the date, use `mcp__Claude_in_Chrome__form_input` to type the
target date in MM/DD/YYYY format into the date input that appears.

For the time, find the start time field and set it to 9:00 AM. Find the end time
field and set it to 9:30 AM. Use `mcp__Claude_in_Chrome__form_input` for each.

If the time pickers are dropdowns, use `mcp__Claude_in_Chrome__javascript_tool`
to select the correct options:
```javascript
// Example: set start time to 9:00 AM
const timeInputs = [...document.querySelectorAll('input[aria-label*="time" i], input[placeholder*="time" i]')];
if (timeInputs[0]) { timeInputs[0].value = '9:00 AM'; timeInputs[0].dispatchEvent(new Event('input', {bubbles:true})); }
if (timeInputs[1]) { timeInputs[1].value = '9:30 AM'; timeInputs[1].dispatchEvent(new Event('input', {bubbles:true})); }
```

### Step E: Add the event description

Find the description field ("Add description or attachments").

Call `mcp__Claude_in_Chrome__form_input` with:
- selector: [contenteditable][aria-label*="description" i] or textarea[aria-label*="description" i]
- value: the full description block from the "Google Calendar Event Format" section above

If `form_input` doesn't work on a contenteditable div, use `javascript_tool`:
```javascript
const desc = document.querySelector('[contenteditable][aria-label*="description" i]') ||
             document.querySelector('[data-testid="description-input"]');
if (desc) {
  desc.focus();
  document.execCommand('insertText', false, `YOUR_DESCRIPTION_TEXT_HERE`);
}
```

### Step F: Save the event

Find the "Save" button and click it.

Call `mcp__Claude_in_Chrome__javascript_tool`:
```javascript
const saveBtn = [...document.querySelectorAll('button')].find(
  b => b.textContent.trim().toLowerCase() === 'save'
);
if (saveBtn) saveBtn.click();
```

### Step G: Verify the event was created

After saving, call `mcp__Claude_in_Chrome__get_page_text` to confirm you are back
on the calendar grid. Call `mcp__Claude_in_Chrome__find` to look for the event
title on the calendar. If the event is visible on the correct date, log:
"Created event: [event title] on [date]".

If the save failed (still on the event editor), try clicking Save again or
call `mcp__Claude_in_Chrome__javascript_tool` to submit the form directly.
```

- [ ] **Step 2: Create one test event manually via Chrome MCP**

Follow the steps documented above to create a single test event:
- Title: "Reach out: Test Contact — Check-in"
- Date: next available weekday from 2026-03-17 (i.e., 2026-03-18)
- Time: 9:00 AM – 9:30 AM
- Description:
  ```
  Contact: Test Contact
  Title: Software Engineer at Test Corp
  LinkedIn: https://linkedin.com/in/test

  Reason for outreach: Check-in
  It has been 400 days since last contact. Reach out to stay top of mind.

  Last Contacted: 2024-11-07
  Follow-up Frequency: Annually
  ```

- [ ] **Step 3: Verify the test event in Google Calendar**

Call `mcp__Claude_in_Chrome__navigate` with URL:
  `https://calendar.google.com`

Navigate to 2026-03-18. Confirm the event "Reach out: Test Contact — Check-in"
is visible at 9:00 AM. Take a screenshot via `mcp__Claude_in_Chrome__javascript_tool`
or read the page text to confirm.

- [ ] **Step 4: Delete the test event**

Click on the test event in Google Calendar, then click the trash/delete icon to
remove it. This was only a verification event — the real events will be created
by the scheduled task.

- [ ] **Step 5: Commit the updated instructions file**

```bash
git add docs/personal-crm/workflows/followup-reminder-instructions.md
git commit -m "docs: add Chrome MCP steps for Google Calendar event creation"
```

---

## Chunk 3: Create the Scheduled Task

### Task 3: Register Weekly Follow-up Reminder Scheduled Task

**Files:**
- Read: `docs/personal-crm/notion-ids.md` — to embed the database ID in the task prompt

The scheduled task prompt must be completely self-contained. When it fires, Claude has no conversation history and no access to local files — every ID, threshold, and step must be embedded in the prompt itself.

- [ ] **Step 1: Read the Notion database ID**

Open `docs/personal-crm/notion-ids.md`. Copy the exact value for "Contacts Database ID". You will paste it literally into the task prompt below.

- [ ] **Step 2: Create the scheduled task**

Call `mcp__scheduled-tasks__create_scheduled_task` with:
- **Schedule:** weekly, every Monday at 8:00 AM
- **Task name:** "Personal CRM — Weekly Follow-up Reminders"
- **Prompt:** (use the full self-contained prompt below, substituting [DATABASE_ID] with the real ID from Step 1)

```
You are running the weekly Personal CRM follow-up reminder workflow.
Determine today's date by running: `date +%Y-%m-%d` in a bash command or use the currentDate from system context. Use it for all threshold calculations below.

## Step 1: Query Overdue Contacts

Query the Professional Contacts Notion database using mcp__notion__API-query-data-source.
Database ID: [DATABASE_ID]

Fetch all pages. For each contact, read these properties:
- Name (title)
- Last Contacted (date)
- Follow-up Frequency (select: Monthly, Quarterly, Annually, As-Needed)
- Latest Notable Event (rich_text)
- Latest Event Date (date)
- Birthday (date)
- Current Title (rich_text)
- Company (rich_text)
- LinkedIn URL (url)

## Step 2: Apply Overdue Logic

For each contact:

1. If Follow-up Frequency = "As-Needed" or is empty → SKIP.

2. If Last Contacted is empty → treat as OVERDUE (never contacted).

3. If Last Contacted is set:
   - Compute days_since = (today - Last Contacted) in whole days
   - Monthly   → overdue if days_since > 30
   - Quarterly → overdue if days_since > 90
   - Annually  → overdue if days_since > 365
   - If NOT overdue → SKIP.

## Step 3: Determine Reason for Outreach

For each overdue contact, evaluate these conditions in priority order (first match wins):

Priority 1 — Birthday:
- Birthday is set AND birthday month/day falls within the next 14 calendar days from today
- Reason: "Birthday message"
- Event title: "Reach out: [Name] — Birthday message"

Priority 2 — Recent notable event not yet acknowledged:
- Latest Event Date is set AND Latest Event Date is within the last 30 days
  AND (Last Contacted is empty OR Last Contacted < Latest Event Date)
- Reason: "Congratulate: [Latest Notable Event]" (if event text is empty: "Congratulate on recent news")
- Event title: "Reach out: [Name] — Congratulate: [Latest Notable Event]"

Priority 3 — Default:
- Reason: "Check-in"
- Event title: "Reach out: [Name] — Check-in"

## Step 4: Determine Event Date

Next available weekday from today (do NOT use today itself):
- Monday–Thursday → tomorrow
- Friday/Saturday/Sunday → next Monday

All events use time: 9:00 AM – 9:30 AM.

## Step 5: Create Google Calendar Events

For EACH overdue contact, create one Google Calendar event using the Chrome MCP.
Process one contact at a time. Complete Steps A–G before moving to the next contact.

### Step A: Navigate
Call mcp__Claude_in_Chrome__navigate with url: https://calendar.google.com
Call mcp__Claude_in_Chrome__get_page_text to confirm calendar loaded.
If login page appears, STOP and output: "ACTION REQUIRED: Please log in to Google Calendar at calendar.google.com in Chrome, then re-trigger this task."

### Step B: Open new event dialog
Call mcp__Claude_in_Chrome__javascript_tool:
```javascript
const btn = document.querySelector('[data-view="CREATE"]') ||
            [...document.querySelectorAll('button')].find(b => b.textContent.trim() === '+ Create') ||
            document.querySelector('button[aria-label="Create"]');
if (btn) btn.click();
```
Wait ~1 second. Then click "More options":
```javascript
const moreOpts = [...document.querySelectorAll('a, button')].find(
  el => el.textContent.trim().toLowerCase().includes('more options')
);
if (moreOpts) moreOpts.click();
```

### Step C: Fill title
Call mcp__Claude_in_Chrome__form_input:
- selector: input[placeholder="Add title"]
- value: "Reach out: [Name] — [Reason]"

### Step D: Set date and time
Click the start date field and type the target date (MM/DD/YYYY format).
Set start time to 9:00 AM and end time to 9:30 AM using form_input on time fields.
If time fields require javascript:
```javascript
const timeInputs = [...document.querySelectorAll('input[aria-label*="time" i], input[placeholder*="time" i]')];
if (timeInputs[0]) { timeInputs[0].value = '9:00 AM'; timeInputs[0].dispatchEvent(new Event('input', {bubbles:true})); }
if (timeInputs[1]) { timeInputs[1].value = '9:30 AM'; timeInputs[1].dispatchEvent(new Event('input', {bubbles:true})); }
```

### Step E: Add description
Use mcp__Claude_in_Chrome__form_input or javascript to set the description field:
```
Contact: [Name]
Title: [Current Title] at [Company]
LinkedIn: [LinkedIn URL]

Reason for outreach: [Reason]
[If Birthday] Their birthday is on [Birthday date]. Send a warm message.
[If Notable Event] They [Latest Notable Event] on [Latest Event Date]. Congratulate them.
[If Check-in] It has been [days_since] days since last contact. Reach out to stay top of mind.

Last Contacted: [Last Contacted date or "Never"]
Follow-up Frequency: [Follow-up Frequency]
```

If form_input fails on a contenteditable div, use javascript_tool:
```javascript
const desc = document.querySelector('[contenteditable][aria-label*="description" i]') ||
             document.querySelector('[data-testid="description-input"]');
if (desc) { desc.focus(); document.execCommand('insertText', false, 'DESCRIPTION_TEXT'); }
```

### Step F: Save
```javascript
const saveBtn = [...document.querySelectorAll('button')].find(
  b => b.textContent.trim().toLowerCase() === 'save'
);
if (saveBtn) saveBtn.click();
```

### Step G: Verify
Call mcp__Claude_in_Chrome__get_page_text to confirm return to calendar grid.
Log: "Created event: [event title] on [date]"

## Step 6: Output Summary

After processing all contacts, output a summary:
```
## Weekly Follow-up Reminders — [today's date]

**Overdue contacts found:** [count]
**Calendar events created:** [count]

| Contact | Reason | Event Date |
|---------|--------|------------|
| [Name] | [Reason] | [Date] |
...

**Skipped (As-Needed or up to date):** [count]
```

If no overdue contacts are found, output: "No overdue contacts this week. All contacts are up to date."
```

- [ ] **Step 3: Verify the task was created**

Call `mcp__scheduled-tasks__list_scheduled_tasks`. Confirm "Personal CRM — Weekly Follow-up Reminders" appears in the list with the correct schedule (weekly, Monday, 8:00 AM).

- [ ] **Step 4: Commit**

```bash
git add docs/personal-crm/workflows/followup-reminder-instructions.md
git commit -m "feat: register weekly follow-up reminder scheduled task with self-contained prompt"
```

---

## Chunk 4: End-to-End Test

### Task 4: Manually Trigger Reminder Workflow on Test Contact

**Files:**
- No new files. This task validates the full pipeline.

This test sets a known contact to "overdue" state, triggers the workflow manually, verifies a Google Calendar event is created, and resets the contact.

- [ ] **Step 1: Identify the test contact from Plan 1**

Open `docs/personal-crm/notion-ids.md` or query the database with `mcp__notion__API-query-data-source` to find the smoke-test contact created in Plan 1 Task 10. Note their page ID.

- [ ] **Step 2: Set the contact to an overdue state**

Call `mcp__notion__API-patch-page` on the test contact's page ID with:
- `Last Contacted`: set to 2024-11-07 (400 days before 2026-03-17)
- `Follow-up Frequency`: "Annually"
- `Current Title`: whatever was set in Plan 1 (or "Software Engineer" if unknown)
- `Company`: whatever was set in Plan 1 (or "Test Corp" if unknown)

This ensures the contact will be flagged as overdue (400 days > 365 day annual threshold).

- [ ] **Step 3: Run the reminder logic manually**

Execute the overdue query and reason logic by hand (as if you were the scheduled task):

1. Call `mcp__notion__API-query-data-source` on the database and find the test contact.
2. Confirm `days_since` = approximately 400 days → overdue under "Annually" threshold.
3. Check Birthday: is their birthday within the next 14 days? (Likely no for a test contact.)
4. Check Latest Event Date: is it within the last 30 days? (Likely no.)
5. Expected reason: "Check-in"
6. Expected event title: "Reach out: [Test Contact Name] — Check-in"
7. Expected date: next weekday after 2026-03-17 = 2026-03-18 (Wednesday)

- [ ] **Step 4: Create the Google Calendar event via Chrome MCP**

Follow the Chrome MCP steps from Chunk 2 / Task 2 exactly:
- Navigate to calendar.google.com
- Create a new event with:
  - Title: "Reach out: [Test Contact Name] — Check-in"
  - Date: 2026-03-18 (next weekday)
  - Time: 9:00 AM – 9:30 AM
  - Description: filled from the contact's actual data

- [ ] **Step 5: Verify the event exists**

Call `mcp__Claude_in_Chrome__navigate` to calendar.google.com. Navigate to 2026-03-18 in day view. Call `mcp__Claude_in_Chrome__get_page_text` and confirm the event title appears.

- [ ] **Step 6: Reset the test contact**

Call `mcp__notion__API-patch-page` on the test contact's page ID with:
- `Last Contacted`: today's date (2026-03-17)

This marks them as "contacted today" so they won't surface again until next year.

- [ ] **Step 7: Delete the test calendar event**

Navigate to the test event on Google Calendar and delete it. This was a verification event only.

- [ ] **Step 8: Confirm end-to-end success**

Verify all of the following:
- [ ] Query returned the test contact as overdue
- [ ] Correct reason determined ("Check-in" since no birthday/event triggers apply)
- [ ] Event was created with correct title, date (2026-03-18), and time (9:00–9:30 AM)
- [ ] Event description included contact name, title, company, and check-in reason
- [ ] Test contact's Last Contacted reset to 2026-03-17

_No git commit needed for this task — no files were changed._

---

## Final Verification Checklist

- [ ] `docs/personal-crm/workflows/followup-reminder-instructions.md` exists with all logic sections:
  - Querying for Overdue Contacts
  - Overdue Threshold Logic (monthly/quarterly/annually thresholds)
  - Reason-for-Outreach Logic (birthday → notable event → check-in priority)
  - Google Calendar Event Format
  - Chrome MCP Steps for Creating a Google Calendar Event
- [ ] Chrome MCP steps were verified with a manual test event (then deleted)
- [ ] Weekly scheduled task exists in `mcp__scheduled-tasks__list_scheduled_tasks` with schedule: Monday 8:00 AM
- [ ] Scheduled task prompt is fully self-contained (contains database ID, all thresholds, all Chrome steps)
- [ ] End-to-end test passed: test contact surfaced as overdue, calendar event created and verified
- [ ] Test contact's Last Contacted reset to today after the test
- [ ] Test calendar event deleted after verification

---

## Reference: Overdue Logic Quick Reference

| Follow-up Frequency | Overdue After | Days Threshold |
|---|---|---|
| Monthly | 30 days | > 30 |
| Quarterly | 90 days | > 90 |
| Annually | 365 days | > 365 |
| As-Needed | Never surfaced | (skipped) |
| (empty) | Never surfaced | (skipped) |
| Last Contacted = empty | Always overdue | N/A |

## Reference: Reason-for-Outreach Priority

| Priority | Condition | Event Title Suffix |
|---|---|---|
| 1 | Birthday within next 14 days | "Birthday message" |
| 2 | Latest Event Date within last 30 days AND not yet contacted since event | "Congratulate: [Latest Notable Event]" |
| 3 | Default | "Check-in" |

## Reference: Google Calendar Event Template

```
Title:       Reach out: [Name] — [Reason]
Date:        [Next weekday after today]
Time:        9:00 AM – 9:30 AM
Description:
  Contact: [Name]
  Title: [Current Title] at [Company]
  LinkedIn: [LinkedIn URL]

  Reason for outreach: [Reason]
  [Context line based on reason type]

  Last Contacted: [date or "Never"]
  Follow-up Frequency: [frequency]
```
