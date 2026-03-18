# Personal CRM — Job Search Query Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable an on-demand workflow where the user describes a job search goal in natural language and Claude fetches their Job Target Profile, queries the Professional Contacts database, scores and ranks relevant contacts, and returns a prioritized list with explanations and outreach suggestions.
**Architecture:** Claude fetches the "My Job Target Profile" page from Notion to understand the user's current target role and key skills, then queries the Professional Contacts database to retrieve all contacts, applies an in-memory relevance scoring rubric across multiple fields, and returns a ranked, formatted list; optionally, for top contacts with missing job posting data, Chrome MCP is used to check live job pages.
**Tech Stack:** Notion MCP (`mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch`, `mcp__notion__API-query-data-source`, `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-search`), Chrome MCP (`mcp__Claude_in_Chrome__navigate`, `mcp__Claude_in_Chrome__get_page_text`), Notion IDs from `docs/personal-crm/notion-ids.md`.
**Prerequisite:** Plan 1 (Foundation) complete, Job Target Profile filled in.

---

## Chunk 1: Job Search Query Instructions Document

### Task 1: Create the Workflow Instructions Document
**Files:**
- Create: `docs/personal-crm/workflows/job-search-instructions.md`

- [ ] **Step 1: Create the `docs/personal-crm/workflows/` directory and the instructions file.**
  Create `docs/personal-crm/workflows/job-search-instructions.md` with the following structure and content:

  ```markdown
  # Job Search Query Workflow — Claude Instructions

  ## Overview
  This document defines the exact steps Claude follows when a user asks a job-search-related
  question such as:
  - "Who in my CRM could help me find a role in commercial real estate?"
  - "I'm targeting data analyst roles at PropTech companies — who should I reach out to?"
  - "Find me contacts relevant to my job search."

  ---

  ## Step 1: Fetch the Job Target Profile

  Read the page ID for "My Job Target Profile" from `docs/personal-crm/notion-ids.md`.
  Fetch the page using:

  ```
  mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch
  url: https://www.notion.so/<JOB_TARGET_PROFILE_PAGE_ID>
  ```

  Extract and hold in memory:
  - **Target Role** (e.g., "Real Estate Analyst, Asset Manager")
  - **Industries of Interest** (e.g., ["Commercial Real Estate", "PropTech"])
  - **Key Skills / Keywords** (e.g., ["financial modeling", "Argus", "Excel", "market analysis"])
  - **Resume Summary** (for context on seniority and background)
  - **What I'm Looking For** (for context on environment, company size, culture)

  If the page is empty or the fields are blank, halt and prompt the user:
  "Your Job Target Profile in Notion appears to be empty. Please fill in your target role,
  industries, and key skills before running a job search query."

  ---

  ## Step 2: Query the Professional Contacts Database

  Read the database ID for "Professional Contacts" from `docs/personal-crm/notion-ids.md`.
  Retrieve all contacts using:

  ```
  mcp__notion__API-query-data-source
  database_id: <PROFESSIONAL_CONTACTS_DB_ID>
  page_size: 100
  ```

  If there are more than 100 contacts (pagination cursor returned), fetch subsequent pages
  until all contacts are retrieved.

  Fields to extract per contact:
  - Name (title)
  - LinkedIn URL (url)
  - Current Title (rich_text)
  - Company (rich_text)
  - Industry (multi_select — list of tags)
  - Professional Expertise (multi_select — list of tags)
  - Personal Interests (multi_select)
  - Last Contacted (date)
  - Connection Detail (rich_text)
  - Open Job Posting URL (url)
  - Latest Notable Event (rich_text)
  - Latest Event Date (date)
  - Follow-up Frequency (select)
  - Notes (rich_text)

  ---

  ## Step 3: Score Each Contact (Relevance Scoring Rubric)

  For each contact, compute a relevance score using the rubric defined in Chunk 2 of the
  implementation plan (see `docs/superpowers/plans/2026-03-17-personal-crm-job-search.md`).

  The full rubric is also reproduced inline below in the scoring section of these instructions.

  ---

  ## Step 4: Rank and Filter

  - Sort all contacts by score descending.
  - Return the top 5–10 contacts (use top 10 if the user's query is broad; top 5 if narrow).
  - Exclude contacts with a score of 0 unless there are fewer than 5 contacts in the database.
  - If two contacts tie in score, rank the one with a more recent Last Contacted date higher
    (warmer relationship wins a tiebreak).

  ---

  ## Step 5: Format the Output

  Use the output format defined in Chunk 3 of the implementation plan. Return results inline
  in the conversation. Do not write results to Notion unless the user explicitly asks.

  ---

  ## Step 6: Optional — Live Job Posting Check (Chrome MCP)

  Trigger this step only if ALL of the following are true:
  1. The contact is in the top 3 ranked results.
  2. Their Open Job Posting URL field is empty OR the Latest Event Date is more than
     3 months ago (posting may be stale).
  3. The user has not explicitly said "skip live check" or "no browser".

  For each qualifying contact:
  1. Navigate to their company's LinkedIn jobs page:
     ```
     mcp__Claude_in_Chrome__navigate
     url: https://www.linkedin.com/company/<company-slug>/jobs/
     ```
     If the company's LinkedIn URL is not known, construct a search URL:
     ```
     https://www.linkedin.com/jobs/search/?keywords=<job-title>&company=<company-name>
     ```
  2. Extract job listings using:
     ```
     mcp__Claude_in_Chrome__get_page_text
     ```
  3. Filter for roles that overlap with the user's Target Role or Key Skills.
  4. If relevant postings are found, include them in the contact's output block under
     "Live postings found."
  5. If LinkedIn requires login or access is blocked, skip silently and note
     "(Live check unavailable — login required)" in the output.

  ---

  ## Error Handling

  | Situation | Action |
  |---|---|
  | Job Target Profile page not found | Halt and prompt user to verify page ID in notion-ids.md |
  | Profile fields all blank | Halt and prompt user to fill in the profile |
  | Database has 0 contacts | Return "No contacts found. Add contacts to your Professional Contacts database first." |
  | Database has <5 contacts | Return all contacts ranked, note that more contacts = better results |
  | Chrome MCP live check fails | Skip silently, note "(Live check unavailable)" in output |
  | Contact is missing Industry and Expertise fields | Score as 0; include only if total result count is below 5 |
  ```

