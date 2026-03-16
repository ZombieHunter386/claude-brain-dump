# Notion Brain Dump System Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up a hierarchical Brain Dump database in Notion with 4-level nesting (Thought → Idea → Task → Sub-task), filtered views by area, and a Claude reference card so future sessions know how to capture and retrieve entries.

**Architecture:** Single Notion database with sub-items enabled. Properties set on every row (Area set explicitly on all levels so filtered views work). No code — all setup is via Notion MCP API calls. Claude behavior is governed by a reference card saved locally and in Claude memory.

**Tech Stack:** Notion MCP (`mcp__notion__*` tools), Claude memory system, local markdown docs.

**Spec:** `docs/superpowers/specs/2026-03-15-notion-brain-dump-design.md`

---

## Chunk 1: Database Setup

### Task 1: Find the Notion workspace root page

**Files:**
- No files created yet

- [ ] **Step 1: Confirm Notion MCP connection**

Call: `mcp__notion__API-get-self`
Expected: Returns bot/user info confirming the integration is active.

- [ ] **Step 2: Search for existing pages to find a parent location**

Call: `mcp__notion__API-post-search` with empty query to list top-level pages.
Expected: Returns a list of pages. Note the ID of a suitable parent page for the database (or confirm workspace-level creation is available).

- [ ] **Step 3: Create the notion-database folder**

```bash
mkdir -p /Users/hunterheyman/Claude/notion-database
```

Expected: Folder `/Users/hunterheyman/Claude/notion-database/` exists.

This folder will store any exported Notion data, database snapshots, or local reference files related to the Brain Dump database.

- [ ] **Step 4: Commit checkpoint**

```bash
git -C /Users/hunterheyman/Claude add docs/superpowers/specs/2026-03-15-notion-brain-dump-design.md docs/superpowers/plans/2026-03-15-notion-brain-dump.md
git -C /Users/hunterheyman/Claude commit -m "feat: add notion brain dump spec, plan docs, and notion-database folder"
```

---

### Task 2: Create the Brain Dump database

**Files:**
- No local files — Notion database created via API

**Important — endpoint selection:** The correct Notion API call for creating a database is `mcp__notion__API-post-page` with a database object (title + properties schema) as the body, NOT `create-a-data-source` (which is for AI connectors). Before executing Step 1, call `mcp__notion__API-retrieve-a-database` on any existing database to confirm the correct schema format, then use `mcp__notion__API-post-page` to create the new database.

- [ ] **Step 1: Create the Brain Dump database**

Call: `mcp__notion__API-post-page` with a database creation body.

Database title: "Brain Dump"
Parent: workspace root or parent page identified in Task 1.

Properties schema:
```
Title (Name)  — built-in title field (always present)
Level         — select: ["Thought", "Idea", "Task", "Sub-task"]
Area          — select: ["Work", "Personal", "Compass Pro Bono"]
Status        — select: ["Captured", "Ready to Build", "In Progress", "Done"]
Summary       — rich_text
Details       — rich_text
Date Captured — date
```

Expected: Database created, returns database ID. Record this ID — needed for all subsequent calls and the reference card.

- [ ] **Step 2: Verify the database was created with correct properties**

Call: `mcp__notion__API-retrieve-a-database` with the returned database ID.
Expected: Returns database with all 7 properties. Confirm select options match exactly.

- [ ] **Step 3: Enable sub-items on the database**

Sub-items must be explicitly enabled as a database setting so the tree view renders correctly. Call: `mcp__notion__API-update-a-data-source` (or the database PATCH endpoint) to set `sub_items: { enabled: true }` on the database.

If the MCP does not expose a direct sub-items toggle, open the database in Notion UI and enable sub-items manually: Database → ⋯ menu → Sub-items → Enable. Confirm it is enabled before proceeding.

Expected: Sub-items enabled. Parent-child rows will render as a collapsible tree in the "All Entries" view.

- [ ] **Step 4: Create the 6 required views**

Notion requires views to be created explicitly. Create each view with the correct filter:

| View | Filter |
|---|---|
| All Entries | No filter — tree view, show all |
| Work | Area = Work |
| Personal | Area = Personal |
| Compass Pro Bono | Area = Compass Pro Bono |
| Ready to Build | Status = Ready to Build |
| In Progress | Status = In Progress |

For each view, call the appropriate MCP endpoint to add a view to the database with the filter above. If a dedicated "create view" endpoint is not available in the MCP, note that views must be created manually in the Notion UI using: Database → Add View → filter as above.

Expected: 6 named views exist on the Brain Dump database.

---

### Task 3: Verify hierarchy with test entries

**Files:**
- No local files

- [ ] **Step 1: Create a top-level Thought entry**

Call: `mcp__notion__API-post-page` with:
```
parent: { database_id: <brain-dump-db-id> }
properties:
  Title: "🧠 TEST — Notion Brain Dump System Setup"
  Level: "Thought"
  Area: "Work"
  Status: "Captured"
  Summary: "Test entry to verify database structure and hierarchy."
  Details: "- Intent: verify Notion setup works end-to-end\n- This entry should be deleted after verification"
  Date Captured: today's date
```
Expected: Page created, returns page ID. Record as <thought-page-id>.

