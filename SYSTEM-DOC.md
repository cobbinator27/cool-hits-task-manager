# Cool Hits 2.0 — System Documentation

*Last updated: April 13, 2026*

This is the master reference doc for Daniel's task management system. It covers what exists, how it works, what's broken, what's planned, and how to maintain it. Keep this updated as the system evolves.

---

## What This Is

A personal task dashboard that replaces the need to live inside Monday.com all day. The dashboard is a static HTML file hosted on Vercel, backed by Supabase as the data layer. Data flows in from Monday.com via a scheduled Claude sync task. Any device or Claude session can read and write tasks directly through the Supabase API.

**Live URL:** https://cool-hits-task-manager.vercel.app  
**Repo:** https://github.com/cobbinator27/cool-hits-task-manager (private)

---

## Architecture

```
Monday.com  ──sync task──>  Supabase DB  <──read/write──  dashboard (Vercel)
                                  ↑                              ↑
                           Claude sessions               any device / browser
                           (skill / tasks)
```

### File Layout

```
cool-hits-task-manager/ (GitHub: cobbinator27/cool-hits-task-manager)
├── CLAUDE.md                   ← Auto-read by Claude Code on every session (session entry point)
├── SYSTEM-DOC.md               ← This file
├── dashboard.html              ← The dashboard (Vercel-hosted; also openable locally via file://)
├── vercel.json                 ← Serves dashboard.html at /
├── scope-multi-device-task-manager.md  ← Historical planning doc (Phase 1 now implemented)
├── task-manager.skill          ← Cowork skill package (ZIP, built artifact)
├── task-manager.plugin         ← Cowork plugin package (ZIP, built artifact)
├── .claude/
│   └── skills/
│       └── task-manager/
│           ├── SKILL.md        ← Source for the skill (auto-loads in any Claude Code session)
│           └── references/
│               └── supabase-api-reference.md  ← Standalone Supabase REST reference
├── starter-kit/
│   ├── dashboard.html          ← Clean starter version (for sharing with colleagues)
│   └── SETUP-GUIDE.md          ← Setup instructions for the starter kit
└── data/
    ├── supabase-config.json    ← Supabase URL + keys (gitignored — never commit)
    ├── config.json             ← Monday board IDs, user ID
    ├── seo-team-ops.json       ← Snapshot written after each Monday sync (backup only)
    ├── content-ops.json        ← Snapshot written after each Monday sync (backup only)
    ├── personal.json           ← Personal tasks snapshot (backup only)
    ├── jobs.json               ← Job listings snapshot (backup only)
    ├── briefs.json             ← Brief pipeline snapshot
    └── backups/                ← Auto-created by sync task (gitignored)
```

**Note on skill path:** `.claude/skills/task-manager/` is the Claude Code convention — any Claude Code session that opens this repo (desktop CLI, laptop, or phone via claude.ai/code) auto-loads the skill. Previously lived at `skill/SKILL.md`; moved April 12, 2026 to enable cross-device auto-loading.

### Persistence Model

Supabase is the source of truth. localStorage is a write-through cache for snappy UI.

1. **On page load** — `init()` authenticates via Supabase magic link (skipped on `file://`), then calls `loadFromSupabase()` to fetch all data. Page renders after load.
2. **On every edit** — `saveState()` writes to localStorage immediately (sync) then calls `syncToSupabase()` in the background (async, fire-and-forget).
3. **Every 3 minutes** — the dashboard silently checks `sync_meta.last_sync`. If it changed (e.g. a sync task ran), it re-fetches from Supabase and re-renders in place. No page reload.
4. **Claude sessions** — read/write directly to Supabase via REST API using the service role key.

### Auth

Magic link email auth via Supabase. Locked to `dacobb27@gmail.com` via RLS policies. "Allow new users" is disabled in Supabase Auth settings — only the pre-approved email can log in.

On `file://` protocol (local), auth is skipped entirely and the app falls back to localStorage.

---

## Supabase Tables

