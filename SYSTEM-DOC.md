# Cool Hits 2.0 — System Documentation

*Last updated: April 12, 2026*

This is the master reference doc for Daniel's task management system. It covers what exists, how it works, what's broken, what's planned, and how to maintain it. Keep this updated as the system evolves.

---

## What This Is

A personal task dashboard that replaces the need to live inside Monday.com all day. It's a single self-contained HTML file (~4,000 lines) with inline CSS and JS, opened locally in the browser as a `file://` URL. Data flows in from Monday.com via scheduled Claude sync tasks, and the browser persists local changes via localStorage.

The system has three layers: the dashboard HTML (the UI), JSON data files (Claude's read/write layer), and Monday.com (the upstream source of truth for work tasks).

---

## Architecture

### File Layout

```
cool-hits-task-manager/ (GitHub: cobbinator27/cool-hits-task-manager)
├── CLAUDE.md                   <- Auto-read by Claude Code on every session (entry point)
├── SYSTEM-DOC.md               <- This file (master reference)
├── scope-multi-device-task-manager.md  <- Historical planning doc (Phase 1 now implemented)
├── dashboard.html              <- The dashboard (Vercel-hosted; also openable locally)
├── vercel.json                 <- Vercel routing config
├── .claude/
│   └── skills/
│       └── task-manager/
│           ├── SKILL.md        <- Claude-facing skill (auto-loads in any session that opens this repo)
│           └── references/
│               └── supabase-api-reference.md
├── data/
│   ├── config.json             <- Monday board IDs, user ID, sync config
│   ├── supabase-config.json    <- Credentials (gitignored — NEVER commit)
│   ├── seo-team-ops.json       <- Last-sync snapshot of SEO work tasks
│   ├── content-ops.json        <- Last-sync snapshot of content pipeline
│   ├── personal.json           <- Last-sync snapshot of personal tasks
│   ├── jobs.json               <- Last-sync snapshot of job listings
│   ├── briefs.json             <- Brief automation pipeline data (read-only)
│   └── backups/                <- Auto-created by sync task (gitignored)
├── starter-kit/
│   ├── dashboard.html          <- Clean starter version (for sharing with colleagues)
│   └── SETUP-GUIDE.md          <- Setup instructions for the starter kit
├── task-manager.plugin         <- Cowork plugin package (built artifact, ZIP)
└── task-manager.skill          <- Skill package (built artifact, ZIP)
```

**Note on skill path:** `.claude/skills/task-manager/` is the Claude Code convention. Any Claude Code session that opens this repo (desktop CLI, laptop, phone via claude.ai/code) will auto-load the skill. Previously lived at `skill/SKILL.md`; moved April 12, 2026 to enable cross-device auto-loading.

### Persistence Model

**Supabase is the source of truth for task data.** The dashboard loads data from Supabase on page init via REST, and writes back to Supabase as you interact with it (drag, status change, add comment, etc.). localStorage is used only for UI state (active tab, dark mode preference, view settings) — not for task data.

**Key Supabase tables:**
- `tasks` (with `scope` column: `'work'` or `'personal'`)
- `content_items`
- `job_listings`
- `task_order` (drag-and-drop sort order per group)
- `deleted_tasks` (IDs to never resurrect)
- `sync_meta` (last-sync timestamps, etc.)

**Local JSON files in `data/`** are kept as read-only snapshots written by the sync task after each Monday pull. They're a backup/audit trail, not a live data source for the dashboard.

### Data Flow

```
Monday.com ──sync task──> Supabase ──REST read/write──> Dashboard (Vercel)
                              │                               ↑
                              │                        any device / browser
                              ↓
                      local JSON snapshots
                      (backup only)
```

**Inbound:** Scheduled sync tasks on Daniel's PC pull from Monday.com and upsert to Supabase.

**Outbound:** Comments flagged `pushToMonday: true` get posted to Monday by the outbound sync step (same task), which then updates the Supabase row to set `pushedToMonday: true`.

**AI task creation (any device):** Claude Code (desktop CLI, laptop, or claude.ai/code on phone) writes directly to Supabase via the REST API. Tasks appear on the dashboard immediately on next page interaction.

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
- Silent background poll every 60 seconds (fetches fresh data from Supabase without reloading the page, so view state is preserved)
- Quick-add bar with bucket, priority, and project dropdowns
- Editable task names, statuses, priorities, due dates, projects in detail panels
- Comment system with Monday.com push support
- Project tagging with hub tasks, filtering, and grouping

---

## Working Across Devices

Daniel works from three contexts:

1. **Desktop PC** — Always-on automation hub: Cowork, Monday sync, LinkedIn job scans
2. **Laptop** — Primary daily driver for deep work and task management
3. **Phone** — claude.ai/code for quick task creation and status checks while away from a computer

### Two Source-of-Truth Systems (Important to Separate)

| What | Source of Truth | Sync Mechanism |
|------|-----------------|----------------|
| **Task data** (tasks, comments, content items, jobs) | **Supabase** | Real-time. All devices read/write directly via REST. No git involved. |
| **Code and docs** (skill, dashboard.html, SYSTEM-DOC.md, CLAUDE.md) | **GitHub repo** | Manual. `git pull` before editing, `git push` after. |

**Practical implication:** Creating or editing tasks from any device is instant and cross-device by default (Supabase handles it). Only changes to how the system works — skill updates, dashboard code, documentation — require git workflow.

### Device Setup

| Device | Skill availability | Supabase credential |
|--------|-------------------|---------------------|
| Desktop PC | Auto-loads when Claude Code opens the cloned repo | `data/supabase-config.json` (gitignored local file) |
| Laptop | Auto-loads when Claude Code opens the cloned repo | Same as PC — file on disk |
| Phone / any browser (claude.ai/code) | Auto-loads from `.claude/skills/task-manager/` when the repo is attached to a session | `SUPABASE_SERVICE_KEY` env var set in the claude.ai/code environment config |

The skill's credential-loading snippet handles both cases transparently — it reads the config file if present, otherwise falls back to the env var. See `.claude/skills/task-manager/SKILL.md` for the exact snippet.

### Git Workflow (Plain-English for Non-Developers)

- Think of GitHub as the master copy. Your laptop/PC folder is a working copy that you sync up and down.
- **Before editing code or docs on a device you haven't used in a while:** `git pull` to bring down the latest. (Claude can do this for you — just ask.)
- **After editing:** commit and push. Other devices will be stale until they pull.
- **Phone sessions** (claude.ai/code): Claude handles git automatically. It clones fresh, edits, and pushes back to GitHub. Your laptop/PC then needs to `git pull` to catch up.

### Conflict Prevention

The only real risk is editing the same file on two devices without syncing in between. Mitigation: always `git pull` before starting an editing session on a given device. If a conflict happens, Claude can help resolve it — it's annoying but not catastrophic.

### Environment Variable Setup (One-Time Per Environment)

The Supabase service key needs to be available wherever Claude Code runs:

- **Desktop CLI (PC/laptop):** either set in `~/.claude/settings.json` under `environment`, or rely on the gitignored `data/supabase-config.json` file. The latter is already in place on Daniel's PC.
- **claude.ai/code web/mobile:** set once in the environment config → Environment variables field. Persists across all sessions in that environment.

Env vars do NOT sync between desktop CLI and web. You set them separately in each place. That's one of the current limitations — see roadmap for potential improvements.

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
- **Does:** Pulls tasks from both Monday boards, upserts to Supabase (`tasks`, `content_items` tables), updates `sync_meta.last_sync`, writes local JSON snapshots to `data/` as backups
- **Key rules:** Check `deleted_tasks` before upserting (never resurrect deleted items), preserve local-source comments when merging Monday data, use upsert (`Prefer: resolution=merge-duplicates`), never touch `dashboard.html`
- **Known issue history:** The sync task originally regenerated data arrays inside `dashboard.html`. It had recurring issues (dropped commas, full-file overwrites wiping UI changes). Fixed April 2026 by migrating to Supabase as the data source; `dashboard.html` is no longer regenerated at all.

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

### ~~The Sync Task Can Break the Dashboard~~
**Status: Resolved (April 2026)** — The sync task no longer touches `dashboard.html`. Data lives in Supabase; the dashboard fetches on load. Class of bug eliminated.

### ~~localStorage Is Per-Browser, Per-Device~~
**Status: Resolved (April 2026)** — Task data now lives in Supabase, not localStorage. All devices see the same data in real time. localStorage only holds UI preferences (dark mode, active tab) which are acceptable to be per-browser.

### Secrets in claude.ai/code Environment Variables
**Severity: Low-Medium** | **Status: Accepted for personal use**

claude.ai/code warns that env vars are "visible to anyone using this environment." For a solo account like Daniel's, this is effectively fine — no one else has access. Note: if the environment is ever shared with collaborators, the `SUPABASE_SERVICE_KEY` would be visible to them. In that case, rotate the key and either use a scoped Secret API key (Supabase's newer rotatable credentials) or move to a setup-script-based secret fetch.

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
| Apr 11, 2026 | v0.8 | Migrated data layer to Supabase; dashboard now loads from DB, no more embedded arrays; added `SUPABASE_SERVICE_KEY` env var fallback |
| Apr 11, 2026 | v0.8.1 | Replaced 60s full-page reload with silent Supabase poll (preserves view state) |
| Apr 12, 2026 | v0.9 | Enabled mobile task creation via claude.ai/code; moved skill to `.claude/skills/task-manager/` for cross-device auto-load; added `CLAUDE.md` as session entry point; added "Working Across Devices" section |

