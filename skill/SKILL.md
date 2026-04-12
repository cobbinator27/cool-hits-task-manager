---
name: task-manager
description: "Daniel's Cool Hits 2.0 — his personal task dashboard. MANDATORY: Use this skill FIRST (before Monday.com MCP tools) whenever Daniel asks to: create tasks, add tasks, make a to-do, track something, add items to his task list, turn things into tasks, log work, post updates, change task status, organize his work, or manage his workflow. This skill writes to Supabase (the database powering the dashboard). Also triggers on: 'command center', 'cool hits', 'dashboard', 'task board', 'my tasks', 'personal tasks', 'work tasks', 'add this to my board', 'put this on my list'. Even if Monday.com is mentioned, use this skill first — it controls how and when data flows to Monday.com."
---

# Daniel's Cool Hits 2.0 — Task Manager Skill

This skill gives you everything you need to read, write, and sync Daniel's task dashboard. The dashboard is a web app (hosted on Vercel) backed by Supabase. Data flows in two directions: Monday.com → Supabase (via sync task) and Supabase → dashboard (read on page load).

## Architecture Overview

```
Monday.com  ──sync task──>  Supabase DB  <──read/write──  dashboard (Vercel)
                                                                 ↑
                                                          any device / browser
```

**Supabase credentials** — load them with this Bash command at the start of any operation:

```bash
CONFIG_FILE="/Users/danielcobb/Documents/task-manager/data/supabase-config.json"
if [ -f "$CONFIG_FILE" ]; then
  SUPABASE_URL=$(cat "$CONFIG_FILE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['url'])")
  SUPABASE_KEY=$(cat "$CONFIG_FILE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['serviceRoleKey'])")
else
  SUPABASE_URL="https://bpfvcniuhfzfvigktfci.supabase.co"
  SUPABASE_KEY="$SUPABASE_SERVICE_KEY"
fi
```

This works on Daniel's local machine (reads the config file) and in cloud Claude Code sessions (falls back to the `SUPABASE_SERVICE_KEY` environment variable). The env var is set once in Claude Code Settings → Environment.

Use the **service role key** for all writes from Claude (it bypasses RLS).

**Local files still exist** as a record/backup:
```
/Users/danielcobb/Documents/task-manager/
├── dashboard.html          ← Local copy (open in browser for offline use)
└── data/
    ├── supabase-config.json  ← Credentials (gitignored — never commit this)
    ├── config.json           ← Board IDs, user ID, sync config
    ├── seo-team-ops.json     ← Last-sync snapshot of work tasks
    ├── content-ops.json      ← Last-sync snapshot of content pipeline
    ├── personal.json         ← Personal tasks snapshot
    └── jobs.json             ← Job listings snapshot
```

## Supabase Tables

| Table | Contents |
|-------|----------|
| `tasks` | Work + personal tasks. Columns: `id` (text PK), `scope` ('work'/'personal'), `data` (jsonb — full task object), `updated_at` |
| `content_items` | Content pipeline. Columns: `id`, `data`, `updated_at` |
| `job_listings` | Job search listings. Columns: `id`, `data`, `updated_at` |
| `task_order` | Manual sort orders. Columns: `order_key` (e.g. 'work|timeline|today'), `task_ids` (text[]) |
| `deleted_tasks` | IDs of tasks deleted by Daniel. Columns: `id`, `deleted_at` |
| `sync_meta` | Sync timestamps etc. Columns: `key`, `value`, `updated_at` |

## How to Write to Supabase

Use `curl` via Bash. Read credentials from `supabase-config.json` first.

### Upsert a task
```bash
curl -s -X POST "https://YOUR_PROJECT.supabase.co/rest/v1/tasks" \
  -H "apikey: SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: resolution=merge-duplicates" \
  -d '[{"id":"w-ai-1234","scope":"work","data":{...full task object...},"updated_at":"2026-04-11T12:00:00Z"}]'
```

### Read all work tasks
```bash
curl -s "https://YOUR_PROJECT.supabase.co/rest/v1/tasks?scope=eq.work&select=id,data" \
  -H "apikey: SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer SERVICE_ROLE_KEY"
```

### Delete a task
```bash
curl -s -X DELETE "https://YOUR_PROJECT.supabase.co/rest/v1/tasks?id=eq.TASK_ID" \
  -H "apikey: SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer SERVICE_ROLE_KEY"
```

