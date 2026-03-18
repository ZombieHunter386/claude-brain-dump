# LinkedIn CSV Export + Import Guide

## Step 1: Export Your LinkedIn Connections

1. Go to LinkedIn → click **Me** (top nav) → **Settings & Privacy**
2. Click **Data Privacy** in the left nav
3. Click **Get a copy of your data**
4. Select **Connections** only (faster than full archive)
5. Click **Request archive**
6. LinkedIn emails you a download link (usually within 10 minutes)
7. Download and unzip — the file is named `Connections.csv`

### What LinkedIn Provides in the CSV

| LinkedIn Field | Maps to Notion Property |
|---|---|
| First Name + Last Name | Name |
| URL | LinkedIn URL |
| Position | Current Title |
| Company | Company |
| Connected On | Connection Detail (e.g., "LinkedIn connection since 2021-03-15") |
| Email Address | **NOT stored** (privacy) |

---

## Step 2: Import into Notion

### How to run it

Tell Claude: `"Import my LinkedIn connections from [file path to Connections.csv]"`

Example: `"Import my LinkedIn connections from ~/Downloads/Connections.csv"`

### What Claude does

For each row in the CSV, Claude calls `notion-create-pages` with:
- **Name** = First Name + " " + Last Name
- **LinkedIn URL** = URL field (normalized: lowercase, no trailing slash)
- **Current Title** = Position field
- **Company** = Company field
- **Connection Detail** = "LinkedIn connection since [Connected On date]"
- **Follow-up Frequency** = "Annually" (default — update key contacts after import)

### What Claude skips
- Rows where URL is empty
- Duplicate contacts (same LinkedIn URL already exists in the database)

### LinkedIn URL normalization
LinkedIn URLs can vary in format (e.g., trailing slashes, `https://www.linkedin.com/in/username/` vs `https://linkedin.com/in/username`). Claude normalizes all URLs to lowercase with no trailing slash before duplicate checking.

---

## Step 3: After Import

1. **Enrich key contacts** (Plan 2) — Run enrichment on your top 20–30 contacts to fill in:
   Industry, Professional Expertise, Personal Interests, Birthday
2. **Update Follow-up Frequency** for contacts you want to stay close to (change from "Annually" to "Monthly" or "Quarterly")
3. **Fill in your Job Target Profile page** — required before running job search queries (Plan 5)
   - URL: https://www.notion.so/32788694d77c81c498e2cc5734a79d09

---

## Notes

- LinkedIn allows up to 30,000 connections. For large imports, Claude processes in batches of 50 to avoid rate limits.
- The CSV export only includes 1st-degree connections, not followers.
- Position and Company in the CSV reflect what the person had on LinkedIn **at the time you connected**, not their current role. Run enrichment (Plan 2) to update stale data.