- [ ] **Step 2: Verify** — Open the file and confirm all six steps are present, the error handling table renders correctly, and the MCP tool names match exactly those listed in the plan.

- [ ] **Step 3: Commit** — `git add docs/personal-crm/workflows/job-search-instructions.md` and commit with message `docs: add job search query workflow instructions for Personal CRM`.

---

## Chunk 2: Relevance Scoring Logic

### Task 2: Define and Document the Scoring Rubric
**Files:**
- Modify: `docs/personal-crm/workflows/job-search-instructions.md` (add scoring section)

- [ ] **Step 1: Insert the scoring rubric** into the instructions document under a new `## Relevance Scoring Rubric` section, placed between Step 2 and Step 3 of the workflow. The rubric:

  ```markdown
  ## Relevance Scoring Rubric

  Scores are additive. Maximum possible score: 10 points.

  | Criterion | Points | How to Check |
  |---|---|---|
  | Industry match | +3 | Contact's `Industry` multi-select contains at least one tag that matches any item in the user's "Industries of Interest" from the Job Target Profile. Match is case-insensitive. |
  | Professional Expertise overlap | +2 | Contact's `Professional Expertise` multi-select shares 2 or more tags with the user's "Key Skills / Keywords". Partial string match is acceptable (e.g., "financial modeling" matches "Argus / Financial Modeling"). |
  | Decision-maker or hiring signal in title | +2 | Contact's `Current Title` contains any of the following (case-insensitive): "Director", "VP", "Vice President", "Head of", "Managing Director", "Principal", "Partner", "Manager", "Recruiter", "Talent", "Hiring", "Acquisition". |
  | Open Job Posting URL is populated | +1 | `Open Job Posting URL` field is non-empty. |
  | Warm relationship (contacted recently) | +1 | `Last Contacted` date is within the last 6 months of today's date. |
  | Strong connection | +1 | `Connection Detail` field contains any of: a shared school name, a shared former employer name, or phrases like "met in person", "close friend", "introduction from". |

  ### Tie-Breaking
  If two contacts share the same total score:
  1. Prefer the contact with a more recent `Last Contacted` date.
  2. If still tied, prefer the contact whose `Industry` tags have more overlapping items with
     Industries of Interest (breadth of industry match).
  3. If still tied, return them in alphabetical order by name.

  ### Scoring Notes
  - A contact with no `Industry`, `Professional Expertise`, or `Current Title` filled in
    receives a score of 0 and appears at the bottom of the list (or is omitted if top-10
    threshold is met by higher-scoring contacts).
  - The scoring is computed in memory by Claude — no Notion database fields are written
    to or modified during a job search query.
  - The user's query itself (e.g., "commercial real estate") can be used as an additional
    signal to boost contacts whose Company, Notes, or Latest Notable Event fields contain
    matching keywords (+0 bonus points, but use as a secondary tie-breaker if needed).
  ```

