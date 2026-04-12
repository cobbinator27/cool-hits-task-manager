# Cool Hits 2.0 — Setup & Configuration Guide

A personal task dashboard that runs as a single HTML file in your browser, powered by Claude (Cowork) for automation, and synced with Monday.com.

---

## How It Works

The dashboard is a self-contained HTML file you open locally in your browser. It has two persistence layers:

1. **Embedded data** — Task arrays inside the `<script>` tag, updated by Claude during syncs
2. **localStorage** — Your browser saves changes (drag-and-drop, status updates, new tasks) locally

On page load, the dashboard merges these: localStorage is the source of truth for existing tasks, but any NEW tasks Claude added to the embedded data get appended. This means Claude can safely add tasks without overwriting your local changes.

The dashboard auto-reloads every 60 seconds to pick up the latest data. Your active tab and view state are saved to localStorage so you stay right where you were.

---

## Quick Start

### 1. Set Up Your Folder

Create this folder structure on your machine:

```
~/Documents/task-manager/
  dashboard.html          <- The dashboard file (included)
  data/
    config.json           <- Your Monday.com board config
    work-board.json       <- Tasks from your primary Monday board
    personal.json         <- Personal tasks (manually managed)
    backups/              <- Auto-created by sync task
```

### 2. Configure Monday.com

Create `data/config.json`:

```json
{
  "mondayUserId": "YOUR_MONDAY_USER_ID",
  "boards": {
    "primary": {
      "id": YOUR_BOARD_ID,
      "name": "Your Board Name"
    }
  },
  "lastFullSync": null
}
```

**Finding your Monday user ID:** Ask Claude: "What's my Monday.com user ID?" — Claude can look it up via the Monday MCP.

**Finding your board ID:** Go to your Monday board in the browser. The URL looks like `https://your-org.monday.com/boards/1234567890` — that number is the board ID.

### 3. Map Your Monday Columns

You need to tell Claude which Monday columns map to which dashboard fields. Open your board and identify the column IDs for:

| Dashboard Field | What to Look For | How to Find the Column ID |
|----------------|-------------------|--------------------------|
| Priority | Your priority/importance column | Ask Claude: "What are the column IDs on board {ID}?" |
| Status | Your workflow status column | Same — Claude can list all columns and their IDs |
| Due Date | Any date column for deadlines | |
| Category | A label/tag column (optional) | |
| Domain/Brand | If you track which product/site (optional) | |

Once you have the IDs, add them to your config:

```json
{
  "mondayUserId": "12345678",
  "boards": {
    "primary": {
      "id": 1234567890,
      "name": "My Team Board",
      "columns": {
        "priority": "color_abc123",
        "status": "status_column",
        "due": "date",
        "category": "label",
        "domain": "status_xyz"
      }
    }
  },
  "columnMappings": {
    "priorityLabels": {
      "Critical": "id_for_critical",
      "High": "id_for_high",
      "Medium": "id_for_medium",
      "Low": "id_for_low"
    },
    "statusLabels": {
      "Not Started": "id_for_not_started",
      "In Progress": "id_for_in_progress",
      "Done": "id_for_done"
    }
  }
}
```

**Pro tip:** Just ask Claude to do this for you. Say: "Look at my Monday board {ID} and map all the columns for my task dashboard config." Claude can read the board schema and build the config automatically.

### 4. Set Up the Claude Skill

Create a skill file so Claude knows how to interact with your dashboard. Place it at:

```
~/Documents/task-manager/skill/SKILL.md
```

A complete template is provided at the end of this guide (see "Skill Template" section). Customize it with your specific board IDs, column mappings, and preferences.

### 5. Set Up the Sync Task

Create a scheduled task in Cowork that pulls your Monday data and updates the dashboard. The task should:

1. Back up `dashboard.html` before making changes
2. Pull your tasks from Monday.com via the MCP
3. Update the JSON data files
4. Replace ONLY the data arrays in dashboard.html (not the full file)
5. Validate JS syntax before saving
6. Restore from backup if validation fails

**To create the task:** Tell Claude: "Create a scheduled task that syncs my Monday board to my task dashboard. Run it every few hours on weekdays."

Then do a **"Run now"** to test it and approve any permission prompts with "Always allow" so future runs are fully autonomous.

---

## Dashboard Features

### Tabs

- **Work** — Tasks from your Monday board + locally created tasks
- **Personal** — Personal to-dos, side projects, etc. (not synced to Monday)

You can add more sub-tabs under either tab (e.g., a content pipeline kanban, a job tracker, a reading list). The existing code has patterns for sub-tab navigation — ask Claude to add one when you're ready.

### Views

- **Board view** — Kanban-style columns by time horizon (Today, Tomorrow, This Week, Later, Backlog). Drag and drop tasks between columns.
- **List view** — Compact grouped table. Click any row to expand the detail panel.

### Grouping

Group tasks by: Timeline (default), Priority, Source, Status, or Project. The grouping dropdown is in the view controls bar.

### Quick Add