| Table | Contents |
|-------|----------|
| `tasks` | Work + personal tasks. Columns: `id`, `scope` ('work'/'personal'), `data` (jsonb), `updated_at` |
| `content_items` | Content pipeline items. Columns: `id`, `data`, `updated_at` |
| `job_listings` | Job search listings. Columns: `id`, `data`, `updated_at` |
| `task_order` | Manual drag-drop sort order. Columns: `order_key`, `task_ids` (text[]) |
| `deleted_tasks` | IDs of tasks Daniel deleted. Columns: `id`, `deleted_at` |
| `sync_meta` | Key/value store (e.g. `last_sync`). Columns: `key`, `value`, `updated_at` |

Full API reference → `.claude/skills/task-manager/references/supabase-api-reference.md`

---

## Dashboard Tabs

### Work Tab
- **My Tasks** — Board view (Today/Tomorrow/This Week/Later/Backlog columns) or List view. Drag-and-drop with manual sort order persisted to Supabase. Groupable by timeline, priority, source, status, or project.
- **Content Pipeline** — Kanban by stage (Not Started → Brief Ready → Writing → Editing → Uploading → Ready for Publishing). Read-only from Content Ops: PMG board.

### Personal Tab
- **My Tasks** — Same layout as Work, independent task list.
- **Job Search** — Hybrid 2×3 grid (Tier 1/Tier 2/Contacted on top, Tier 3/Declined/Archived on bottom) plus a pipeline kanban view.

### Briefs Tab
- Read-only status monitor for the content brief automation pipeline. Decision: Monday.com is the primary UI for briefs. This tab is a diagnostic/spot-check view only.

### Shared Features
- Dark/light mode (persisted to localStorage)
- Silent 3-minute background sync (polls `sync_meta`, re-renders if changed — no page reload)
- Quick-add bar with bucket, priority, and project dropdowns
- Editable task names, statuses, priorities, due dates, projects in detail panels
- Comment system with Monday.com push support
- Project tagging with hub tasks, filtering, and grouping
- Drag-and-drop manual sort order within columns (order saved to Supabase)

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
- Priority: `color_mkrn7asr`
- Status: `project_status`
- Due: `date`
- Category: `label`
- Domain: `status_1_mkm5f38j`
- Quarter: `color_mm1zeeck`

**Content Ops: PMG:**
- Status: `dup__of_cs___status`
- Owner: `people__1` | Writer: `person` | Editor: `people`
- Dates: `date_mkny4yxd` (Brief), `date4` (Writing), `date__1` (Edits), `date_Mjj25RxX` (Upload), `date2__1` (Publishing)

---

## Scheduled Tasks

### sync-monday-to-supabase
- **Schedule:** 8 AM, 10 AM, 1 PM, 4 PM — weekdays
- **Does:**
  1. **Outbound** — reads Supabase tasks, finds comments with `pushToMonday: true`, posts to Monday via MCP, marks them `pushedToMonday: true`
  2. **Inbound** — pulls Daniel's assigned items from SEO Team Ops + Content Ops, merges with existing Supabase rows (preserving local comments), upserts to `tasks` and `content_items` tables, writes JSON snapshots as backup
  3. Updates `sync_meta.last_sync`
- **Key rules:** Never overwrites local comments. Checks `deleted_tasks` before upserting. Never touches `dashboard.html`.
- **To approve tools:** Click "Run now" in sidebar and approve each tool with "Always allow" on first run.

### linkedin-job-scan-morning / linkedin-job-scan-evening
- **Schedule:** 6:05 AM / 6:06 PM daily
- **Lives in:** Cowork (work account Claude Code)
- **Does:** Scans LinkedIn for content/SEO leadership roles, scores and tiers them, creates tailored resumes, writes to Supabase `job_listings` table (and `jobs.json` snapshot), emails Daniel for Tier 1 opportunities
- **Credential pattern:** Reads `SUPABASE_SERVICE_KEY` env var (set in Cowork Claude Code settings). Reference: `.claude/skills/task-manager/references/supabase-api-reference.md`.

### brief-orchestrator
- **Schedule:** Disabled
- **Status:** On hold — Monday-first approach for briefs is working.

---

## Credentials & Secrets

