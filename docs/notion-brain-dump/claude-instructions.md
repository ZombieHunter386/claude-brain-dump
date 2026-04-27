# Brain Dump — Claude Instructions

Read this file at the start of any session where the user wants to capture or retrieve brain dump entries.

## Database

**Name:** Brain Dump Entries
**Notion Database ID:** 4239eb81-babc-4751-bd05-41039420c264

## MCP Tools

Use ONLY these Notion MCP tools — `create-a-data-source` and `update-a-data-source` are broken and will always fail:

| Action | Tool |
|---|---|
| Create an entry (Thought, Idea, Task, Sub-task) | `mcp__notion__API-post-page` |
| Query / search entries | `mcp__notion__API-query-data-source` |
| Retrieve a specific entry | `mcp__notion__API-retrieve-a-page` |
| Update an existing entry | `mcp__notion__API-patch-page` |
| Delete an entry | `mcp__notion__API-delete-a-block` |
| Get database schema | `mcp__notion__API-retrieve-a-database` |

## Hierarchy

Entries have 4 levels — always respect this structure:

| Emoji | Level | What it is |
|---|---|---|
| 🧠 | Thought | Big vision or overarching concept |
| 💡 | Idea | Specific direction that stems from a Thought |
| ✅ | Task | Actionable work to implement an Idea |
| ↳ | Sub-task | Step that makes up a Task |

**How nesting works:** Sub-items use a self-referencing relation property called **"Parent item"**. To create a child entry under a parent, include `"Parent item": { "relation": [{ "id": "<parent-page-id>" }] }` in the properties when calling `post-page`.

## Capture Workflow

When the user sends a brain dump (voice or text):

1. **Parse** — extract meaning and sub-components. If nothing classifiable, ask them to rephrase. Do NOT create blank entries.
2. **Classify** — assign Level and Area to each component. If it spans multiple Areas (e.g., "fix Compass intake flow AND redesign my personal site"), split into separate Thoughts — one per Area — and confirm both.
3. **Scan for existing Thoughts** — call `mcp__notion__API-query-data-source` on database `4239eb81-babc-4751-bd05-41039420c264` with filter `Level = Thought`. If that returns errors, fall back to `mcp__notion__API-post-search` with relevant keywords. Compare results against existing Thought titles and Summaries for keyword or concept overlap.
4. **Route:**
   - **Clear match** (keyword/concept overlap with one existing Thought) → attach at the appropriate level beneath it; confirm to user afterward: "Added *X* as an Idea under *Y*"
   - **Ambiguous match** (two or more plausible parent Thoughts) → ask user which parent before writing: "This could go under *Y* or *Z* — which fits better?"
   - **No match** → create a new top-level Thought (with nested Ideas/Tasks if enough detail was provided)
   - **Merge opportunity** (new Thought content significantly overlaps an existing standalone Thought) → offer to merge before writing: "This seems related to *[existing Thought]* — want me to combine them?"
5. **Write** — use `mcp__notion__API-post-page`. Set Area on EVERY row created (Thought and all children). Set "Parent item" relation for all non-Thought entries.
6. **Generate Tasks** — after writing every Thought or Idea, always generate 2-3 Tasks under it. See Task Generation rules below.
7. **Confirm** — tell user exactly what was saved and where, including tasks created.

## Creating an Entry — Property Format

```json
{
  "parent": { "database_id": "4239eb81-babc-4751-bd05-41039420c264" },
  "properties": {
    "Name": { "title": [{ "text": { "content": "🧠 Your title here" } }] },
    "Level": { "select": { "name": "Thought" } },
    "Area": { "multi_select": [{ "name": "Work" }] },
    "Status": { "select": { "name": "Captured" } },
    "Summary": { "rich_text": [{ "text": { "content": "2-3 sentence recap." } }] },
    "Details": { "rich_text": [{ "text": { "content": "- Intent: ...\n- Constraints: ...\n- Requirements: ..." } }] },
    "Date Captured": { "date": { "start": "YYYY-MM-DD" } },
    "Parent item": { "relation": [{ "id": "<parent-page-id>" }] }
  }
}
```

Omit "Parent item" for top-level Thoughts. Include it for Ideas, Tasks, and Sub-tasks.

## Entry Properties

Every entry needs all of these:

| Property | Value |
|---|---|
| Name | Claude-generated 1-line summary with emoji prefix (🧠 💡 ✅ ↳) |
| Level | Thought / Idea / Task / Sub-task |
| Area | multi_select — inherit from parent on all child entries |
| Status | Always "Captured" on creation |
| Summary | 2-3 sentence recap |
| Details | 3-5 bullet points: intent (what and why), constraints, specific requirements |
| Date Captured | Today's date |
| Parent item | Set for all non-Thought entries — relation to parent page ID |

## Task Generation

**Always generate 2-3 Tasks under every new Thought or Idea.** Do not wait to be asked. Tasks make entries actionable immediately.

### Rules

- **Derive tasks from the Details "Next step" and the Summary** — read both fields and extract concrete, specific actions
- **Each task should be completable in a single sitting** — not vague goals, but specific doable actions
- **Create Tasks in parallel with or immediately after the parent Thought/Idea** — get the parent ID first, then post tasks using "Parent item" relation
- **Task names start with ✅** and describe the action clearly (e.g., "✅ Export LinkedIn connections as CSV")
- **Task Summary**: 1-2 sentences describing the outcome of completing this task
- **Task Details**: 3-5 bullet points with step-by-step how-to, referencing specific tools, sources, or approaches from the parent entry
- **Task Area**: inherit from the parent entry (same multi_select values)

### Good task patterns (drawn from entry Details)

| Parent Details field | → Task to create |
|---|---|
| "Next step: Research X" | ✅ Research X — with specific sources listed in Details |
| "Potential tool: Y or Z" | ✅ Evaluate Y vs Z — with decision criteria in Details |
| "Data sources: A, B, C" | ✅ Pull and assess data from A — with steps to access it |
| "Constraint: Check if X exists" | ✅ Search for existing X tools — with specific places to look |
| "Approach: Do X then Y" | ✅ Do X, then ✅ Do Y as separate sequential tasks |

### What NOT to do

- Don't create tasks that are just restatements of the parent Idea name
- Don't create vague tasks like "✅ Work on this" or "✅ Think about next steps"
- Don't skip task generation because the entry seems high-level — every entry has at least one concrete first action

## Status Rules

- Default on creation: **Captured**
- Only change status on explicit user instruction
- "Let's build X" → upgrade to **In Progress**
- Never auto-advance status without being told to

## Retrieval

When user asks to see or pull entries:
- Call `mcp__notion__API-query-data-source` with relevant filters (Area, Level, Status, or keyword search)
- Present results as a tree (Thought → Ideas → Tasks → Sub-tasks)
- For "build X" requests: fetch the full hierarchy for that Thought and present it ready for implementation planning (then invoke the writing-plans skill)

## Views in Notion (for user browsing)

The database has these saved views the user can browse directly:

| View | Shows |
|---|---|
| All Entries | Everything, tree view |
| Work | Area = Work |
| Personal | Area = Personal |
| Compass Pro Bono | Area = Compass Pro Bono |
| Ready to Build | Status = Ready to Build |
| In Progress | Status = In Progress |

## Areas

- **Work** — professional tasks and projects
- **Personal** — personal goals, life plans, random thoughts
- **Compass Pro Bono** — pro bono work for Compass organization
