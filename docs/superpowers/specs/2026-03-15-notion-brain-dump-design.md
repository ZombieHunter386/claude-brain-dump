# Notion Brain Dump System — Design Spec
**Date:** 2026-03-15
**Status:** Approved

---

## Overview

A Notion-based capture and organization system for ideas, projects, tasks, and thoughts. The user brain dumps via voice or text in the Claude mobile app. Claude processes the input, structures it into a 4-level hierarchy, and saves it to Notion. Entries can later be retrieved by Claude for implementation or browsed directly in Notion.

---

## Goals

- Never lose an idea, thought, project concept, or task
- Organize entries hierarchically so the "shape" of a big idea is always visible
- Make entries actionable — structured enough that Claude can pick them up later and implement without re-explanation
- Keep capture frictionless (voice or text, no formatting required from user)

---

## Hierarchy

All entries live in a single Notion database with 4 levels of nesting:

```
🧠 Thought     — the big vision or overarching concept
   💡 Idea     — specific directions that stem from the thought
      ✅ Task   — actionable work to implement the idea
         ↳ Sub-task — smaller steps that make up the task
```

**Examples:**

```
🧠 Thought: "Improve Compass Pro Bono intake process"
   💡 Idea: "Build a mobile intake form"
      ✅ Task: "Design the form flow"
         ↳ Sub-task: "Research existing tools"
         ↳ Sub-task: "Sketch 3 layout options"
      ✅ Task: "Set up backend to store submissions"
   💡 Idea: "Auto-email confirmation to clients"
      ✅ Task: "Write email templates"
```

---

## Notion Database

**Name:** Brain Dump

**Sub-items:** Enabled (allows parent-child row nesting for tree view)

### Properties

| Property | Type | Applies To | Notes |
|---|---|---|---|
| Title | Text | All levels | Claude-generated 1-line summary |
| Level | Select | All levels | Thought / Idea / Task / Sub-task |
| Area | Select | All levels | Work / Personal / Compass Pro Bono — set on Thought, manually copied to children so views filter correctly |
| Status | Select | All levels | Captured / Ready to Build / In Progress / Done |
| Summary | Text | All levels | 2-3 sentence recap written by Claude |
| Details | Text | All levels | 3-5 bullet points: intent, constraints, specific requirements mentioned |
| Date Captured | Date | All levels | Auto-set on creation |

### Views

| View Name | Filter | Notes |
|---|---|---|
| All Entries | None | Tree view, collapsed by default |
| Work | Area = Work | Shows all levels tagged Work |
| Personal | Area = Personal | Shows all levels tagged Personal |
| Compass Pro Bono | Area = Compass Pro Bono | Shows all levels tagged Compass Pro Bono |
| Ready to Build | Status = Ready to Build | Across all Areas |
| In Progress | Status = In Progress | Across all Areas |

**Note on nesting in Notion:** Sub-items are implemented via Notion's native sub-item feature (child rows within the same database, not child pages). The Notion MCP API creates child entries by setting the `parent` field to the parent row's page ID. Area is set explicitly on every row (not inherited) so filtered views show the full entry set correctly.

---

## Capture Workflow

**Input:** User sends voice or text message to Claude via Claude mobile app.

**Claude processing steps:**

1. **Parse** the brain dump — extract meaning, intent, and any sub-components mentioned. If no classifiable content can be extracted (e.g., "I'll think about this later"), inform the user and ask them to rephrase — do not create a blank entry.
2. **Classify** each component by Level (Thought / Idea / Task / Sub-task) and Area (Work / Personal / Compass Pro Bono). If a single dump spans multiple Areas (e.g., "fix the Compass intake flow and redesign my personal site"), split it into separate Thoughts, one per Area, and confirm both with the user.
3. **Scan existing Thoughts** in Notion for semantic overlap — check for keyword or concept matches in existing Thought titles and summaries.
4. **Route using this decision rule:**
   - **High confidence match** (clear keyword or concept overlap with an existing Thought) → attach at the appropriate level and confirm afterward: "Added *X* as an Idea under *Y*"
   - **Ambiguous match** (two or more plausible parent Thoughts) → present options to the user before writing: "This could go under *Y* or *Z* — which fits better?"
   - **No match** → create a new top-level Thought (with Ideas/Tasks nested below if enough detail was provided)
   - **Merge opportunity** (new Thought title/content significantly overlaps an existing standalone Thought) → proactively offer a merge before writing: "This seems related to *[existing Thought]* — want me to combine them?"
5. **Write to Notion** via MCP integration. Set Area on every row created (Thought and all children).
6. **Confirm to user:** e.g., "Added *Design intake form* as an Idea under *Compass Pro Bono intake process*" or "Created new Thought: *[Title]*"

---

## Retrieval Workflow

**Via Claude (conversational):**
- "Show me all my Compass Pro Bono ideas" → Claude queries Notion, returns tree view of matching entries
- "What tasks are ready to build?" → Claude filters by Status = Ready to Build
- "Pull my [project name] so we can build it" → Claude fetches the full hierarchy for that Thought, presents it, and is ready to begin implementation

**Via Notion directly:**
- User browses using saved views (Work, Personal, Compass Pro Bono, etc.)
- Tree structure is visible inline — expand any Thought to see all nested Ideas, Tasks, Sub-tasks

---

## Claude Behavior Guidelines

- **Always confirm** what was saved and where after each brain dump
- **Never silently drop** any part of a brain dump — if something is ambiguous, create an entry and flag it
- **Suggest merges** conversationally, don't auto-merge without user awareness
- **Write Details field as 3-5 bullet points** covering: intent (what and why), constraints, and any specific requirements mentioned — enough for future-Claude to implement without re-explanation
- **Status defaults to "Captured"** on creation
- **Status is only changed by explicit user instruction** or when the user says to begin building (e.g., "let's build X" upgrades it to "In Progress"). Claude never auto-advances status without being told to.

---

## Out of Scope (v1)

- Calendar or deadline integration
- Automatic reminders or notifications
- Integration with task managers (Linear, Jira, etc.)
- Multi-user / team access
- Automated implementation triggers (user always initiates)

---

## Success Criteria

- User can brain dump in under 30 seconds with no formatting effort
- Every entry is findable in Notion within 2 clicks or via a single Claude query
- When a user says "build [X]", Claude can retrieve the full Thought hierarchy from Notion and produce an implementation plan or first code artifact without the user providing additional context beyond the project name
- The hierarchy is visible and navigable in Notion's tree view
