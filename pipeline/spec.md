# 📋 Spec: tempo — v1

**Status:** ✅ Approved
**Date:** 2026-04-22
**Project:** tempo (personal workspace) — SvelteKit PWA sibling to kit-db

---

## 🎯 Overview

- **Problem:** Nicolas's workplace forbids reminders and notifications, so nothing surfaces tasks he intends to do — they get dropped. His notes live in scattered `.txt` files and are effectively unsearchable. He wants one dashboard for the rhythm of the working day: daily routines, current work items, and notes.
- **Target user:** Nicolas (sole user). Local-only, single-machine, no auth, no sync.
- **Platforms:** macOS (primary) via Chrome/Safari as an installed PWA. Same codebase runs anywhere Bun runs, but v1 is optimised for Apple's system font stack + macOS window chrome.
- **Delivery:** local web app served by `bun start` on `http://127.0.0.1:3001`, installed as a PWA (Chrome/Safari "Install as app") for a standalone window + dock icon.
- **Thesis:** anti-badgering. Nudges exist (age labels, window-title counter, row-level triage) but nothing auto-fires a modal or notification.

---

## ✅ Requirements

### FR-001: Daily list

- **Description:** A column on the Dashboard listing the user's **recurring routines**. Each row is an instance of a template for today. Template defines title, optional one-line note, optional time-estimate tag (`15m` / `30m` / `1h` / `longer`), and schedule (`every_day` or `weekdays`). Each morning, today's instances are materialised from templates whose schedule matches today's weekday. A new instance has `completed_at = NULL`. Rows are created, completed, edited, and triaged inline — no create-modal.
- **Error handling:** 💬 toast on write failures (reuse `friendlyError()` + toast pattern from kit-db). Inline red text under title input for validation (empty title).
- **Acceptance criteria:**
  - [ ] A user can add a daily routine via the inline adder at the top of the Daily column; creating it also materialises an instance for today if the schedule matches.
  - [ ] A user can toggle a task between **every day** and **weekdays only** from the task's expanded editor (click row to expand).
  - [ ] At 00:01 local time on a new day, today's instances materialise for each template whose schedule matches today.
  - [ ] Clicking the checkbox on an instance sets `completed_at = now()`; unchecking clears it.
  - [ ] Completed instances render with a strike-through style and muted text colour.
  - [ ] The column header shows `N of M done · <schedule-summary>` and a small 40×40 progress ring reflecting the ratio.
  - [ ] Archiving a template (edit → delete in expanded row) sets `archived_at`; future days stop materialising instances but history remains searchable.

### FR-002: Working list

- **Description:** A column on the Dashboard listing **multi-day items** that persist until completed or dropped. No daily reset. Completed items move into a collapsed "Recently completed" section at the bottom of the column for 7 days, after which they are only visible via the History view. Rows show a note-count badge when the item has linked notes.
- **Error handling:** 💬 toast on write failures. Inline red text for validation.
- **Acceptance criteria:**
  - [ ] A user can add a working item via the inline adder at the top of the Working column.
  - [ ] Clicking the checkbox sets `completed_at = now()`; the row enters the "Recently completed" collapsed section immediately.
  - [ ] "Recently completed" shows count in its header; expanding reveals completed items from the last 7 days.
  - [ ] Items with `completed_at` older than 7 days no longer appear on the Dashboard; they remain in `/history` forever.
  - [ ] Each working item shows a note-count badge (linked notes via `notes.work_item_id`) when > 0; clicking the badge scrolls the Notes strip to filter to this item's notes.
  - [ ] Clicking a row expands an inline editor for title / time-estimate / note.

### FR-003: Notes