### Plugin Versions
- **v0.6.0** — Current. Updated job fields, project system, job search board docs.

---

## Roadmap / Future Ideas

### Near-term
- [ ] Fix scheduled task folder access (do "Run now" + "Always allow" on job scan tasks)
- [ ] Decide on brief pipeline: fully commit to Monday-first or keep dashboard UI
- [ ] Upload resumes/summaries to Google Drive (blocked on Drive upload MCP)
- [ ] Consider migrating from legacy Supabase `service_role` key to newer Secret API keys (individually rotatable)

### Medium-term
- [x] ~~Multi-device access~~ — **Done (v0.8 + v0.9):** Dashboard is hosted on Vercel, backed by Supabase; Claude Code skill loads cross-device from `.claude/skills/task-manager/`. See `scope-multi-device-task-manager.md` for the original planning analysis.
- [x] ~~Mobile quick-add~~ — **Done (v0.9):** claude.ai/code on phone now creates tasks directly to Supabase using the shared skill.
- [x] ~~Capture browser-side edits~~ — **Done (v0.8):** Dashboard writes directly to Supabase; no more localStorage gap for task data.
- [ ] Auto-archive jobs with no activity after 4 weeks
- [ ] Notification system (upcoming due dates, gated items ready to run)
- [ ] Install the skill as a Claude Code plugin globally (so it works outside this repo) — may not be needed now that repo-attached skill works cross-device

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
