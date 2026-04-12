# Cool Hits 2.0 — Supabase API Reference

This document is the single source of truth for reading from and writing to the Supabase database that powers Daniel's Cool Hits 2.0 task dashboard. Include or link to this in any skill, sync task, or scheduled job that touches dashboard data.

---

## Credentials

### If running on Daniel's personal machine
Read credentials from:
```
/Users/danielcobb/Documents/task-manager/data/supabase-config.json
```
```json
{
  "url": "https://bpfvcniuhfzfvigktfci.supabase.co",
  "serviceRoleKey": "<secret — read from file>",
  "anonKey": "<read from file>"
}
```
Use the **service role key** for all writes. It bypasses Row Level Security (RLS).

### If running on another machine (e.g. Cowork)
Ask Daniel to provide the service role key at the start of the task, or store a copy of `supabase-config.json` at the same path on that machine.

The Supabase project URL is always:
```
https://bpfvcniuhfzfvigktfci.supabase.co
```

---

## Standard Request Headers

Use these headers on every Supabase REST call:
```
apikey: <SERVICE_ROLE_KEY>
Authorization: Bearer <SERVICE_ROLE_KEY>
Content-Type: application/json
```
For upserts, add:
```
Prefer: resolution=merge-duplicates
```
For insert-only (error on conflict), omit the `Prefer` header.

---

## Tables

| Table | Primary Key | Contents |
|-------|-------------|----------|
| `tasks` | `id` (text) | Work + personal tasks. Has `scope` column ('work'/'personal'). |
| `content_items` | `id` (text) | Content pipeline items. |
| `job_listings` | `id` (text) | Job search listings. |
| `task_order` | `order_key` (text) | Manual drag-drop sort orders per group. |
| `deleted_tasks` | `id` (text) | IDs of tasks Daniel has deleted — never re-create these. |
| `sync_meta` | `key` (text) | Key/value store for sync timestamps etc. |

All tables (except `task_order` and `deleted_tasks`) follow this column pattern:
```
id          text  PRIMARY KEY
data        jsonb              ← full object stored here
updated_at  timestamptz
```
`tasks` additionally has:
```
scope  text  ← 'work' or 'personal'
```

---

## REST API Patterns

Replace `{URL}` with the project URL and `{KEY}` with the service role key.

### Upsert (insert or update)
```bash
curl -s -X POST "{URL}/rest/v1/{table}" \
  -H "apikey: {KEY}" \
  -H "Authorization: Bearer {KEY}" \
  -H "Content-Type: application/json" \
  -H "Prefer: resolution=merge-duplicates" \
  -d '[{ "id": "...", "data": {...}, "updated_at": "2026-04-11T12:00:00Z" }]'
```

### Read all rows
```bash
curl -s "{URL}/rest/v1/{table}?select=*" \
  -H "apikey: {KEY}" \
  -H "Authorization: Bearer {KEY}"
```

### Read with filter
```bash
# Exact match
curl -s "{URL}/rest/v1/tasks?scope=eq.work&select=id,data" \
  -H "apikey: {KEY}" -H "Authorization: Bearer {KEY}"

# Multiple filters (AND)
curl -s "{URL}/rest/v1/tasks?scope=eq.work&select=id,data&id=eq.w-123" \
  -H "apikey: {KEY}" -H "Authorization: Bearer {KEY}"
```

### Read a single row by ID
```bash
curl -s "{URL}/rest/v1/tasks?id=eq.{TASK_ID}&select=id,data" \
  -H "apikey: {KEY}" -H "Authorization: Bearer {KEY}"
```

### Delete a row
```bash
curl -s -X DELETE "{URL}/rest/v1/{table}?id=eq.{ID}" \
  -H "apikey: {KEY}" \
  -H "Authorization: Bearer {KEY}"
```

### Update sync timestamp
```bash
curl -s -X POST "{URL}/rest/v1/sync_meta" \
  -H "apikey: {KEY}" \
  -H "Authorization: Bearer {KEY}" \
  -H "Content-Type: application/json" \
  -H "Prefer: resolution=merge-duplicates" \
  -d '[{"key":"last_sync","value":"2026-04-11T12:00:00Z","updated_at":"2026-04-11T12:00:00Z"}]'
```

---

## Data Shapes

### Work Task
```json
{
  "id": "w-<mondayId or unique string>",
  "name": "Task name",
  "group": "today",
  "priority": "Boulder",
  "status": "Not Started",
  "source": "seo",
  "mondayId": "12345678",
  "boardId": 6197128988,
  "due": "2026-04-15",
  "category": "Strategy",
  "domain": "HireaHelper",
  "url": null,
  "completed": false,
  "project": null,
  "isProjectHub": false,
  "comments": []
}
```

