# Personal CRM — Follow-up Reminder Instructions

This document defines the logic Claude follows when the weekly follow-up reminder
scheduled task fires, or when manually triggered by the user.

---

## Querying for Overdue Contacts

Use `mcp__notion__API-query-data-source` with the Professional Contacts database ID
(from `docs/personal-crm/notion-ids.md`).

Fetch all contacts. For each contact, read these properties:
- Name (title)
- Last Contacted (date)
- Follow-up Frequency (select)
- Latest Notable Event (rich_text)
- Latest Event Date (date)
- Birthday (date)
- Current Title (rich_text)
- Company (rich_text)
- LinkedIn URL (url)

Apply the overdue threshold logic in memory (not via Notion filter) so the logic
is transparent and auditable.

---

## Overdue Threshold Logic

Today's date is the date the task fires. For each contact:

1. If Follow-up Frequency = "As-Needed" or is empty → SKIP.

2. If Last Contacted is empty (null/blank) → treat as OVERDUE regardless of frequency.
   Reason: no record of ever reaching out.

3. If Last Contacted is set, compute days since Last Contacted:
   days_since = (today - Last Contacted) in whole days

   - Monthly   → overdue if days_since > 30
   - Quarterly → overdue if days_since > 90
   - Annually  → overdue if days_since > 365

4. If NOT overdue → SKIP.

5. If OVERDUE → proceed to Reason-for-Outreach Logic below.

---

## Reason-for-Outreach Logic

Evaluate conditions top-to-bottom. Use the FIRST match.

**Priority 1 — Birthday coming up:**
- Condition: Birthday is set AND the birthday (month + day) falls within the next
  14 calendar days from today (year-agnostic — compare month/day only).
- Reason text: "Birthday message"
- Event title: "Reach out: [Name] — Birthday message"

**Priority 2 — Recent notable event not yet acknowledged:**
- Condition: Latest Event Date is set AND Latest Event Date is within the last 30 days
  AND (Last Contacted is empty OR Last Contacted < Latest Event Date)
- Reason text: "Congratulate: [Latest Notable Event]"
  (If Latest Notable Event is empty: "Congratulate on recent news")
- Event title: "Reach out: [Name] — Congratulate: [Latest Notable Event]"

**Priority 3 — Default check-in:**
- Condition: none of the above matched
- Reason text: "Check-in"
- Event title: "Reach out: [Name] — Check-in"

---

## Google Calendar Event Format

For each overdue contact, create ONE Google Calendar event with:

- **Title:** "Reach out: [Name] — [Reason]"

- **Date:** Next available weekday from today (do NOT use today itself):
  - Monday–Thursday → tomorrow
  - Friday/Saturday/Sunday → next Monday

- **Time:** 9:00 AM – 9:30 AM (30-minute block)

- **Description:**
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

---

## Chrome MCP Steps for Creating a Google Calendar Event

Repeat these steps for EACH overdue contact. Create one event at a time.

### Step A: Navigate to Google Calendar

```
mcp__Claude_in_Chrome__navigate({ url: "https://calendar.google.com" })
```

Call `mcp__Claude_in_Chrome__javascript_tool` with `document.title` to confirm calendar loaded.
If redirected to login, STOP: "Google Calendar requires sign-in. Please log in at calendar.google.com in Chrome, then re-run."

### Step B: Open the New Event dialog

```javascript
// Click "+ Create" button
const btn = document.querySelector('[data-view="CREATE"]') ||
            [...document.querySelectorAll('button')].find(b => b.textContent.trim() === '+ Create') ||
            document.querySelector('button[aria-label="Create"]');
if (btn) btn.click();
```

Wait ~1 second, then click "More options":
```javascript
const moreOpts = [...document.querySelectorAll('a, button')].find(
  el => el.textContent.trim().toLowerCase().includes('more options')
);
if (moreOpts) moreOpts.click();
```

### Step C: Fill in the event title

Use `mcp__Claude_in_Chrome__form_input`:
- selector: `input[placeholder="Add title"]`
- value: "Reach out: [Name] — [Reason]"

### Step D: Set the event date and time

Click the start date field and enter the target date (MM/DD/YYYY format).
Set start time to 9:00 AM and end time to 9:30 AM using `form_input`.

If time fields require JavaScript:
```javascript
const timeInputs = [...document.querySelectorAll('input[aria-label*="time" i], input[placeholder*="time" i]')];
if (timeInputs[0]) { timeInputs[0].value = '9:00 AM'; timeInputs[0].dispatchEvent(new Event('input', {bubbles:true})); }
if (timeInputs[1]) { timeInputs[1].value = '9:30 AM'; timeInputs[1].dispatchEvent(new Event('input', {bubbles:true})); }
```

### Step E: Add the event description

Use `mcp__Claude_in_Chrome__form_input` on the description field, or JavaScript:
```javascript
const desc = document.querySelector('[contenteditable][aria-label*="description" i]') ||
             document.querySelector('[data-testid="description-input"]');
if (desc) { desc.focus(); document.execCommand('insertText', false, 'DESCRIPTION_TEXT'); }
```

### Step F: Save the event

```javascript
const saveBtn = [...document.querySelectorAll('button')].find(
  b => b.textContent.trim().toLowerCase() === 'save'
);
if (saveBtn) saveBtn.click();
```

### Step G: Verify

Call `mcp__Claude_in_Chrome__javascript_tool` with `document.title` to confirm return to calendar.
Log: "Created event: [event title] on [date]"

---

## Output Summary Format

```
## Weekly Follow-up Reminders — [today's date]

**Overdue contacts found:** [count]
**Calendar events created:** [count]

| Contact | Reason | Event Date |
|---------|--------|------------|
| [Name] | [Reason] | [Date] |

**Skipped (As-Needed or up to date):** [count]
```

If no overdue contacts: "No overdue contacts this week. All contacts are up to date."