- **Description:** A full-width strip below Daily + Working. **Markdown** notes, each with a title and body. A note is optionally linked to exactly one task (daily template OR working item; one-note-to-one-task cardinality). A task can have many notes linked to it. Clicking a note card **replaces the strip contents inline** with an editor (title input + markdown textarea + rendered preview toggle). Notes grid shows ~4 cards across with a 3-line excerpt and metadata (link pill if task-linked, "Updated …" otherwise).
- **Error handling:** 💬 toast on write failures. Inline validation for empty title.
- **Acceptance criteria:**
  - [ ] A user can create a note via the inline "New note…" adder in the Notes strip header.
  - [ ] A note card shows: linked-task pill (if any), title, 2–3 line excerpt rendered from markdown, relative updated time.
  - [ ] Clicking a note card replaces the strip's grid with an inline editor (title + editor + Markdown preview mode), with a "Back to notes" / "Close" control to return.
  - [ ] From the editor, a user can link the note to a daily template or working item (combobox showing all active tasks) — setting `task_template_id` OR `work_item_id` (mutually exclusive).
  - [ ] Deleting the linked task unlinks the note (set FK to NULL) — note is preserved.
  - [ ] All notes are full-text-searchable via FTS5 (title + body).
  - [ ] The editor supports the full markdown feature set via `marked` (headings, lists, checkboxes, links, code blocks, bold, italic).
  - [ ] Autosave triggers 500ms after the last keystroke; a subtle "Saved" toast-free indicator shows in the editor footer.

### FR-004: Spillover + age label + row triage

- **Description:** Unfinished daily instances from prior days **silently appear** on today's list until completed or triaged. Each spillover row shows an amber **age label** once it's 2+ days old (`2d old`, `5d old`, …). Each daily row has a `⋯` button revealing a popover menu with **Complete**, **Roll-over**, **Defer (1 day)**, **Drop**. Working rows have the same `⋯` but only **Complete** and **Drop** (no roll-over/defer — no daily reset). Row triage replaces the planned EOD review modal entirely.
- **Error handling:** 💬 toast on write failures.
- **Acceptance criteria:**
  - [ ] When a daily instance has `completed_at = NULL` and its `day` is < today, it appears on today's Dashboard in the Daily column.
  - [ ] For a spillover row where `today - day >= 2`, an amber `{N}d old` chip renders to the right of the title; for `day = yesterday`, no chip.
  - [ ] Clicking `⋯` on a daily row opens a popover with 4 items: Complete / Roll-over / Defer / Drop, each with an explanatory subtitle.
  - [ ] **Complete** sets `completed_at = now()` on the latest open instance.
  - [ ] **Roll-over** explicitly carries the instance to tomorrow — functionally the same as taking no action (silent spillover) but records an `action = 'rolled_over'` event in a `task_events` log for History visibility.
  - [ ] **Defer** marks `deferred_until = today + 1 day`; the row disappears from today's list and reappears on the specified day. Records `action = 'deferred'`.
  - [ ] **Drop** sets `dismissed_at = now()`; the row vanishes from the Dashboard and is tagged in History as "Dropped". Records `action = 'dropped'`.
  - [ ] The popover is keyboard-accessible (Tab into `⋯`, Enter to open; arrow-keys + Enter to choose; Esc to close).

### FR-005: Window-title counter