| Secret | Where stored | Used by |
|--------|-------------|---------|
| Supabase service role key | `data/supabase-config.json` (gitignored) | Local Claude sessions |
| `SUPABASE_SERVICE_KEY` env var | Claude Code Settings → Environment | Cloud sessions, Cowork, GitHub Codespaces |
| Supabase anon key | Embedded in `dashboard.html` | Browser (public, safe to expose) |

**Credential loading order in skill/tasks:**
1. Try to read `data/supabase-config.json`
2. Fall back to `$SUPABASE_SERVICE_KEY` env var

---

## Skill & Plugin

The task creation skill tells Claude how to add/update tasks from any session.

- **Source (canonical):** `.claude/skills/task-manager/SKILL.md` — edit this file when updating the skill. Auto-loads in any Claude Code session that opens this repo.
- **Packaged as:** `task-manager.skill` and `task-manager.plugin` (both ZIPs, built artifacts for older Cowork-style distribution)
- **To update:** Edit `.claude/skills/task-manager/SKILL.md` and commit. If you also want to refresh the packaged ZIPs, repack:
  ```bash
  mkdir -p /tmp/skill_repack/task-manager
  cp .claude/skills/task-manager/SKILL.md /tmp/skill_repack/task-manager/SKILL.md
  cd /tmp/skill_repack && zip -r /path/to/task-manager.skill task-manager/
  # Repack .plugin: extract, replace both SKILL.md copies, rezip
  ```
- **Works from:** Local machine (reads config file), cloud Claude Code web/mobile (reads env var), GitHub Codespaces (reads env var)

---

## Working Across Devices

Daniel works from three contexts:
1. **Desktop PC** — always-on automation hub: Cowork, Monday sync, LinkedIn job scans
2. **Laptop** — primary daily driver for deep work and task management
3. **Phone** — claude.ai/code for quick task creation and status checks away from a computer

### Two Source-of-Truth Systems (Don't Confuse These)

| What | Source of Truth | Sync Mechanism |
|------|-----------------|----------------|
| **Task data** (tasks, comments, content items, jobs) | **Supabase** | Real-time. All devices read/write directly via REST. No git involved. |
| **Code and docs** (skill, dashboard.html, SYSTEM-DOC.md, CLAUDE.md) | **GitHub repo** | Manual. `git pull` before editing, `git push` after. |

**Practical implication:** Creating or editing tasks from any device is instant and cross-device by default (Supabase). Only changes to how the system works — skill updates, dashboard code, documentation — require git workflow.

### Device Setup

| Device | Skill availability | Supabase credential |
|--------|-------------------|---------------------|
| Desktop PC | Auto-loads when Claude Code opens the cloned repo | `data/supabase-config.json` (gitignored local file) |
| Laptop | Auto-loads when Claude Code opens the cloned repo | Same as PC — file on disk |
| Phone / any browser (claude.ai/code) | Auto-loads from `.claude/skills/task-manager/` when the repo is attached to a session | `SUPABASE_SERVICE_KEY` env var set in the claude.ai/code environment config |

The skill's credential-loading snippet handles both cases transparently — reads the config file if present, otherwise falls back to the env var.

### Git Workflow (Plain-English)

- GitHub is the master copy. Your laptop/PC folder is a working copy that you sync up and down.
- **Before editing code or docs on a device you haven't used in a while:** `git pull` first. Claude can do this for you.
- **After editing:** commit and push. Other devices will be stale until they pull.
- **Phone / cloud sessions:** Claude handles git automatically — clones fresh, edits, and pushes. Your laptop/PC then needs to `git pull` to catch up.
- **Conflicts:** the main risk is editing the same file on two devices without syncing in between. Mitigation: always `git pull` before starting an editing session.

### Env Var Setup (One-Time Per Environment)

Env vars do NOT sync between desktop CLI and claude.ai/code web. You set them separately in each place:
- **Desktop CLI (PC/laptop):** either set in `~/.claude/settings.json` under `environment`, or rely on the gitignored `data/supabase-config.json` file.
- **claude.ai/code web/mobile:** set once in the environment config → Environment variables field. Persists across all sessions in that environment.

