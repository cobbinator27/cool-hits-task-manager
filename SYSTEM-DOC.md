# Cool Hits 2.0 — System Documentation

*Last updated: April 10, 2026*

This is the master reference doc for Daniel's task management system. It covers what exists, how it works, what's broken, what's planned, and how to maintain it. Keep this updated as the system evolves.

---

## What This Is

A personal task dashboard that replaces the need to live inside Monday.com all day. It's a single self-contained HTML file (~4,000 lines) with inline CSS and JS, opened locally in the browser as a `file://` URL. Data flows in from Monday.com via scheduled Claude sync tasks, and the browser persists local changes via localStorage.

The system has three layers: the dashboard HTML (the UI), JSON data files (Claude's read/write layer), and Monday.com (the upstream source of truth for work tasks).

---

## Architecture

### File Layout

```
/Users/danielcobb/Documents/task-manager/
├── dashboard.html              <- The dashboard (open in browser)
├── SYSTEM-DOC.md               <- This file
├── scope-multi-device-task-manager.md  <- Future mobile/multi-device scope
├── skill/
│   └── SKILL.md                <- Claude-facing skill (instructions for all Cowork sessions)
│   └── references/             <- Supporting files for the skill
├── data/
│   ├── config.json             <- Monday board IDs, user ID, sync config
│   ├── seo-team-ops.json       <- Work tasks from SEO Team Ops board
│   ├── content-ops.json        <- Content pipeline from Content Ops: PMG board
│   ├── personal.json           <- Personal tasks (manual + AI-created)
│   ├── jobs.json               <- Job search listings (AI scan output)
│   ├── briefs.json             <- Brief automation pipeline data (read-only for now)
│   └── backups/                <- Auto-created by sync task before each write
├── starter-kit/
│   ├── dashboard.html          <- Clean starter version (for sharing with colleagues)
│   └── SETUP-GUIDE.md          <- Setup instructions for the starter kit
└── task-manager.plugin         <- Cowork plugin package (v0.6.0)
```

### Persistence Model

Two layers, merged on page load:

1. **Embedded data arrays** in `dashboard.html` — Claude writes these during syncs. They're the "fresh load" baseline. Arrays: `WORK_TASKS`, `PERSONAL_TASKS`, `CONTENT_PIPELINE`, `JOB_LISTINGS`, `BRIEF_ITEMS`.

2. **localStorage** in the browser — Every time you interact with the dashboard (drag a task, change a status, add a comment), it saves to localStorage immediately. On page load, localStorage wins for existing tasks (by ID), and any NEW tasks Claude embedded since last load get appended.

This means: Claude can safely add tasks without stomping your local changes. But if you clear browser data, you lose any edits that weren't synced back to the JSON files.

### Data Flow

```
Monday.com  ──sync task──>  JSON files  ──embedded in──>  dashboard.html
                                                              │
                                                         localStorage
                                                         (browser-side
                                                          edits)
```

Outbound: Comments flagged `pushToMonday: true` get posted to Monday by the sync task.

---

## Dashboard Tabs

### Work Tab
- **My Tasks** — Board view (Today/Tomorrow/This Week/Later/Backlog columns with drag-and-drop) or List view. Groupable by timeline, priority, source, status, or project.
- **Content Pipeline** — Kanban by stage (Not Started → Brief Ready → Writing → Editing → Uploading → Ready for Publishing). Read-only from Content Ops: PMG board.

### Personal Tab
- **My Tasks** — Same layout as Work, independent task list.
- **Job Search** — Hybrid 2×3 grid (Tier 1/Tier 2/Contacted on top, Tier 3/Declined/Archived on bottom) plus a pipeline kanban view. Tracks LinkedIn opportunities from the AI scan.

### Briefs Tab
- Read-only status monitor for the content brief automation pipeline. 13-step status system. Decision made to use Monday.com as the primary UI for briefs, with this tab as a diagnostic/spot-check view. Batch operations and editing are built but will be deprecated in favor of Monday-first approach.

### Shared Features (all tabs)
- Dark/light mode (persisted to localStorage)
- Auto-reload every 60 seconds (view state persisted so you don't lose your place)
- Quick-add bar with bucket, priority, and project dropdowns
- Editable task names, statuses, priorities, due dates, projects in detail panels
- Comment system with Monday.com push support
- Project tagging with hub tasks, filtering, and grouping

---

## Monday.com Integration

### Boards

| Board | ID | Purpose |
|-------|-----|---------|
| SEO Team Ops | `6197128988` | Daniel's SEO work tasks |
| Content Ops: PMG | `6197228419` | Content pipeline (read-only) |

**Daniel's Monday user ID:** `68065823`

### Column Mappings

**SEO Team Ops:**
- Priority: `color_mkrn7asr` — Critical (10), Boulder (110), Pebble (109), Low (7)
- Status: `project_status` — Not Started (5), On Deck (2), In Progress (0), Waiting/Blocked (3), At Risk (4), Done (1), Cancelled (6)
- Due: `date`
- Category: `label`
- Domain: `status_1_mkm5f38j`
- Quarter: `color_mm1zeeck`

**Content Ops: PMG:**
- Status: `dup__of_cs___status`
- Owner: `people__1`
- Writer: `person`
- Editor: `people`
- Dates: `date_mkny4yxd` (Brief), `date4` (Writing), `date__1` (Edits), `date_Mjj25RxX` (Upload), `date2__1` (Publishing)

---

## Scheduled Tasks

### sync-task-dashboard
- **Schedule:** 8 AM, 10 AM, 1 PM, 4 PM on weekdays
- **Does:** Pulls tasks from both Monday boards, updates JSON data files, regenerates the data arrays in dashboard.html
- **Key rules:** Backs up before writing, only replaces data arrays (not full HTML), validates JS syntax after writing, restores from backup on failure
- **Known issue history:** Previously overwrote the full HTML file, wiping structural changes (new tabs, UI modifications). Fixed April 2026 — prompt now explicitly says "ONLY replace data arrays." Also previously dropped commas between array items before comment lines, causing blank dashboard. Formatting rules now in both the task prompt and SKILL.md.

### linkedin-job-scan-morning
- **Schedule:** 6:05 AM daily
- **Does:** Scans LinkedIn for content/SEO leadership roles, scores and tiers them, creates tailored resumes, emails Daniel for Tier 1 opportunities
- **Known issue:** Couldn't access `data/jobs.json` because the scheduled task session didn't have the folder mounted. Fix: task needs to request access to `/Users/danielcobb/Documents/task-manager` at startup. Do a "Run now" and approve with "Always allow" to persist permissions.

### linkedin-job-scan-evening
- **Schedule:** 6:06 PM daily
- **Does:** Same as morning scan, catches afternoon/evening postings
- **Same folder access issue as morning scan**

### brief-orchestrator
- **Schedule:** 11:04 PM daily (currently **disabled**)
- **Does:** Reads feeder spreadsheet, finds briefs marked "Ready for AI Brief", spawns brief runners
- **Status:** On hold pending decision to move brief management to Monday-first approach

---

## Known Issues & Watchouts

### The Sync Task Can Break the Dashboard
**Severity: High** | **Status: Mitigated but monitor**

The sync task regenerates data arrays in dashboard.html. If it drops commas, uses wrong syntax, or rewrites the full file, the dashboard goes blank. Mitigations in place: backup before write, syntax validation, explicit formatting rules in prompt, "only touch data arrays" instruction. But Claude sessions are non-deterministic — a future sync could still deviate. Always check the dashboard after the first sync run of the day.

**Recovery:** Go to `data/backups/`, find the most recent file, copy it over `dashboard.html`.

### localStorage Is Per-Browser, Per-Device
**Severity: Medium** | **Status: Accepted for now**

If you open the dashboard in a different browser or clear data, you lose any edits that only existed in localStorage (tasks added via quick-add, drag-and-drop moves, status changes). The sync task doesn't capture browser-side edits back to JSON.

**Workaround:** For important tasks, ask Claude to add them (they go into the JSON and survive across browsers). Quick-add tasks are ephemeral until the next sync embeds them.

### Job Search Resume Links Don't Work
**Severity: Low** | **Status: Open**

The dashboard shows `file://` links for resumes and summaries, but browsers block `file://` URLs for security. The intended fix was to upload to Google Drive and link there, but the Google Drive MCP only supports search/read, not upload. Deferred until multi-device migration (where files would be in cloud storage anyway).

### Scheduled Tasks Need Folder Access
**Severity: Medium** | **Status: Partially fixed**

Scheduled tasks run in isolated sessions and need explicit folder access. The fix is to request the directory in the task prompt and do a "Run now" with "Always allow" to persist permissions. The sync task has been updated but the job scan tasks still need the same treatment.

### Briefs Tab Architecture Decision
**Severity: Info** | **Status: Decided**

Built a full briefs management UI (13-step pipeline, batch operations, import, filtering), then decided Monday.com should be the primary UI for briefs. The tab remains as a read-only status monitor. If the Monday-first approach works well, the editing features can be removed. If it doesn't work, the infrastructure is there to reactivate.

---

## Active Projects

### Work Projects
- **HaH Migration** — HireAHelper site migration tasks

### Personal Projects
- **Simple Budgets** — Personal budgeting app (bugs and improvements tracked here)

---

## Version History

| Date | Version | Changes |
|------|---------|---------|
| Apr 6, 2026 | v0.1 | Initial dashboard with Work + Personal tabs, Monday sync |
| Apr 7, 2026 | v0.2 | Added project system (tagging, hubs, filtering, grouping) |
| Apr 7, 2026 | v0.3 | Fixed duplicate task IDs, changed quick-add default to This Week |
| Apr 7, 2026 | v0.4 | Added Job Search board under Personal (hybrid grid + pipeline views) |
| Apr 8, 2026 | v0.5 | Job field updates (pros/cons, score, jobUrl, folderPath, summaryPath) |
| Apr 8, 2026 | v0.5.1 | Removed hard page refresh, added view state persistence |
| Apr 8, 2026 | v0.5.2 | Added 60-second auto-reload with view state persistence |
| Apr 9, 2026 | v0.6 | Added Briefs tab (13-step pipeline, batch ops, import, filters) |
| Apr 9, 2026 | v0.6.1 | Fixed briefs tab always visible (CSS `!important` override) |
| Apr 10, 2026 | v0.7 | Fixed sync task breaking dashboard (comma issues), added backup + validation to sync task, added formatting rules to SKILL.md |
| Apr 10, 2026 | v0.7.1 | Added project dropdown and Tomorrow to quick-add, editable task names |
| Apr 10, 2026 | v0.7.2 | Updated sync task to only replace data arrays (not full file rewrite) |
| Apr 10, 2026 | — | Created starter kit for sharing, wrote this system doc |

### Plugin Versions
- **v0.6.0** — Current. Updated job fields, project system, job search board docs.

---

## Roadmap / Future Ideas

### Near-term
- [ ] Fix scheduled task folder access (do "Run now" + "Always allow" on job scan tasks)
- [ ] Monitor sync task behavior after the "only replace data arrays" fix
- [ ] Decide on brief pipeline: fully commit to Monday-first or keep dashboard UI
- [ ] Upload resumes/summaries to Google Drive (blocked on Drive upload MCP)

### Medium-term
- [ ] Multi-device access — see `scope-multi-device-task-manager.md` for full analysis
  - Phase 1: Host dashboard on GitHub Pages + JSON in GitHub repo (any device can view)
  - Phase 2: PWA + Supabase for full mobile app with quick-add
- [ ] Mobile quick-add (Telegram bot, Apple Shortcut, or PWA)
- [ ] Capture browser-side edits back to JSON (close the localStorage gap)
- [ ] Auto-archive jobs with no activity after 4 weeks
- [ ] Notification system (upcoming due dates, gated items ready to run)

### Someday/Maybe
- [ ] Slack/email integration for task creation ("email a special inbox to create a task")
- [ ] Auto-push status changes to Monday (currently only comments push)
- [ ] AI-powered daily standup summary ("here's what changed since yesterday")
- [ ] Shared dashboard for team visibility (read-only view for colleagues)

---

## How to Maintain This Doc

Update this file whenever:
- A new tab or major feature is added
- A scheduled task is created or modified
- A known issue is discovered or resolved
- A roadmap item is started or completed
- The architecture changes (new data files, new integrations, etc.)

The SKILL.md is for Claude — it tells Claude how to interact with the system. This doc is for Daniel — it tells you how the system works, what's broken, and where it's going.
