# Job Search Query Workflow — Claude Instructions

## Overview

This document defines the exact steps Claude follows when a user asks a job-search-related
question such as:
- "Who in my CRM could help me find a role in commercial real estate?"
- "I'm targeting real estate analyst roles in Chicago — who should I reach out to?"
- "Find me contacts relevant to my job search."

---

## Step 1: Fetch the Job Target Profile

Read the page ID for "My Job Target Profile" from `docs/personal-crm/notion-ids.md`.
Fetch the page using `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch`.

Extract and hold in memory:
- **Target Role** (e.g., "Real Estate Development Analyst, Real Estate Analyst")
- **Industries of Interest** (e.g., ["Commercial Real Estate", "Real Estate Development"])
- **Key Skills / Keywords** (e.g., ["financial modeling", "Excel", "deal underwriting"])
- **Resume Summary** (for context on seniority and background)
- **What I'm Looking For** (for context on environment, company size, location)

If the page is empty or fields are blank, halt and prompt the user:
"Your Job Target Profile in Notion appears to be empty. Please fill in your target role,
industries, and key skills before running a job search query."

---

## Step 2: Query the Professional Contacts Database

Read the database ID for "Professional Contacts" from `docs/personal-crm/notion-ids.md`.
Retrieve all contacts using `mcp__notion__API-query-data-source`.

If pagination cursor is returned, fetch subsequent pages until all contacts are retrieved.

Fields to extract per contact:
- Name (title)
- LinkedIn URL (url)
- Current Title (rich_text)
- Company (rich_text)
- Industry (multi_select)
- Professional Expertise (multi_select)
- Personal Interests (multi_select)
- Last Contacted (date)
- Connection Detail (rich_text)
- Open Job Posting URL (url)
- Latest Notable Event (rich_text)
- Latest Event Date (date)
- Follow-up Frequency (select)
- Notes (rich_text)

---

## Relevance Scoring Rubric

Scores are additive. Maximum possible score: 10 points.

| Criterion | Points | How to Check |
|---|---|---|
| Industry match | +3 | Contact's `Industry` contains at least one tag matching any item in the user's "Industries of Interest". Case-insensitive. |
| Professional Expertise overlap | +2 | Contact's `Professional Expertise` shares 2+ tags with the user's "Key Skills / Keywords". Partial string match acceptable. |
| Decision-maker or hiring signal in title | +2 | `Current Title` contains (case-insensitive): "Director", "VP", "Vice President", "Head of", "Managing Director", "Principal", "Partner", "Manager", "Recruiter", "Talent", "Hiring", "Acquisition". |
| Open Job Posting URL is populated | +1 | `Open Job Posting URL` field is non-empty. |
| Warm relationship | +1 | `Last Contacted` date is within the last 6 months. |
| Strong connection | +1 | `Connection Detail` contains a shared school, shared employer, "met in person", "close friend", or "introduction from". |

### Tie-Breaking
1. More recent `Last Contacted` date wins.
2. More overlapping `Industry` tags wins.
3. Alphabetical by name.

### Scoring Notes
- Contacts with no `Industry`, `Professional Expertise`, or `Current Title` score 0 and appear last (or omitted if top-10 is met).
- Scoring is computed in memory — no Notion fields are written during a job search query.
- The user's query text can boost contacts whose Company, Notes, or Latest Notable Event contain matching keywords (secondary tie-breaker, no point bonus).

---

## Step 3: Score Each Contact

Apply the rubric above to every contact retrieved in Step 2.

---

## Step 4: Rank and Filter

- Sort all contacts by score descending.
- Return top 5–10 contacts (top 10 for broad queries; top 5 for narrow).
- Exclude contacts with score of 0 unless fewer than 5 contacts total.
- Apply tie-breaking rules as specified.

---

## Step 5: Format the Output

Use the output format below. Return results inline in the conversation. Do not write
results to Notion unless the user explicitly asks.

---

## Output Format

```
## Job Search Results for: "<user's query>"
*Profile used: [Target Role] | Industries: [list] | Key Skills: [list]*
*Contacts evaluated: N | Results shown: M*

---

### 1. [Full Name] — [Current Title] at [Company] (Score: X/10)
**Why relevant:** [1–2 sentences explaining which scoring criteria fired and why this
person is a strong match. Be specific — name the overlapping industry, skill, or title signal.]
**Connection:** [One sentence from Connection Detail, or "No connection detail on file."]
**Last contacted:** [Date, or "Never" if blank]
**Open posting:** [URL if populated, or "None on file"]
**Live check:** [Result of Chrome MCP check if triggered, or omit if not triggered]
**Suggested outreach:** [1–2 sentence personalized suggestion. Reference Latest Notable Event
if populated. Otherwise, suggest a warm re-engagement or intro approach based on title and target role.]

---

### 2. [Full Name] — [Current Title] at [Company] (Score: X/10)
[same structure]

---

*To update a contact's job posting URL or log an outreach, ask Claude to update that
contact's record in the Professional Contacts database.*
```

### Output Format Rules

1. Always include the profile summary line so the user can verify the correct profile was used.
2. Always include "Contacts evaluated: N".
3. "Suggested outreach" must be personalized — never generic. Use Latest Notable Event,
   Connection Detail, or the user's stated goal to make it specific.
4. If Open Job Posting URL is populated, always include it.
5. If fewer than 3 contacts score above 0, add: "Few strong matches found. Consider adding more
   contacts or expanding your Industries of Interest in your Job Target Profile."
6. Never return more than 10 contacts. Never return score-0 contacts unless total pool is below 5.

---

## Step 6: Optional — Live Job Posting Check (Chrome MCP)

Trigger only if ALL of the following are true:
1. Contact is in the top 3 ranked results.
2. Their Open Job Posting URL is empty OR Latest Event Date is more than 3 months ago.
3. User has not said "skip live check" or "no browser".

For each qualifying contact:
1. Navigate to their company's LinkedIn jobs page using the numeric company ID from their profile links.
2. Extract job listings using `mcp__Claude_in_Chrome__javascript_tool` with `document.body.innerText`.
3. Filter for roles overlapping with user's Target Role or Key Skills.
4. If relevant postings found, include under "Live postings found."
5. If LinkedIn requires login, skip and note "(Live check unavailable — login required)".

---

## Error Handling

| Situation | Action |
|---|---|
| Job Target Profile page not found | Halt, prompt user to verify page ID in notion-ids.md |
| Profile fields all blank | Halt, prompt user to fill in the profile |
| Database has 0 contacts | Return "No contacts found. Add contacts first." |
| Database has <5 contacts | Return all contacts ranked, note more contacts = better results |
| Chrome MCP live check fails | Skip silently, note "(Live check unavailable)" in output |
| Contact missing Industry and Expertise | Score as 0; include only if result pool is below 5 |