- [ ] **Step 2: Verify** — Confirm the rubric table renders correctly in markdown, all six criteria are present with correct point values summing to a max of 10, and the tie-breaking logic is clear.

- [ ] **Step 3: Commit** — `git add docs/personal-crm/workflows/job-search-instructions.md` and commit with message `docs: add relevance scoring rubric to job search workflow`.

---

## Chunk 3: Output Format

### Task 3: Define the Output Format
**Files:**
- Modify: `docs/personal-crm/workflows/job-search-instructions.md` (add output format section)

- [ ] **Step 1: Insert the output format specification** into the instructions document under a new `## Output Format` section, placed after the scoring rubric. The format:

  ```markdown
  ## Output Format

  Claude must return results in the following exact structure. Do not use tables — use the
  ranked heading format below for readability.

  ---

  ## Job Search Results for: "<user's query>"
  *Profile used: [Target Role] | Industries: [list] | Key Skills: [list]*
  *Contacts evaluated: N | Results shown: M*

  ---

  ### 1. [Full Name] — [Current Title] at [Company] (Score: X/10)
  **Why relevant:** [1–2 sentences explaining which scoring criteria fired and why this
  person is a strong match. Be specific — name the overlapping industry, skill, or title signal.]
  **Connection:** [One sentence from Connection Detail field, or "No connection detail on file."]
  **Last contacted:** [Date, or "Never" if blank]
  **Open posting:** [URL with job title if Open Job Posting URL is populated, or "None on file"]
  **Live check:** [Result of Chrome MCP check if triggered, or omit this line if not triggered]
  **Suggested outreach:** [1–2 sentence personalized suggestion. Reference Latest Notable Event
  if populated (e.g., "Congratulate on [event]"). Otherwise, suggest a warm re-engagement or
  cold intro approach based on their title and the user's target role.]

  ---

  ### 2. [Full Name] — [Current Title] at [Company] (Score: X/10)
  [same structure]

  ---

  [... repeat for all returned contacts ...]

  ---

  *To update a contact's job posting URL or log an outreach, ask Claude to update that
  contact's record in the Professional Contacts database.*
  ```

  ### Output Format Rules

  1. Always include the profile summary line so the user can verify Claude used the right profile.
  2. Always include "Contacts evaluated: N" so the user knows how many records were scanned.
  3. "Suggested outreach" must be personalized — never generic. Use Latest Notable Event,
     Connection Detail, or the user's stated goal to make it specific.
  4. If Open Job Posting URL is populated, always hyperlink it with an approximate job title
     (parse from the URL or Notes field if possible).
  5. If fewer than 3 contacts score above 0, add a note at the end:
     "Few strong matches found. Consider adding more contacts or expanding your Industries of
     Interest in your Job Target Profile."
  6. Never return more than 10 contacts. Never return contacts with a score of 0 unless
     the total results pool is below 5 contacts.