- [ ] **Step 2: Create a child Idea under the Thought**

Call: `mcp__notion__API-post-page` with:
```
parent: { page_id: <thought-page-id> }
properties:
  Title: "💡 TEST — Child Idea"
  Level: "Idea"
  Area: "Work"
  Status: "Captured"
  Summary: "Child idea to test nesting."
  Details: "- Intent: verify that child entries appear under parent in tree view"
  Date Captured: today's date
```
Expected: Page created as sub-item of the Thought. Record as <idea-page-id>.

- [ ] **Step 3: Create a child Task under the Idea**

Call: `mcp__notion__API-post-page` with:
```
parent: { page_id: <idea-page-id> }
properties:
  Title: "✅ TEST — Child Task"
  Level: "Task"
  Area: "Work"
  Status: "Captured"
  Summary: "Child task to test 3rd level nesting."
  Details: "- Intent: verify 3-level hierarchy renders correctly"
  Date Captured: today's date
```
Record as <task-page-id>.

- [ ] **Step 4: Create a Sub-task under the Task**

Call: `mcp__notion__API-post-page` with:
```
parent: { page_id: <task-page-id> }
properties:
  Title: "↳ TEST — Sub-task"
  Level: "Sub-task"
  Area: "Work"
  Status: "Captured"
  Summary: "Sub-task to test 4th level nesting."
  Details: "- Intent: verify deepest hierarchy level works"
  Date Captured: today's date
```

- [ ] **Step 5: Verify hierarchy via API**

Call: `mcp__notion__API-query-data-source` on the Brain Dump database.
Expected: All 4 test entries appear. Confirm each entry's parent field matches the intended parent's page ID.

- [ ] **Step 6: Visually confirm tree view in Notion UI**

Open the Brain Dump database in Notion (via browser or app). In the "All Entries" view, confirm:
- The Thought appears as a top-level row
- Expanding it shows the Idea nested below
- Expanding the Idea shows the Task
- Expanding the Task shows the Sub-task
- The tree collapses and expands correctly

If the tree does not render, sub-items are likely not enabled — revisit Task 2 Step 3.

- [ ] **Step 7: Delete all 4 test entries (leaf-to-root order)**

Call: `mcp__notion__API-delete-a-block` for each test entry in this order:
1. Sub-task
2. Task
3. Idea
4. Thought

Expected: All test entries removed. Database is empty and ready for real use.

---

## Chunk 2: Reference Card and Memory

### Task 4: Write the Claude reference card

**Files:**
- Create: `docs/notion-brain-dump/claude-instructions.md`

This file is what Claude reads at the start of any future brain dump session. It must be fully self-contained — no assumptions that the reader has seen the spec or this plan.

- [ ] **Step 1: Create the reference card**

Write `docs/notion-brain-dump/claude-instructions.md` with this content (insert real database ID in Step 2):

```markdown
# Brain Dump — Claude Instructions

Read this file at the start of any session where the user wants to capture or retrieve brain dump entries.

## Database

**Name:** Brain Dump
**Notion Database ID:** <insert-db-id-from-task-2>

## MCP Tools

Use these Notion MCP tools to interact with the database:

| Action | Tool |
|---|---|
| Create an entry (Thought, Idea, Task, Sub-task) | `mcp__notion__API-post-page` |
| Query / search entries | `mcp__notion__API-query-data-source` |
| Retrieve a specific entry | `mcp__notion__API-retrieve-a-page` |
| Update an existing entry | `mcp__notion__API-patch-page` |
| Delete an entry | `mcp__notion__API-delete-a-block` |

## Hierarchy

Entries have 4 levels — always respect this structure:

| Emoji | Level | What it is |
|---|---|---|
| 🧠 | Thought | Big vision or overarching concept |
| 💡 | Idea | Specific direction that stems from a Thought |
| ✅ | Task | Actionable work to implement an Idea |
| ↳ | Sub-task | Step that makes up a Task |

## Capture Workflow

When the user sends a brain dump (voice or text):

1. **Parse** — extract meaning and sub-components. If nothing classifiable, ask them to rephrase. Do NOT create blank entries.
2. **Classify** — assign Level and Area to each component. If it spans multiple Areas (e.g., "fix Compass intake flow AND redesign my personal site"), split into separate Thoughts — one per Area — and confirm both.
3. **Scan for existing Thoughts** — call `mcp__notion__API-query-data-source` with filter `Level = Thought`. Compare the new entry's title and Summary against existing Thought titles and Summaries for keyword or concept overlap.
4. **Route:**
   - **Clear match** (keyword/concept overlap with one existing Thought) → attach at the appropriate level beneath it; confirm to user afterward: "Added *X* as an Idea under *Y*"
   - **Ambiguous match** (two or more plausible parent Thoughts) → ask user which parent before writing: "This could go under *Y* or *Z* — which fits better?"
   - **No match** → create a new top-level Thought (with nested Ideas/Tasks if enough detail was provided)
   - **Merge opportunity** (new Thought content significantly overlaps an existing standalone Thought) → offer to merge before writing: "This seems related to *[existing Thought]* — want me to combine them?"
5. **Write** — use `mcp__notion__API-post-page`. Set Area on EVERY row created (Thought and all children).
6. **Confirm** — tell user exactly what was saved and where.

## Entry Properties

Every entry needs all of these:

| Property | Value |
|---|---|
| Title | Claude-generated 1-line summary with emoji prefix (🧠 💡 ✅ ↳) |
| Level | Thought / Idea / Task / Sub-task |
| Area | Work / Personal / Compass Pro Bono — set on ALL levels |
| Status | Always "Captured" on creation |
| Summary | 2-3 sentence recap |
| Details | 3-5 bullet points: intent (what and why), constraints, specific requirements |
| Date Captured | Today's date |

## Status Rules

- Default on creation: **Captured**
- Only change status on explicit user instruction
- "Let's build X" → upgrade to **In Progress**
- Never auto-advance status without being told to

## Retrieval

When user asks to see or pull entries:
- Call `mcp__notion__API-query-data-source` with relevant filters (Area, Status, or keyword search)
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
```

