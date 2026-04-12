# Cool Hits 2.0 — Multi-Device Scope

## The Problem

Daniel needs his task manager accessible from three contexts:

1. **Desktop PC** — runs Cowork, job scans, Monday.com sync (always-on automation hub)
2. **Laptop** — daily work in Cowork (enterprise account), active task management
3. **Phone** — quick task viewing, adding tasks on the go

Currently everything is local: an HTML file + JSON files on one machine. No cross-device access, no mobile experience.

## The Three Approaches

### Option A: Progressive Web App (PWA) with Cloud Database

**What it is:** A lightweight web app hosted somewhere (Vercel, Netlify, Cloudflare Pages) backed by a simple database (Supabase, Firebase, or Turso/SQLite). The same URL works on desktop browsers and installs as a home-screen app on mobile.

**Architecture:**
- Frontend: Single-page app (React or plain HTML/JS like the current dashboard)
- Backend: Serverless API routes or direct database client
- Database: Supabase (Postgres) or Firebase Firestore
- Auth: Simple — could be a single shared API key since it's personal, or Supabase magic link

**How Claude reads/writes:**
- Claude calls the database API directly (Supabase has a REST API, Firebase has one too)
- OR: Claude writes to a JSON file that gets synced to the database on a schedule
- OR: A lightweight MCP server wraps the database, giving Claude native tool access

**Pros:**
- Real mobile app experience (installable PWA, push notifications possible)
- Accessible from any device with a browser
- Database is a proper source of truth — no merge conflicts
- Can scale to more features (sharing, notifications, etc.)

**Cons:**
- More infrastructure to set up and maintain (hosting, database, auth)
- Claude can't just write a JSON file anymore — needs API calls
- Monthly costs (likely $0-5/mo at personal scale, but not zero)
- More complex debugging when things break

**Claude integration difficulty: Medium.** Supabase REST API is well-documented and Claude can call it via HTTP. Could also build a small MCP server that wraps the database operations, making it feel as natural as writing JSON.

---

### Option B: Shared File Storage (Google Drive / Dropbox / iCloud)

**What it is:** Keep the current HTML + JSON architecture, but move the files to a cloud-synced folder (Google Drive, Dropbox, or iCloud). All devices see the same files.

**Architecture:**
- Same dashboard.html + JSON files, just stored in a synced folder
- Mobile: Open dashboard.html in a mobile browser (limited but functional)
- Desktop: Open the same file locally — it syncs via the cloud provider

**How Claude reads/writes:**
- Same as today — write JSON files, they sync automatically
- On PC: direct file access via mounted folder
- On Laptop: same, if the folder is synced there too

**Pros:**
- Minimal changes to current setup
- Claude keeps writing JSON directly — easiest integration
- No hosting, no database, no API

**Cons:**
- Mobile experience is poor (opening an HTML file on a phone is clunky)
- Sync conflicts are possible if two devices edit simultaneously
- No real "app" feel — no quick-add from phone, no notifications
- Google Drive file:// links still won't work in browsers
- localStorage won't sync across devices (each browser has its own)

**Claude integration difficulty: Low.** Basically the same as today.

---

### Option C: Hybrid — Static App + JSON API

**What it is:** Host the dashboard as a static site (GitHub Pages, Netlify) but keep a JSON file as the data layer, stored somewhere Claude can write to (a GitHub repo, an S3 bucket, or a simple key-value store like Val Town or Deno KV).

**Architecture:**
- Frontend: The current dashboard, hosted as a static site
- Data: JSON files in a GitHub repo or S3 bucket
- Sync: The dashboard fetches JSON on load (instead of having it embedded)
- Claude writes: Push JSON to GitHub (via API) or upload to S3

**How Claude reads/writes:**
- Push updated JSON to a GitHub repo via the GitHub API (Claude already has GitHub tools)
- OR: Write to an S3/R2 bucket via a simple API
- The hosted dashboard fetches the latest JSON on page load

**Pros:**
- Mobile-friendly (it's just a website)
- Claude writes JSON (familiar) and pushes it somewhere accessible
- No database to manage — just static files
- Free hosting (GitHub Pages, Netlify, Cloudflare Pages)
- Could add a simple mobile-optimized view without much work

**Cons:**
- Slightly more latency (fetch JSON on load vs. embedded)
- localStorage per-device issue remains for local edits
- Need to solve write-back: if Daniel edits on the dashboard, how do changes get saved? (Would need a small API endpoint or use GitHub API from the browser)
- Less real-time than a database

**Claude integration difficulty: Low-Medium.** Writing JSON and pushing to GitHub is straightforward. The tricky part is enabling Daniel to save edits from the browser back to the data source.

---

## Recommendation

**Option A (PWA + Supabase) is the right long-term answer** if you want a real mobile experience with quick-add and cross-device sync. The setup cost is a weekend of work, but once it's running, it's the cleanest architecture.

**Option C (Static + JSON API) is the pragmatic middle ground** if you want to move fast. It gets you cross-device access and mobile viewing with minimal changes to the current setup. The limitation is that browser-side edits need a solution (either a thin API or you accept that edits only happen via Cowork/Claude).

**Option B is a band-aid** — it solves file access but doesn't give you a mobile app or a good experience.

## Suggested Path

**Phase 1 (now):** Option C — host the dashboard on GitHub Pages, move data to JSON in a GitHub repo. Claude pushes updates via GitHub API. You can view from any device immediately. Mobile is read-only but functional.

**Phase 2 (when ready):** Option A — migrate to Supabase + PWA. Build a mobile-optimized view with quick-add. Claude gets an MCP server wrapping the database. Full read/write from everywhere.

## Answering Your Specific Questions

**Is it harder for Claude to read/write a database vs. JSON?**
Not significantly. Supabase has a REST API that works like writing JSON — you POST an object and it stores it. Reading is a GET request with filters. An MCP server could wrap this so Claude just calls `create_task()` and `get_tasks()` natively. The main difference: instead of `fs.writeFile('tasks.json', data)`, it's `fetch('https://your-supabase.co/rest/v1/tasks', { method: 'POST', body: data })`. Same data, different transport.

**What does "Claude sees something in email and creates a task" look like?**
- With local JSON: Claude writes to personal.json, task appears on next dashboard reload
- With database: Claude POSTs to Supabase API, task appears immediately on all devices
- With GitHub JSON: Claude pushes to the repo, dashboard fetches on next load (~30s delay)

All three work. The database approach is the most real-time.

**Any other solutions for phone access?**
- **Telegram/Slack bot:** Claude posts tasks to a channel, you add tasks by messaging the bot. No app to build, works on phone immediately. Doesn't replace the dashboard but gives you mobile quick-add.
- **Apple Shortcuts + JSON:** An iOS shortcut that POSTs a task to an API endpoint. Very quick to set up if you go with Option A or C.
- **Obsidian/Notion sync:** Move task data to Notion (has an API Claude can write to, has a mobile app). Downside: you're now in Notion's ecosystem.