---

## Known Issues & Watchouts

### Job Search Resume Links Don't Work
**Severity: Low** | **Status: Open**

The dashboard shows local file paths for resumes/summaries. Browsers block `file://` links. Deferred — needs a cloud storage solution (Google Drive upload not yet supported by MCP).

### Mobile UI is Desktop-Only
**Severity: Low** | **Status: Accepted**

The dashboard has basic responsive breakpoints (1-column board at 768px) but was designed for desktop. No dedicated mobile pass has been done. Functional but cramped on phone.

### Briefs Tab Has Dead Editing Code
**Severity: Info** | **Status: Accepted**

Full brief management UI is built but hidden/read-only. Leaving in place in case the Monday-first approach is abandoned.

---

## Active Projects

### Work
- **HaH Migration** — HireAHelper site migration tasks

### Personal
- **Simple Budgets** — Personal budgeting app

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
| Apr 9, 2026 | v0.6.1 | Fixed briefs tab always visible |
| Apr 10, 2026 | v0.7 | Fixed sync task breaking dashboard, added backup + validation |
| Apr 10, 2026 | v0.7.1 | Added project dropdown and Tomorrow to quick-add, editable task names |
| Apr 10, 2026 | v0.7.2 | Sync task now only replaces data arrays (not full file rewrite) |
| Apr 11, 2026 | v0.8 | **Supabase migration** — replaced JSON/localStorage architecture with Supabase as data layer. Dashboard hosted on Vercel. Magic link auth. Multi-device access. |
| Apr 11, 2026 | v0.8.1 | Three bug fixes: smart auto-reload (skip if typing/card open), deleted task dedup, drag-and-drop manual sort order |
| Apr 11, 2026 | v0.8.2 | Replaced 60s page reload with silent 3-minute Supabase poll |
| Apr 11, 2026 | v0.8.3 | Skill updated for Supabase architecture + env var credential fallback for cloud sessions |
| Apr 12, 2026 | v0.9 | Mobile task creation via claude.ai/code enabled: moved skill to `.claude/skills/task-manager/` for cross-device auto-load; added `CLAUDE.md` as session entry point; added "Working Across Devices" section; added historical-status note to scope doc |
| Apr 12, 2026 | v0.9.1 | Mobile UI pass: responsive CSS for phones (<600px), touch-visible task actions, 44px tap targets, padding reduction, job grid single-column breakpoint. Added "Reviewed" job status between New and Applied. Job posted date now prominent on cards. Job search counter shows only New. |

---

## Roadmap

### Near-term
- [ ] Run Monday sync task → approve tools with "Always allow" on first run
- [ ] Update Cowork job scan task to write to Supabase `job_listings` table
- [x] ~~Set `SUPABASE_SERVICE_KEY` env var in Claude Code Settings for cloud session support~~ — **Done Apr 12:** set via claude.ai/code environment config
- [x] ~~Mobile UI pass (proper touch targets, tab nav, etc.)~~ — **Done Apr 12:** responsive CSS, touch actions, padding reduction
- [ ] Consider migrating from legacy Supabase `service_role` key to newer Secret API keys (individually rotatable)

### Medium-term
- [ ] Upload resumes/summaries to cloud storage (blocked on Drive upload support)
- [ ] Auto-archive jobs with no activity after 4 weeks
- [ ] Notification system (upcoming due dates, gated items ready to run)

### Someday/Maybe
- [ ] Auto-push status changes to Monday (currently only comments push)
- [ ] AI-powered daily standup summary
- [ ] Shared read-only dashboard view for team

---

## How to Maintain This Doc

Update this file whenever:
- A new tab or major feature is added
- A scheduled task is created or modified
- A known issue is discovered or resolved
- A roadmap item is started or completed
- The architecture changes

The `.claude/skills/task-manager/SKILL.md` is Claude-facing — how to interact with the system. `CLAUDE.md` is the session entry point (auto-loaded). This doc is Daniel-facing — how the system works, what's broken, where it's going. Keep all three in agreement.