- [ ] **Step 2: Insert the real database ID**

Edit the file: replace `<insert-db-id-from-task-2>` with the actual Notion database ID recorded in Task 2 Step 1.

Verify the replacement: read the file back and confirm the ID is a valid UUID format (e.g., `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`).

- [ ] **Step 3: Commit**

```bash
git -C /Users/hunterheyman/Claude add docs/notion-brain-dump/claude-instructions.md
git -C /Users/hunterheyman/Claude commit -m "feat: add claude brain dump reference card with database ID"
```

---

### Task 5: Save Claude memory

**Files:**
- Create: `/Users/hunterheyman/.claude/projects/-Users-hunterheyman-Claude/memory/project_notion_brain_dump.md`
- Update: `/Users/hunterheyman/.claude/projects/-Users-hunterheyman-Claude/memory/MEMORY.md`

- [ ] **Step 1: Write the memory file**

Create `/Users/hunterheyman/.claude/projects/-Users-hunterheyman-Claude/memory/project_notion_brain_dump.md`:

```markdown
---
name: Notion Brain Dump System
description: Hunter uses a Notion database called "Brain Dump" to capture and organize ideas via Claude. Claude processes voice/text dumps and saves structured entries. Reference card at docs/notion-brain-dump/claude-instructions.md.
type: project
---

Hunter has a Notion "Brain Dump" database for capturing ideas, projects, tasks, and thoughts via Claude mobile (voice or text). Claude processes each dump and saves structured entries using a 4-level hierarchy (Thought → Idea → Task → Sub-task) with Area tags (Work, Personal, Compass Pro Bono).

**At the start of any brain dump session, read:** `docs/notion-brain-dump/claude-instructions.md`

This file contains the database ID, MCP tool names, capture workflow, routing rules, and all property specs needed to operate the system.
```

- [ ] **Step 2: Create MEMORY.md if it doesn't exist, then add pointer**

Check if `/Users/hunterheyman/.claude/projects/-Users-hunterheyman-Claude/memory/MEMORY.md` exists.

If it doesn't exist, create it with:
```markdown
# Memory Index

- [Notion Brain Dump System](project_notion_brain_dump.md) — Notion capture workflow; read docs/notion-brain-dump/claude-instructions.md at session start
```

If it exists, append the pointer line:
```markdown
- [Notion Brain Dump System](project_notion_brain_dump.md) — Notion capture workflow; read docs/notion-brain-dump/claude-instructions.md at session start
```

- [ ] **Step 3: Final commit**

```bash
git -C /Users/hunterheyman/Claude add docs/notion-brain-dump/claude-instructions.md
git -C /Users/hunterheyman/Claude commit -m "feat: complete notion brain dump system — reference card and memory saved"
```

---

## Verification Checklist

Before declaring done, confirm all of the following:

- [ ] `/Users/hunterheyman/Claude/notion-database/` folder exists
- [ ] Brain Dump database exists in Notion with all 7 properties (Title, Level, Area, Status, Summary, Details, Date Captured)
- [ ] Sub-items are enabled on the database
- [ ] All 6 views exist: All Entries, Work, Personal, Compass Pro Bono, Ready to Build, In Progress
- [ ] Database ID is recorded in `docs/notion-brain-dump/claude-instructions.md` as a valid UUID
- [ ] 4-level hierarchy (Thought → Idea → Task → Sub-task) was tested and visually confirmed in Notion UI
- [ ] Test entries were deleted — database is empty
- [ ] Claude reference card includes MCP tool names, capture workflow, routing rules, property specs, and view list
- [ ] Memory file exists at `/Users/hunterheyman/.claude/projects/-Users-hunterheyman-Claude/memory/project_notion_brain_dump.md`
- [ ] MEMORY.md has the pointer entry
- [ ] All commits are clean