The bar at the top of each tab lets you quickly create a task with a name, time bucket, priority, and optional project tag. Press Enter or click "Add Task."

### Detail Panel

Click any task card or list row to expand the detail panel where you can edit:
- Task name
- Status (Not Started, In Progress, Working on it, On Deck, Blocked, Waiting, Ready for Review, Done)
- Priority (Critical, Boulder, Pebble, High, Medium, Low)
- Time bucket (Today → Backlog)
- Due date
- Project tag (with autocomplete from existing projects)
- Hub designation (marks a task as the central "hub" for a project)
- Comments/updates (with optional push-to-Monday flag)

### Projects

Projects are a tagging system for grouping related tasks:

- Tag any task with a project name (e.g., "Website Redesign")
- One task per project can be the **hub** — the central task representing the project itself. Hub tasks get a green star badge and border.
- Filter by project using the dropdown in the view controls
- Group by project to see all related tasks together
- Projects work across both Work and Personal tabs independently

### Dark/Light Mode

Toggle in the top-right corner. Preference is saved to localStorage.

---

## Data Model

### Task Shape

```json
{
  "id": "w-ai-1234",
  "name": "Review Q2 strategy",
  "group": "thisWeek",
  "priority": "Boulder",
  "status": "Not Started",
  "source": "local",
  "mondayId": null,
  "boardId": null,
  "due": "2026-04-15",
  "category": "Strategy",
  "domain": null,
  "url": null,
  "completed": false,
  "project": "Q2 Planning",
  "isProjectHub": false,
  "comments": [
    {
      "text": "Created by Claude from our planning conversation.",
      "timestamp": "2026-04-06T15:00:00Z",
      "source": "local",
      "pushToMonday": false,
      "pushedToMonday": false,
      "mondayUpdateId": null
    }
  ]
}
```

### Field Reference

| Field | Type | Values | Notes |
|-------|------|--------|-------|
| `id` | string | `w-ai-{unique}` (work), `p-ai-{unique}` (personal) | Use timestamp or counter. Never reuse. |
| `name` | string | Short, action-oriented | "Review X", "Fix Y", "Set up Z" |
| `group` | string | `today`, `tomorrow`, `thisWeek`, `later`, `backlog` | Time horizon |
| `priority` | string | `Critical`, `Boulder`, `Pebble`, `High`, `Medium`, `Low`, `""` | Boulder = big effort, Pebble = small |
| `status` | string | `Not Started`, `In Progress`, `Working on it`, `On Deck`, `Blocked`, `Waiting`, `Ready for Review`, `Done` | |
| `source` | string | Your board name, `local` | Where the task came from |
| `mondayId` | string/null | Monday.com pulse ID | Only for linked tasks |
| `boardId` | number/null | Monday.com board ID | Only for linked tasks |
| `due` | string/null | `YYYY-MM-DD` | |
| `category` | string | Flexible | Whatever categories work for you |
| `domain` | string/null | Brand/product name | Optional — for multi-brand teams |
| `url` | string/null | Full URL | Monday link or any relevant URL |
| `completed` | boolean | | |
| `project` | string/null | Project name | Same string across related tasks |
| `isProjectHub` | boolean | | One hub per project |
| `comments` | array | See comment shape | Always include context on creation |

### Comment Shape

```json
{
  "text": "The update text. URLs render as clickable links.",
  "timestamp": "2026-04-06T15:00:00Z",
  "source": "local",
  "pushToMonday": false,
  "pushedToMonday": false,
  "mondayUpdateId": null
}
```

| Field | Notes |
|-------|-------|
| `source` | `"local"` = created in dashboard/by Claude. `"monday"` = pulled from Monday |
| `pushToMonday` | If true, the sync task pushes this update to the linked Monday item |
| `pushedToMonday` | Set to true after successful push |
| `mondayUpdateId` | Monday's update ID for dedup |

---

## Monday.com Integration

### Sync Strategy

**Inbound (Monday -> Dashboard):**
- Pull your assigned items from your board(s)
- Map Monday columns to dashboard fields
- Write to JSON data files
- Update the embedded data arrays in dashboard.html
- Deduplicate by mondayId

**Outbound (Dashboard -> Monday):**
- Scan for comments with `pushToMonday: true` and `pushedToMonday: false`
- Use Monday MCP to post them to the linked item
- Mark as pushed and store the mondayId

**Status changes are NOT auto-pushed** — only explicit update posts. This prevents accidental overwrites when you're managing status differently on the dashboard vs. Monday.

### Adding More Boards

To pull from multiple Monday boards, add them to your config and update the sync task prompt. Each board gets its own JSON data file and its own `source` label so you can filter by source in the dashboard.

---

## Extending the Dashboard

### Adding Sub-Tabs

The dashboard supports sub-tabs under each main tab. Common additions:

- **Content Pipeline** — A kanban board tracking articles/content through stages (Brief -> Writing -> Editing -> Publishing)
- **Job Tracker** — A board for tracking job applications by status
- **Reading List** — A simple list of articles/resources to read
- **Project Board** — A dedicated view for a single large project

