# Flex Structure

A single-page HTML scheduler for **highly repeated daily/weekly tasks with flexible start times**.
Unlike a traditional scheduler, each task doesn't start at a fixed time — it starts inside a
**Flex Window** (± N minutes around the scheduled time).

Example: a task set to **12:30 PM** with a Flex Window of **±15 min** can be started anytime
between **12:15 PM and 12:45 PM**. At 12:15 a banner appears:

> Now, −15 min. of *Lunch*. **Start?**

Pressing **Start** begins the task's countdown (its configured duration, e.g. 60 minutes),
shown on a Time-Timer-style dial.

## The flexible cascade

- At the beginning of each day every schedule sits on its exact set time.
- When you start a task off-time (e.g. the 12:30 one at 12:40), **every later schedule that day
  shifts by the same delta** (+10 min) — and each still keeps its own Flex Window around the
  shifted time. Starting early shifts later tasks earlier, too.
- The original time is shown struck-through in the list whenever a shift is in effect.

## UI

Responsive: single vertical column on phones (top→bottom: Timer, Day, Weeks); on wide screens
a 3-column desktop layout (left→right: Timer, Day, Weeks). The green start banner always
appears at the very top.

| Column | Content |
|---|---|
| **Timer** (left / top) | Time-Timer-style countdown — a red sector that shrinks as the running task's remaining minutes tick down. "Finish now" ends a task early; tasks auto-complete when the countdown reaches zero. |
| **Day** (center) | A 12-hour pie dial (10 AM – 10 PM, **12 PM at the top**). Tasks are drawn as colored wedges at their *effective* (shifted) times with the schedule name written inside each wedge (label length scales with duration: ≤15 min → 2 chars, ≤30 → 5, ≤60 → 8, longer → 12), their Flex Window as an arc on the rim, and a black "now" hand. Below it, a table lists the day's schedules with time, duration, status and Start/Finish buttons. Date headings are shown in Korean (e.g. `금요일, 6월 12일, 2026 (오늘)`). |
| **Weeks** (right / bottom) | 3-week calendar — last / this / next week. Colored dots mark which tasks occur on each day (weekday repeat narrowed by Range / Excluded days). Click a day to inspect it. |

Statuses: `Upcoming → Start now (window open) → Running → Done`, or `Missed` if the window
passed without starting (a missed task can still be started late — the cascade then applies).

**Status colors:** red is not selectable as a schedule color — it is reserved. A schedule that
was **started inside its Flex Window** turns **red** (both its tag in the list and its pie
wedge). A schedule that was **not started in time** (window missed, or started outside the
window) turns **black**. Untouched/upcoming schedules keep their own chosen color.

## Cloud storage (Supabase)

All schedules, settings, and per-day run history are stored as a CSV file in **Supabase
Storage** — the same project and bucket already used by `html_Stock_Monitor`
(note: that project's Cloudflare Worker is only a CORS proxy; the actual data store is Supabase):

| | |
|---|---|
| Project | `https://upqnxyllnenivehmallp.supabase.co` |
| Bucket | `csv-data` (already exists, anon read/write policies already in place) |
| Data file | `flex_structure_data.csv` (dedicated to this app; created on first save) |

**No new setup is required** — the bucket and policies are already configured.
If you ever recreate the project, the needed steps are:

1. Dashboard → Project Settings → API → copy the `anon public` key into `SUPABASE_KEY` in `index.html`.
2. Storage → New Bucket → name `csv-data` (public).
3. Bucket → Policies → New Policy (full customization), role `anon`:
   - `SELECT` → USING `true`
   - `INSERT` + `UPDATE` → USING `true`, WITH CHECK `true`

### Data file format

RFC-4180 key-value CSV with the whole app state as JSON in the `value` cell
(same convention as `stock_pl_settings.csv`):

```csv
key,value
data,"{""settings"":{""defaultFlex"":15},""schedules"":[{""id"":""s1"",""name"":""Lunch"",""time"":""12:30"",""dur"":60,""flex"":15,""days"":[1,2,3,4,5],""color"":""#3b82f6""}],""dayStates"":{""2026-06-12"":{""runs"":{""s1"":{""start"":1781240400000,""done"":true}}}}}"
```

- `settings` — default Flex Window plus Google sync settings: `gcalClientId`, `gcalCalendarId`, `autoSync`.
- `schedules` — task definitions: name, set time, duration (min), Flex Window (± min), weekdays (0=Sun…6=Sat), `range` (`"6.12-7.8"` or empty), `excl` (`"7.19, 7.26"` or empty), color, and `gcalId` once synced to Google Calendar.
- `dayStates` — per-date run records (actual start timestamps + done flags). Entries older than 30 days are pruned automatically on save.
- Saving is automatic (debounced ~1 s after any change); **Save**/**Reload** buttons force it manually. Loads always cache-bust so every device sees fresh data.

## Google Calendar sync

Each schedule can be pushed to Google Calendar as a **recurring event**: weekly `RRULE` from
its weekdays, `UNTIL` from its Range, `EXDATE` from its Excluded days. Re-syncing updates the
existing event (the event id is stored per schedule in the CSV) instead of duplicating it.
Sync also cleans up: schedules deleted in the app have their Google events deleted on the next
sync, duplicates and orphaned app-created events are detected (via a private
`flexStructureId` property on each event) and removed, and a lost event link is recovered by
lookup rather than re-created.

Setup (one time, in [Google Cloud Console](https://console.cloud.google.com/)):

1. Create (or pick) a project → **APIs & Services → Library** → enable **Google Calendar API**.
2. **APIs & Services → OAuth consent screen** → External → add yourself as a test user.
3. **APIs & Services → Credentials → Create Credentials → OAuth client ID** → type
   **Web application** → under *Authorized JavaScript origins* add the origin you serve the
   page from (e.g. `http://localhost:8000`, or your GitHub Pages `https://…` origin).
4. Copy the Client ID into ⚙ **Settings → OAuth Client ID** in the app.

> ⚠️ Google OAuth does not work from a `file://` URL. To sync, serve the page from an
> authorized origin — locally e.g. `python3 -m http.server 8000` in this folder, then open
> `http://localhost:8000`.

In ⚙ Settings:

- **Calendar** — `primary` for your main calendar, a calendar's display name (e.g. `Daily`,
  matched case-insensitively against your calendar list), or a full calendar ID
  (`…@group.calendar.google.com`).
- **Sync Now** — pushes all schedules immediately (first time opens the Google consent popup).
- **Auto Sync on startup** toggle — when on, the app syncs right after loading its data on
  every page open. **Off by default.** All of these settings are saved to the CSV file too.

## Usage

1. Open `index.html` in any browser (double-click, or host it — see Deploy below).
2. Click **+** → set name, time, duration, Flex Window, weekdays, color.
   - **Range** (optional): type `6.12-7.8` to repeat only from June 12 to July 8 (year-boundary ranges like `12.20-1.10` work too).
   - **Excluded days** (optional): type `7.19, 7.26` to skip those dates.
3. ⚙ sets the default Flex Window for new schedules.
4. When a task's window opens, the green banner appears — press **Start**.
5. Watch the remaining time on the timer dial; **Finish now** ends early.

Keep the page open on the device you use during the day; the banner and timer update every second.

## Deploy (optional)

- **GitHub Pages:** push the folder to a repo → Settings → Pages → Source: `main` / root.
- **Netlify:** drag-and-drop the folder at app.netlify.com.

The page is fully static — no build step, no server. The anon key is stored in plain text in
the HTML, as in the Stock Monitor; this assumes personal use.