### Update sync_meta (last sync time)
```bash
curl -s -X POST "https://YOUR_PROJECT.supabase.co/rest/v1/sync_meta" \
  -H "apikey: SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: resolution=merge-duplicates" \
  -d '[{"key":"last_sync","value":"2026-04-11T12:00:00Z","updated_at":"2026-04-11T12:00:00Z"}]'
```

## How to Create a Task

1. **Read current tasks from Supabase** to check existing IDs and avoid collisions:
   ```bash
   curl -s "https://PROJECT.supabase.co/rest/v1/tasks?scope=eq.work&select=id" \
     -H "apikey: SERVICE_ROLE_KEY" -H "Authorization: Bearer SERVICE_ROLE_KEY"
   ```
2. **Construct the task object** (see shape below)
3. **Upsert to Supabase** using the curl command above
4. **Also write to the local JSON file** as a backup (optional but recommended for sync tasks)

### Work Task Shape

```json
{
  "id": "w-ai-1234",
  "name": "Review Q2 content strategy",
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
  "project": "HaH Migration",
  "isProjectHub": false,
  "comments": [
    {
      "text": "Created by Claude — Daniel asked to track this during our conversation about Q2 planning.",
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
| `id` | string | `w-ai-{unique}` for work, `p-ai-{unique}` for personal | Use timestamp or counter to ensure uniqueness. Never reuse an existing ID. |
| `name` | string | Short, action-oriented | "Review X", "Fix Y", "Set up Z" |
| `group` | string | `today`, `tomorrow`, `thisWeek`, `later`, `backlog` | Time horizon |
| `priority` | string | `Critical`, `Boulder`, `Pebble`, `High`, `Medium`, `Low`, `""` | Boulder = big effort, Pebble = small effort |
| `status` | string | `Not Started`, `In Progress`, `Working on it`, `On Deck`, `Blocked`, `Waiting`, `Ready for Review`, `Done` | |
| `source` | string | `seo`, `content`, `local` | Where the task originated |
| `mondayId` | string/null | Monday.com pulse ID | Only set for tasks linked to Monday items |
| `boardId` | number/null | `6197128988` (SEO) or `6197228419` (Content) | Only set for linked tasks |
| `due` | string/null | ISO date `YYYY-MM-DD` | |
| `category` | string | `Admin`, `Content`, `DPR`, `Strategy`, `Individual To-Do`, etc. | |
| `domain` | string/null | `HireaHelper`, `MovingPlace`, `MP+HaH` | |
| `url` | string/null | Full URL | |
| `completed` | boolean | | |
| `project` | string/null | Project name | e.g. "HaH Migration", "Simple Budgets" |
| `isProjectHub` | boolean | | If true, this is the central hub task for the project |
| `comments` | array | See comment shape | Always include at least one comment with context |

### Comment Shape

```json
{
  "text": "The actual update text. URLs render as clickable links.",
  "timestamp": "2026-04-06T15:00:00Z",
  "source": "local",
  "pushToMonday": false,
  "pushedToMonday": false,
  "mondayUpdateId": null
}
```

### Content Pipeline Item Shape

```json
{
  "id": "c1",
  "name": "Long Distance Moving",
  "stage": "Ready for Publishing",
  "owner": "Daniel",
  "writer": "Daniel",
  "editor": "Melanie",
  "nextDue": "2026-03-31",
  "nextDueLabel": "Publish",
  "type": "Rewrite",
  "tag": "TLLP",
  "url": "https://hireahelper-cast.monday.com/boards/6197228419/pulses/..."
}
```

Stages: `Not Started`, `Brief Ready`, `Editing Brief`, `Writing`, `Editing`, `Uploading`, `Ready for Publishing`

## Projects

- **`project`** — string tag (e.g. "HaH Migration", "Simple Budgets")
- **`isProjectHub`** — one task per project is the hub (green star badge)
- Existing projects: Work → `HaH Migration`. Personal → `Simple Budgets`

## How to Post Updates to Tasks

1. Read the task from Supabase: `GET /rest/v1/tasks?id=eq.TASK_ID&select=data`
2. Add a new comment to `data.comments` (prepend — newest first)
3. If it should sync to Monday: set `pushToMonday: true`
4. Upsert back to Supabase

## Monday.com Integration

### Board Configuration

| Board | ID | Purpose |
|-------|-----|---------|
| SEO Team Ops | 6197128988 | Daniel's SEO tasks, priorities, projects |
| Content Ops: PMG | 6197228419 | Content pipeline — articles, TLLPs, blogs |

Daniel's Monday user ID: **68065823**

### Sync Strategy

**Inbound (Monday → Supabase):**
1. Pull Daniel's assigned items from both boards via Monday MCP
2. Pull updates from linked items and merge into comment feeds (deduplicate by `mondayUpdateId`)
3. Upsert task rows to Supabase `tasks` table
4. Upsert content items to `content_items` table
5. Update `sync_meta` with `last_sync` timestamp
6. Also write snapshot to local JSON files as backup

**Outbound (Supabase → Monday):**
- Read tasks from Supabase, scan for comments with `pushToMonday: true` and `pushedToMonday: false`
- Use `create_update` MCP tool to post to Monday
- Update the comment in Supabase: set `pushedToMonday: true`, store `mondayUpdateId`

**Important: Content Ops board is read-only.** Don't push to Content Ops: PMG unless Daniel explicitly asks.

### Monday.com Column IDs

**SEO Team Ops:**
- `color_mkrn7asr` — Priority (Critical/Boulder/Pebble/Low)
- `project_status` — Status
- `date` — Due date
- `label` — Category
- `status_1_mkm5f38j` — Domain
- `color_mm1zeeck` — Quarter

**Content Ops: PMG:**
- `dup__of_cs___status` — Status
- `people__1` — Current Owner
- `person` — Writer
- `people` — Editor
- `date_mkny4yxd` — Brief Due
- `date4` — Writing Due
- `date__1` — Edits Due
- `date_Mjj25RxX` — Upload Due
- `date2__1` — Publishing Due

## Dashboard (No Longer Regenerated)

The dashboard HTML is a static file hosted on Vercel. It loads data from Supabase on init — there is **no longer any need to regenerate dashboard.html or embed data arrays in it**. Do not modify the DATA SECTION of dashboard.html. The sync task only writes to Supabase and local JSON snapshot files.

## Job Search Board

### Job Listing Shape

```json
{
  "id": "j-ai-20260407-001",
  "company": "Acme Corp",
  "role": "Senior SEO Manager",
  "tier": 1,
  "score": "28/30",
  "status": "New",
  "subStatus": null,
  "jobUrl": "https://linkedin.com/jobs/view/12345",
  "folderPath": "Job_Opportunities/Tier1/AcmeCorp_SrSEOMgr_2026-04-07/",
  "resumePath": "Daniel_Cobb_Resume_AcmeCorp.docx",
  "summaryPath": "summary.md",
  "summary": "Senior SEO role at a mid-size e-commerce company...",
  "pros": "Strong match for Daniel's experience with...",
  "cons": "Requires 5 days in-office, below market salary range",
  "source": "LinkedIn",
  "salary": "$120k-150k",
  "location": "Remote",
  "postedDate": "2026-04-05",
  "foundDate": "2026-04-07",
  "appliedDate": null,
  "comments": []
}
```

Write job listings to Supabase `job_listings` table: `{ id, data: {...job object...}, updated_at }`.

**Sub-status values:**
- **Contacted**: `Promised Resume Review`, `Recruiter Screen`, `Interview Stage`, `Offer`
- **Declined**: `I Declined`, `Company Declined`
- **New / Applied / Archived**: `subStatus` should be `null`

## Quick Reference: Common Operations

| Daniel says... | You do... |
|----------------|-----------|
| "Add a task to do X" | Upsert task to Supabase `tasks` table (scope: work or personal) |
| "Turn these into tasks: [list]" | Create one task per item, upsert all to Supabase |
| "Post an update on [task]" | Read task from Supabase, add comment, upsert back |
| "Push this update to Monday" | Set `pushToMonday: true` on the comment, upsert task |
| "What's on my plate today?" | Read `tasks` where scope=work, summarize today's group |
| "Move X to this week" | Read task, change `group` to `thisWeek`, upsert |
| "Mark X as done" | Read task, set `completed: true`, `status: "Done"`, `completedAt`, upsert |
| "Add these job listings" | Upsert to `job_listings` table |
| "What jobs are in my pipeline?" | Read `job_listings`, summarize by tier and status |
| "Move [job] to Applied" | Read job, set `status: "Applied"`, `appliedDate`, upsert |