Ask Claude to add one: "Add a [name] sub-tab under my Work tab with a kanban view." Claude knows the patterns from the existing code.

### Adding Fields

To add custom fields to task cards:
1. Add the field to the task data shape
2. Add rendering in the card/list-row functions
3. Add an edit control in the detail panel
4. Update the skill doc so Claude knows about the new field

### Adding Filters

The view controls bar can be extended with additional filter dropdowns (by category, domain, due date range, etc.). The existing project filter is a good pattern to copy.

---

## Important: Dashboard Regeneration Rules

When Claude (or a sync task) updates the dashboard HTML, these rules MUST be followed to avoid breaking the page:

1. **ONLY replace the data arrays** — never rewrite the full HTML file
2. **Every array item MUST end with a trailing comma** (including before comments and at the end)
3. **Use `var`** for data declarations (not `const` or `let`)
4. **Validate syntax after every write:**
   ```bash
   node -e "const html=require('fs').readFileSync('dashboard.html','utf8'); const s=html.match(/<script>([\s\S]*?)<\/script>/)[1]; new Function(s); console.log('OK')"
   ```
5. **Back up before writing** — copy to `data/backups/` with a timestamp
6. **Restore from backup if validation fails** — never leave a broken file

These rules should be in both the sync task prompt AND the skill doc.

---

## Skill Template

Copy this to `skill/SKILL.md` and customize the bracketed values:

````markdown
---
name: task-manager
description: "[Your Name]'s Cool Hits 2.0 — personal task dashboard. Use this skill whenever asked to: create tasks, add tasks, track something, change task status, organize work, or manage workflow. This skill writes to local JSON data files that feed the dashboard. Triggers on: 'dashboard', 'cool hits', 'task board', 'my tasks', 'add this to my board'."
---

# Task Manager Skill

> **IMPORTANT: The dashboard and data files already exist. Read existing files first, then append/modify.**

## Architecture

```
[YOUR_PATH]/task-manager/
  dashboard.html          <- The dashboard
  data/
    config.json           <- Board IDs, user ID, sync settings
    work-board.json       <- Tasks from Monday board
    personal.json         <- Personal tasks
    backups/              <- Auto-created backups
```

## How to Create a Task

Write to the appropriate JSON data file. Follow the task shape documented in the setup guide.

**ID convention:** `w-ai-{unique}` for work, `p-ai-{unique}` for personal.

**Defaults for quick creation:**
- Group: `today` if urgent, `thisWeek` if not specified
- Priority: empty
- Status: `Not Started`
- Source: `local`
- Always add a comment explaining context

**Writing good tasks:**
- `name` is the title — keep it short, action-oriented
- `comments` array is where detail goes — context, links, reproduction steps
- Don't dump raw descriptions into the title

## Monday.com Integration

**Board:** [YOUR_BOARD_NAME] (ID: [YOUR_BOARD_ID])
**User ID:** [YOUR_MONDAY_USER_ID]

**Column mappings:**
- Priority: `[COLUMN_ID]`
- Status: `[COLUMN_ID]`
- Due date: `[COLUMN_ID]`
- Category: `[COLUMN_ID]`

## Regenerating the Dashboard

1. Back up dashboard.html to data/backups/
2. Read the current dashboard.html
3. Read JSON data files
4. Replace ONLY the WORK_TASKS and PERSONAL_TASKS arrays
5. Update LAST_SYNC timestamp
6. Validate JS syntax before writing
7. Write the file back

**CRITICAL:** Every array item needs a trailing comma. Use `var`. Only touch data arrays. Validate after writing. Restore from backup on failure.

## Common Operations

| User says... | You do... |
|--------------|-----------|
| "Add a task" | Create task in JSON, add context comment |
| "Post an update on [task]" | Add comment to task's comments array |
| "Push this to Monday" | Set pushToMonday: true on the comment |
| "What's on my plate?" | Read data, summarize today's tasks |
| "Move X to this week" | Change task's group to thisWeek |
| "Mark X as done" | Set completed: true, status: Done |
| "Tag under [project]" | Set project field on the task |
| "Create a project for X" | Create hub task + tag related tasks |
````

---

## Troubleshooting

**Dashboard is blank:**
The sync task probably broke the JS. Check the browser console (F12) for errors. Restore from `data/backups/` — the most recent backup should work.

**Tasks disappear after sync:**
The sync task may be overwriting the full file instead of just the data arrays. Check the sync task prompt — it should explicitly say "ONLY replace the data arrays."

**Duplicate tasks after sync:**
The dedup function handles this on load, but check that IDs aren't being reused. Monday-sourced tasks should use the mondayId as part of their dashboard ID.

**localStorage conflicts:**
If you clear your browser data or switch browsers, you'll see only the embedded data. Any tasks created locally in the browser that weren't synced back to JSON will be lost. The sync task should capture locally-created tasks and add them to the JSON files.
