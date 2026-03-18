# Personal CRM — Notion IDs

_Reference file for all Notion IDs used across CRM automation plans._

## Parent Page
- **ID:** 32788694-d77c-8131-8b5c-d752679abfe0
- **Name:** Personal CRM
- **URL:** https://www.notion.so/32788694d77c81318b5cd752679abfe0

## Contacts Database
- **ID:** 5986aae5-fb77-49e3-a2ef-29ac67bb90a6
- **Data Source ID:** 4f999215-0963-4e24-9da9-4bcdf8e3ae57
- **URL:** https://www.notion.so/5986aae5fb7749e3a2ef29ac67bb90a6
- **Name:** Professional Contacts

## Job Target Profile Page
- **ID:** 32788694-d77c-81c4-98e2-cc5734a79d09
- **URL:** https://www.notion.so/32788694d77c81c498e2cc5734a79d09
- **Name:** My Job Target Profile

## Views
- **All Contacts:** view://32788694-d77c-8165-9e95-000c868e0567
- **Follow-up Queue:** view://32788694-d77c-8142-b1e8-000c721366ef
- **Birthdays:** view://32788694-d77c-8171-846a-000c5bb7061a
- **By Industry:** view://32788694-d77c-81d7-a983-000c403b9887
- **Recent Events:** view://32788694-d77c-8116-ba95-000cdd79c06d
- **Job Search:** view://32788694-d77c-814f-b3de-000c994113f3

## Manual Setup Required
### Follow-up Queue view filter
Open the "Follow-up Queue" view in Notion UI and add this filter group:
- OR group 1: Follow-up Frequency = "Monthly" AND Last Contacted < 30 days ago
- OR group 2: Follow-up Frequency = "Quarterly" AND Last Contacted < 90 days ago
- OR group 3: Follow-up Frequency = "Annually" AND Last Contacted < 365 days ago