**`group` values:** `today` `tomorrow` `thisWeek` `later` `backlog`

**`priority` values:** `Critical` `Boulder` `Pebble` `High` `Medium` `Low` `""`

**`status` values:** `Not Started` `In Progress` `Working on it` `On Deck` `Blocked` `Waiting` `Ready for Review` `Done`

**`source` values:** `seo` `content` `local`

### Personal Task
Same shape as work task, but:
- `id` prefix: `p-`
- `scope` in the Supabase row: `"personal"`
- `source`: `"local"`
- `mondayId`, `boardId`, `domain`: typically `null`

### Content Pipeline Item
```json
{
  "id": "c-<mondayId>",
  "name": "Article title",
  "stage": "Writing",
  "owner": "Daniel",
  "writer": "Daniel",
  "editor": "Melanie",
  "nextDue": "2026-04-20",
  "nextDueLabel": "Edits",
  "type": "Rewrite",
  "tag": "TLLP",
  "url": "https://..."
}
```

**`stage` values:** `Not Started` `Brief Ready` `Editing Brief` `Writing` `Editing` `Uploading` `Ready for Publishing`

### Job Listing
```json
{
  "id": "j-<source>-<YYYYMMDD>-<counter>",
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
  "summary": "...",
  "pros": "...",
  "cons": "...",
  "source": "LinkedIn",
  "salary": "$120k-150k",
  "location": "Remote",
  "postedDate": "2026-04-05",
  "foundDate": "2026-04-07",
  "appliedDate": null,
  "comments": []
}
```

**`status` values:** `New` `Applied` `Contacted` `Declined` `Archived`

**`subStatus` values:**
- When `Contacted`: `Promised Resume Review` `Recruiter Screen` `Interview Stage` `Offer`
- When `Declined`: `I Declined` `Company Declined`
- All others: `null`

**`tier` values:** `1` (strong fit) `2` (good fit) `3` (stretch/speculative)

### Comment (on tasks or job listings)
```json
{
  "text": "Update text. URLs are rendered as clickable links.",
  "timestamp": "2026-04-11T12:00:00Z",
  "source": "local",
  "pushToMonday": false,
  "pushedToMonday": false,
  "mondayUpdateId": null
}
```

---

## Guardrails

1. **Check `deleted_tasks` before upserting.** Always read the `deleted_tasks` table first and skip any item whose ID appears there. Daniel deleted those intentionally — do not bring them back.

2. **Never overwrite local comments.** When merging Monday data into a task, preserve all comments with `source: "local"`. Only add or update entries with a `mondayUpdateId`.

3. **Preserve unknown fields.** When updating a task, read the existing row from Supabase first, then merge your changes into it. Never send a partial object that drops fields you didn't intend to remove.

4. **Use upsert, not insert.** Always use `Prefer: resolution=merge-duplicates` so re-running a sync is idempotent.

5. **Never delete from `tasks`, `content_items`, or `job_listings`.** Only upsert. The dashboard and Daniel's own actions manage deletions.

6. **Do not touch `dashboard.html`.** The dashboard is a static file on Vercel that reads from Supabase on load. There is no longer any need to regenerate or modify it during syncs.

7. **Content Ops board is read-only.** Pull from it, never push to it, unless Daniel explicitly asks.

8. **ID uniqueness.** For new AI-created tasks: `w-ai-{timestamp}` for work, `p-ai-{timestamp}` for personal, `j-{source}-{YYYYMMDD}-{counter}` for jobs. Never reuse an existing ID.

---

## Monday.com Board Reference

| Board | ID | Purpose |
|-------|----|---------|
| SEO Team Ops | `6197128988` | Daniel's work tasks |
| Content Ops: PMG | `6197228419` | Content pipeline (read-only) |

**Daniel's Monday user ID:** `68065823`

### SEO Team Ops column IDs
| Column | ID |
|--------|----|
| Priority | `color_mkrn7asr` |
| Status | `project_status` |
| Due date | `date` |
| Category | `label` |
| Domain | `status_1_mkm5f38j` |

### Content Ops: PMG column IDs
| Column | ID |
|--------|----|
| Stage | `dup__of_cs___status` |
| Owner | `people__1` |
| Writer | `person` |
| Editor | `people` |
| Brief Due | `date_mkny4yxd` |
| Writing Due | `date4` |
| Edits Due | `date__1` |
| Upload Due | `date_Mjj25RxX` |
| Publishing Due | `date2__1` |