- [ ] **Step 2: Verify** — Read the output format section and confirm the example renders correctly in markdown, all required fields (name, title, company, score, why relevant, connection, last contacted, open posting, suggested outreach) are present, and the format rules are numbered and clear.

- [ ] **Step 3: Commit** — `git add docs/personal-crm/workflows/job-search-instructions.md` and commit with message `docs: add output format specification to job search workflow`.

---

## Chunk 4: End-to-End Test

### Task 4: Test the Workflow
**Files:**
- Read: `docs/personal-crm/notion-ids.md`
- Read: `docs/personal-crm/workflows/job-search-instructions.md`

- [ ] **Step 1: Verify prerequisites.** Confirm that:
  - `docs/personal-crm/notion-ids.md` contains a valid page ID for "My Job Target Profile".
  - `docs/personal-crm/notion-ids.md` contains a valid database ID for "Professional Contacts".
  - The Job Target Profile page in Notion has at least "Target Role" and "Industries of Interest" filled in.
  - The Professional Contacts database has at least 2 contacts with `Industry` or `Professional Expertise` tags populated.

  If any prerequisite is missing, stop and instruct the user to complete Plan 1 (Foundation) first.

- [ ] **Step 2: Run a test query.** With the instructions file loaded, trigger the workflow with the prompt:
  > "Who can help me find a real estate analyst role?"

  Follow job-search-instructions.md exactly:
  1. Fetch the Job Target Profile page via `mcp__53b43db7-f28a-4045-ad2f-198c5c8dada1__notion-fetch`.
  2. Query the Professional Contacts database via `mcp__notion__API-query-data-source`.
  3. Score each contact using the rubric from Chunk 2.
  4. Rank contacts and prepare the top results.
  5. Format output per the template from Chunk 3.

- [ ] **Step 3: Verify output correctness.** Confirm:
  - The profile summary line shows the correct Target Role and Industries from Notion.
  - "Contacts evaluated: N" reflects the actual number of records in the database.
  - Contacts are ranked in descending score order.
  - The highest-scoring contact has at least one scoring criterion called out under "Why relevant."
  - "Suggested outreach" is specific (references a field value, not generic filler text).
  - If a contact has an Open Job Posting URL, it appears in their result block.

- [ ] **Step 4: Run a second test query** with a more specific prompt:
  > "I'm looking for a data analyst role at a PropTech company — who should I reach out to?"

  Verify that:
  - Only contacts with PropTech or related industry tags appear near the top.
  - Contacts missing both Industry and Professional Expertise score 0 and appear last (or are omitted).
  - The output header reflects the new query text.

- [ ] **Step 5: Finalize** — The end-to-end test produces no file changes. Verify `git status` is clean. If any corrections to `job-search-instructions.md` were needed during testing, stage and commit them now:
```bash
git add docs/personal-crm/workflows/job-search-instructions.md
git commit -m "fix: refine job search instructions based on end-to-end test"
```
If no corrections were needed, no commit is required.

---

## Summary

This plan produces one primary deliverable: `docs/personal-crm/workflows/job-search-instructions.md`, a self-contained workflow document that Claude follows verbatim when answering job-search queries. It covers:

1. How to fetch and parse the Job Target Profile from Notion.
2. How to retrieve all contacts from the Professional Contacts database.
3. A 6-criterion relevance scoring rubric (max 10 points) with explicit tie-breaking rules.
4. An exact output format template with per-contact fields and formatting rules.
5. An optional Chrome MCP branch for live job posting checks on top contacts.
6. Error handling for all common failure cases.

No code is written. The "implementation" is the instructions document itself — the workflow is entirely prompt-driven, using Claude as the runtime engine and Notion/Chrome MCP as data sources.