- **Description:** The browser `document.title` reads `tempo` when nothing is pending, and `(N) tempo` when N total pending items exist across Daily (today's open + spillover) and Working (active). In an installed PWA window the title drives the standalone-window title bar too.
- **Error handling:** None (cosmetic).
- **Acceptance criteria:**
  - [ ] On app load, the title is computed from `pending_daily + pending_working` and set once.
  - [ ] After any create / complete / dismiss / defer / drop action the title is recomputed and set.
  - [ ] When the sum is 0, the title is exactly `tempo` (no parentheses, no zero).
  - [ ] The title format for non-zero: `(N) tempo` — total count only, no split.

### FR-006: Global ⌘K palette

- **Description:** A modal command palette opened by `⌘K` (Cmd+K on macOS) or the search pill in the top bar. Searches across **notes** (title + body via FTS5), **daily task templates**, **working items**, and **history** (completed / dropped / deferred). Top section surfaces **Create** actions for the typed query: "Create daily task: '{query}'", "Create working item: '{query}'", "Create note titled: '{query}'". Keyboard-first navigation; Esc to close. Result sections: Create / Daily tasks / Working items / Notes / History.
- **Error handling:** None (no writes from the palette except Create actions, which show a 💬 toast on failure).
- **Acceptance criteria:**
  - [ ] `⌘K` (or clicking the search pill in the top bar) opens the palette centered, autofocused search input.
  - [ ] An empty query shows recent / pinned items (last 10 touched across all surfaces) — no Create rows.
  - [ ] A non-empty query shows three Create rows at the top (daily task, working item, note), followed by matching sections with section labels (`DAILY TASKS · N`, `NOTES · N`, `HISTORY · N`) each capped at 5 results.
  - [ ] ↑ / ↓ navigate the combined result list; ↵ activates the selected row.
  - [ ] Esc closes the palette without changes.
  - [ ] Clicking a result navigates to the relevant surface (Dashboard with the row scrolled/highlighted; Notes strip in editor mode for a note; History view filtered for a history result).
  - [ ] Create rows submit via `fetch` to the relevant form action; on success the palette closes and a 💬 toast confirms creation.

### FR-007: History view

- **Description:** A `/history` route listing completed, deferred, and dropped items grouped by day, newest first. Filter chips (All / Daily / Working / Dropped / Deferred). Date-range picker (default: last 30 days, with shortcuts: 7d, 30d, 90d, all-time). Each row shows a coloured status dot (green = completed, orange = deferred, grey = dropped), title, kind chip, optional secondary chip (`Deferred 1d`, `Dropped`), and time-of-action. No pagination in v1 (list virtualised if > 500 items; simple scroll).
- **Error handling:** 💬 toast on filter-apply failures (unlikely — local query).
- **Acceptance criteria:**
  - [ ] Route `/history` lists completed / deferred / dropped events, grouped by day.
  - [ ] Filter chips (All default) restrict the list to matching kinds.
  - [ ] Date-range picker applies to the query; the range summary renders in the picker control.
  - [ ] Rows are clickable: a completed daily row opens its template for editing; a completed working row opens the item's editor (expanded); a history-only entry with no live target shows a read-only detail.
  - [ ] Filter and date range persist in URL query params (`?filter=daily&range=30d`) for bookmarkability.

### FR-008: Settings

- **Description:** A `/settings` route accessed only via the cog icon in the top-right corner of the top nav. Two sections: **Appearance** (theme: Light / Dark / System, default System) and **About** (version, build SHA, build date, database path, GitHub link). No EOD hour — the EOD review modal was dropped in favour of row-level triage.
- **Error handling:** 💬 toast if theme save fails. None for About (read-only).
- **Acceptance criteria:**
  - [ ] Clicking the cog icon navigates to `/settings`.
  - [ ] The cog icon renders in an "active" state (soft accent fill) while on `/settings`; Dashboard/History tab underlines are hidden.
  - [ ] Theme is a 3-way segmented control; changing it immediately applies and persists to `localStorage` under key `tempo-theme`.
  - [ ] System theme respects `prefers-color-scheme`; switching the OS theme while on System updates immediately.
  - [ ] About section renders: version (from `package.json`), build SHA (short), build date, DB path (from `TEMPO_SQLITE_PATH` or default), and a live-rendered link to `github.com/nicolasiorio/tempo`.

### FR-009: PWA shell + manifest

- **Description:** `static/manifest.webmanifest` wired so Chrome (and Safari) can "Install as app" for a standalone window with its own dock icon. `start_url: /`, `display: standalone`, name `tempo`, `theme_color` matching the top-bar background, icons (stub) at 192 and 512. Document title drives the PWA window title (FR-005 counter visible there).
- **Error handling:** None (static asset).
- **Acceptance criteria:**
  - [ ] `static/manifest.webmanifest` exists and parses as valid JSON with fields: `name`, `short_name: "tempo"`, `start_url: "/"`, `display: "standalone"`, `theme_color`, `background_color`, `icons[]`.
  - [ ] `app.html` references the manifest via `<link rel="manifest" href="/manifest.webmanifest">`.
  - [ ] Chrome → "Install tempo" offers the install flow; after install, launching the app opens a standalone window with no browser chrome.
  - [ ] Icons at `static/icon-192.png` and `static/icon-512.png` exist as stub placeholders (music-note glyph on blue gradient, replaceable later).
  - [ ] No install-launcher.sh in v1 — `bun run dev` / `bun start` handle launch locally.

### FR-010: First-run empty states

- **Description:** When the database has zero daily templates, zero working items, and zero notes, the Dashboard shows a welcome banner at the top (auto-dismisses once any item exists) plus per-column empty states. Each column shows an **accent-bordered adder** ("Add your first daily routine…") plus a "TRY ONE OF THESE" row of 2–3 clickable suggestion chips that pre-fill the adder. No wizard, no forced walkthrough.
- **Error handling:** None (rendering-only).
- **Acceptance criteria:**
  - [ ] When `COUNT(task_templates) + COUNT(work_items) + COUNT(notes) = 0`, the welcome banner renders at the top of the Dashboard.
  - [ ] Each column's empty state is independently computed: Daily is "empty" iff no templates (archived or not); Working iff no active items; Notes iff no notes.
  - [ ] Suggestion chips, on click, focus the adder input and pre-fill its value (user still has to Enter to confirm).
  - [ ] Once any item of the relevant kind exists, that column's empty state is replaced by the populated view. Welcome banner hides as soon as any of the three counts becomes non-zero.
  - [ ] Empty-state hint text below each column references the locked behaviour ("every day / weekdays only", "Recently completed for 7 days", "Notes support full markdown").

---

## ⚙️ Non-Functional Requirements

- **Performance:** Dashboard first paint < 200ms on local SQLite (no network). ⌘K palette result updates < 50ms per keystroke (FTS5 + indexed queries). Day rollover job is a single transaction with batched inserts.
- **Security:** Server binds only to `127.0.0.1:3001`. `ORIGIN=http://localhost:3001` set in `start` script for SvelteKit CSRF. No auth (single-user local). No external network calls at runtime.
- **Accessibility:** Full keyboard navigation across adders, rows, `⋯` menus, palette. Visible focus rings (kit-db's pattern — inherited from `src/routes/layout.css`). Contrast tokens reused from kit-db's Apple palette. No Dynamic Type (web app; browser zoom substitutes).
- **Reliability:** SQLite is ACID. DB writes transactional via `bun:sqlite`. Day rollover is idempotent (unique constraint on `(template_id, day)` — retries cause no duplicates).
- **Browser target:** Safari and Chrome on macOS, current versions. No IE/Edge-legacy support.

---

## 📊 Data Requirements

### Storage

- **SQLite** file at `~/Developer/tempo-data/tempo.sqlite` by default. Override via `TEMPO_SQLITE_PATH` env var. Parent directory auto-created on boot.
- **Migrations** forward-only, `.sql` files in `src/lib/server/migrations/`, applied on boot via `runMigrations()` (kit-db pattern — split on `;` and run each statement via `db.run()` so errors propagate).

### Schema (v1 — `001_v1_init.sql`)

```sql
-- Daily routines (templates). One row per recurring task.
CREATE TABLE task_templates (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    title         TEXT    NOT NULL,
    schedule      TEXT    NOT NULL DEFAULT 'every_day',  -- 'every_day' | 'weekdays'
    time_estimate TEXT,                                    -- '15m' | '30m' | '1h' | 'longer' | NULL
    note          TEXT,
    created_at    TEXT    NOT NULL DEFAULT (datetime('now')),
    archived_at   TEXT
);
CREATE INDEX idx_task_templates_active ON task_templates(archived_at) WHERE archived_at IS NULL;

-- Instances — one row per (template, day). Materialised each morning.
CREATE TABLE task_instances (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    template_id     INTEGER NOT NULL REFERENCES task_templates(id) ON DELETE CASCADE,
    day             TEXT    NOT NULL,                     -- 'YYYY-MM-DD' (local)
    completed_at    TEXT,
    dismissed_at    TEXT,                                 -- set when Dropped
    deferred_until  TEXT,                                 -- set when Deferred; the day this will reappear
    UNIQUE (template_id, day)
);
CREATE INDEX idx_instances_day ON task_instances(day);
CREATE INDEX idx_instances_open ON task_instances(day, completed_at, dismissed_at);

-- Working items — multi-day persist-until-done.
CREATE TABLE work_items (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    title          TEXT    NOT NULL,
    time_estimate  TEXT,
    note           TEXT,
    completed_at   TEXT,
    dismissed_at   TEXT,
    created_at     TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at     TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_work_active ON work_items(completed_at, dismissed_at);

-- Notes — markdown, optionally linked to one task (either template OR work_item, XOR enforced in app layer).
CREATE TABLE notes (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    title             TEXT    NOT NULL,
    body              TEXT    NOT NULL DEFAULT '',
    task_template_id  INTEGER REFERENCES task_templates(id) ON DELETE SET NULL,
    work_item_id      INTEGER REFERENCES work_items(id) ON DELETE SET NULL,
    created_at        TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at        TEXT    NOT NULL DEFAULT (datetime('now')),
    CHECK (task_template_id IS NULL OR work_item_id IS NULL)
);
CREATE INDEX idx_notes_template ON notes(task_template_id) WHERE task_template_id IS NOT NULL;
CREATE INDEX idx_notes_work ON notes(work_item_id) WHERE work_item_id IS NOT NULL;

-- FTS5 shadow for notes search (title + body).
CREATE VIRTUAL TABLE notes_fts USING fts5(title, body, content='notes', content_rowid='id');
-- Triggers to keep notes_fts in sync on insert/update/delete (see migrations).

-- Event log — records every Roll-over / Defer / Drop action for History visibility.
-- (Completions are already timestamped on the row itself; no need to duplicate.)
CREATE TABLE task_events (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    kind           TEXT    NOT NULL,                      -- 'daily' | 'working'
    target_id      INTEGER NOT NULL,                      -- task_instances.id or work_items.id
    action         TEXT    NOT NULL,                      -- 'rolled_over' | 'deferred' | 'dropped'
    created_at     TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_task_events_time ON task_events(created_at);

-- Settings — single-row key/value store. theme lives in localStorage (client side); DB reserved for anything that needs persistence across devices later.
-- v1: empty table, reserved for future.
CREATE TABLE app_settings (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
```

### Derived queries

- **Today's Daily list:** all `task_instances` where `day = TODAY` AND `dismissed_at IS NULL` AND `(deferred_until IS NULL OR deferred_until <= TODAY)`, plus spillover = all instances where `day < TODAY` AND `completed_at IS NULL` AND `dismissed_at IS NULL` AND `(deferred_until IS NULL OR deferred_until <= TODAY)`.
- **Pending count (for window title):** `(count of the above that are incomplete) + (count of work_items where completed_at IS NULL AND dismissed_at IS NULL)`.
- **Notes FTS:** `SELECT notes.* FROM notes JOIN notes_fts ON notes.id = notes_fts.rowid WHERE notes_fts MATCH ? ORDER BY rank`.

### Day rollover

- A server-side job runs at boot (and on every request handler entry, cached to run at most once per local day) that checks: is today's set of instances materialised? If not, insert missing instances for each active template whose schedule matches today's weekday, using `INSERT OR IGNORE` on the `UNIQUE(template_id, day)` index. Zero-cost after the first call per day.

---

## 🔗 Dependencies

Inherit the kit-db dependency set with one addition:

| Package | Purpose | Notes |
|---|---|---|
| Bun `1.3.x` | Runtime + package manager | Kit-db match. |
| SvelteKit `2.x` + Svelte 5 | Framework | Kit-db match. |
| TypeScript `^5` | Language | Kit-db match. |
| Tailwind CSS `v4` | Styling | Kit-db match, same tokens. |
| `bun:sqlite` (built-in) | Database | No separate dep. |
| `@sveltejs/adapter-node` `5.x` | Production server | Kit-db match. |
| `lucide-svelte` `1.x` | Icons | Kit-db match, reuse curated icon registry. |
| **`marked` `^12.x`** | **Markdown → HTML renderer (new)** | ~20KB. MIT. Client-side only, applied to note body on view; editor stays plain textarea. |
| ESLint / Prettier | Linting | Kit-db match. |

All licences MIT / BSD / Apache — compatible with tempo's MIT licence.

---

## 🚫 Out of Scope (v1)

Explicitly deferred — add to `planning/BACKLOG.md` on first real demand:

- **EOD review modal** — replaced by row-level `⋯` triage. Never shipped.
- **Install-launcher script** — no persistent `launchctl`-managed background server in v1. Run via `bun run dev` or `bun start` in a terminal.
- **Native Swift / Tauri wrapper** — PWA is the delivery. Revisit only if PWA proves insufficient after a month of daily use.
- **Defer picker** — Defer is fixed 1-day in v1. No 3-day / next-week / custom-date pickers.
- **Custom weekday schedule** — only `every_day` and `weekdays` in v1. No Mon/Wed/Fri-style pickers.
- **Time-estimate sum / filter** — time estimates are display-only chips in v1. No "today's load" aggregation, no "quick wins" filter.
- **Per-column filter bars** — global ⌘K palette handles filtering.
- **Note attachments / images** — plain markdown body only in v1.
- **Multi-task note links** — one note → one task maximum.
- **Data import** — no folder-scan `.txt` importer. Paste-in only.
- **Data export** — no CSV export in v1. SQLite file itself is the backup.
- **Notifications / sound / badges** — this contradicts the anti-badgering thesis. Never.
- **Multiple users / profiles** — single-user local.
- **Sync / cloud** — explicitly local-only.
- **Bulk edit** — row-by-row only.
- **Template-to-template linking / dependencies** — tasks are independent.
- **Archive view for dropped items** — dropped items live in History with the "Dropped" chip; no separate archive surface.

---

## 🧭 Wireframes

High-fidelity SVG mockups at 1440×900 (matches the target resolution per UI preferences) live in `pipeline/mockups/`. Open each with `open pipeline/mockups/<name>.svg` or preview in a browser.

### Navigation structure

Top nav bar (56px) sits below the OS/PWA titlebar and contains:
- **Brand** (tempo logo + wordmark) — left
- **Tabs:** `Dashboard`, `History` — inline with brand, underline-on-active
- **⌘K search pill** — right-of-centre, `Search everything`
- **Cog icon (Settings)** — far right corner, active fill on `/settings`

### Screens

| Screen | Mockup | Requirements covered |
|---|---|---|
| **Dashboard** (populated) | `mockups/dashboard.svg` | FR-001, FR-002, FR-003, FR-004 (age label + `⋯`), FR-005 (window title), FR-006 (⌘K pill), FR-008 (cog) |
| **Dashboard** (first-run) | `mockups/dashboard-empty.svg` | FR-010, FR-001/002/003 empty shells, FR-005 (zero count → plain `tempo`) |
| **⌘K palette** | `mockups/palette.svg` | FR-006 |
| **History** | `mockups/history.svg` | FR-007 |
| **Settings** | `mockups/settings.svg` | FR-008 |
| **Row `⋯` menu** | `mockups/row-menu.svg` | FR-004 |

### UI elements and interactions

- **Adders** — dashed-border rounded inputs at the top of each column. Focus grows the border to solid accent. Enter submits, focus returns to the adder for rapid entry. Suggestion chips (empty state) populate the adder value on click.
- **Rows** — horizontal flex: checkbox · title · meta chips (age, note-count, time-estimate) · `⋯` button. Click the title area to expand inline for editing; click again to collapse.
- **`⋯` popover** — 240×220 popover anchored below the button. Options: Complete (primary), Roll-over, Defer (1 day), Drop (destructive red). Keyboard nav with arrow keys; Enter activates; Esc closes.
- **Notes cards** — 4-up grid in the Notes strip. Click replaces the strip with the editor (full-width title input + markdown textarea + Preview toggle + Close).
- **⌘K palette** — 680px centred modal overlay with dimmed backdrop. Sections: Create (top 3), Daily tasks, Working items, Notes, History. Keyboard-first.
- **Theme toggle** — segmented control (Light / Dark / System). Instant apply, persists to `localStorage`.

### Accessibility notes

- All interactive elements keyboard-reachable in logical tab order.
- `⋯` popover: Tab to button, Enter opens, Arrow keys navigate, Enter selects, Esc closes. Role `menu` + `menuitem`.
- ⌘K palette: modal role, focus trap, Esc closes, results list has `role="listbox"` with `aria-activedescendant`.
- Rows use semantic `<ul>` / `<li>` with checkbox inputs — screen readers read status naturally.
- Contrast tokens inherit kit-db's Apple palette, meets WCAG AA in both light and dark.

---

## 📋 Traceability Matrix

Every requirement maps to at least one screen and at least one data element.

| FR | Screen(s) | Data |
|---|---|---|
| FR-001 Daily list | dashboard, dashboard-empty, row-menu | `task_templates`, `task_instances` |
| FR-002 Working list | dashboard, dashboard-empty | `work_items`, `notes.work_item_id` |
| FR-003 Notes | dashboard, dashboard-empty | `notes`, `notes_fts` |
| FR-004 Spillover + row triage | dashboard (age label), row-menu | `task_instances.dismissed_at / deferred_until`, `task_events` |
| FR-005 Window-title counter | all (titlebar) | derived count (no storage) |
| FR-006 ⌘K palette | palette | All tables (read-only) + Create actions |
| FR-007 History view | history | `task_instances`, `work_items`, `task_events` |
| FR-008 Settings | settings | `localStorage` (theme), `package.json` (version), env (DB path) |
| FR-009 PWA shell | all (installed window) | `static/manifest.webmanifest` |
| FR-010 First-run empty states | dashboard-empty | counts over all 3 tables |

---

## 🔒 Locked Decisions (carried from pre-spec Q&A, 2026-04-22)

Captured in `CLAUDE.md` under "Locked Decisions"; mirrored here for spec consumers:

- **Daily schedule model:** `every_day` or `weekdays` only (no custom picker).
- **End-of-day triage:** no modal; row `⋯` menu only.
- **Window-title counter:** single total, `tempo` when zero, `(N) tempo` otherwise.
- **Notes ↔ tasks:** many notes per task; one task per note.
- **Two lists, two tables:** `task_templates` + `task_instances` for daily; `work_items` for working.
- **Working completion:** 7-day "Recently completed" collapsed section.
- **History:** kept forever.
- **Icons:** stub placeholders in v1.
- **Delivery:** PWA only.
- **Launcher:** dropped from v1.
- **Dashboard layout:** Daily + Working top row (2 columns), Notes strip below full-width.
- **First-run:** inline empty-state prompts per column.
- **Note editor:** markdown via `marked`; click-to-open replaces the Notes strip inline.
- **Adding items:** inline adder + ⌘K Create action. No create-modal.
- **Search:** global ⌘K palette.
- **Time estimates:** display-only chips. No sum, no filter in v1.
- **Navigation:** top-bar tabs (Dashboard / History); cog in top-right corner for Settings.
