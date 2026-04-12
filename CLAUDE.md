# Cool Hits 2.0 — Repo Guide for Claude

This file is auto-read at the start of every Claude Code session on every device (desktop CLI, laptop, phone web). Read it first. Then read the skill and system doc below as needed.

## What This Repo Is

Daniel's personal task management system. A dashboard (Vercel-hosted HTML) backed by Supabase, fed by Monday.com syncs. Used for work tasks, personal tasks, content pipeline, and job search tracking.

## Read These Before Acting

1. **`.claude/skills/task-manager/SKILL.md`** — How to read/write tasks. Loaded automatically by Claude Code; still read it if you're doing anything with tasks.
2. **`.claude/skills/task-manager/references/supabase-api-reference.md`** — Exact REST API patterns for Supabase.
3. **`SYSTEM-DOC.md`** — The master reference doc. Architecture, known issues, roadmap, version history. Read this when you need to understand *why* something is the way it is.

## Two Sources of Truth (Important — Don't Confuse These)

| What | Lives in | Changes flow via |
|------|----------|------------------|
| **Task data** (tasks, comments, job listings, content items) | **Supabase** database | Direct REST writes. No git. |
| **Code & docs** (skill, dashboard.html, SYSTEM-DOC.md, this file) | **GitHub** repo | Git commit + push. |

Day-to-day task creation = Supabase only, no git. Changes to how the system works = git.

## Credentials

The Supabase key loads via this snippet (already in the skill). Use it at the start of any Supabase operation:

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

- **Local machine:** reads `data/supabase-config.json` (gitignored)
- **Cloud Claude Code (web/phone):** reads `$SUPABASE_SERVICE_KEY` env var set in the environment config at claude.ai/code

## Hard Rules

1. **Never commit the Supabase key.** The `data/supabase-config.json` file is gitignored. Keep it that way. Never inline the key into code or docs.
2. **Never print a secret's value in tool output.** When verifying a key is set, use a check that only echoes `SET` / `NOT_SET`, not the value:
   ```bash
   [ -n "$SUPABASE_SERVICE_KEY" ] && echo "SET" || echo "NOT_SET"
   ```
3. **Don't touch `dashboard.html`'s data arrays.** The dashboard now loads data from Supabase on init; it does not need regenerated data arrays. (See SYSTEM-DOC.md for history of why this rule exists.)
4. **Use upsert, not insert**, when writing to Supabase (`Prefer: resolution=merge-duplicates`).
5. **Check `deleted_tasks` before upserting** a task — Daniel deleted those on purpose.
6. **Content Ops (Monday board 6197228419) is read-only.** Never push to it unless Daniel explicitly asks.

## Working Across Devices

Daniel works from three places: desktop PC (Cowork, scheduled syncs), laptop (primary daily driver), and phone (claude.ai/code for quick task creation).

- **Daily task management from any device:** goes to Supabase. Real-time, no sync needed.
- **Editing code or docs in this repo:** go through git.
  - Before editing on a device you haven't used in a while: `git pull origin main` (or the branch you're on).
  - After editing: commit and push. Other devices should `git pull` before their next edit.
- **Env var setup:** set once per Claude Code "environment" (see claude.ai/code → environment settings). Persists across web/mobile sessions in that environment. Must also be set separately on desktop CLI machines (in `~/.claude/settings.json` or shell env).

If in doubt about any of this, SYSTEM-DOC.md has the full explanation under "Working Across Devices."

## When You Make Changes

If the change affects how the system works (architecture, new feature, new rule, new device in the mix):
- Update `SYSTEM-DOC.md` (especially version history and roadmap)
- If it affects Claude's behavior: update `.claude/skills/task-manager/SKILL.md`
- If it's a new golden rule: update this file

Keep all three in agreement. The goal: any Claude session, on any device, should read these files and give Daniel consistent answers.
