# 🎵 tempo

Personal workspace tool — daily routines, working items, and notes. One dashboard for the rhythm of the working day.

---

## 🎯 Project Meta

- **Type:** Local-only web app (PWA), personal-use
- **Audience:** Single user (Nicolas)
- **Runs on:** `http://127.0.0.1:3001`, installed as a PWA via Chrome/Safari "Install as app"
- **Local-only:** `false` — GitHub + Forgejo (dual-push planned; Forgejo to be added when mini is reachable)
- **License:** MIT

## 🛠 Stack (planned — cloned from kit-db)

| Layer | Choice | Version |
|---|---|---|
| Runtime | Bun | 1.3.x |
| Framework | SvelteKit | 2.x (Svelte 5) |
| Language | TypeScript | ^5 |
| Styling | Tailwind CSS | v4 |
| Database | SQLite (via `bun:sqlite`) | built-in |
| Linting | ESLint + Prettier | scaffolded defaults |
| Adapter | `@sveltejs/adapter-node` | 5.x |
| Icons | `lucide-svelte` | 1.x |

Nothing is installed yet — stack is committed to in `/plan`, installed in `/build`.

## 🏗 Architecture (planned)

- **Pattern:** MV (simple) — matches kit-db.
- **DB location:** default `~/Developer/tempo-data/tempo.sqlite` (outside the project so code folder can be drag-replaced without touching data). Override with `TEMPO_SQLITE_PATH`.
- **Port:** `:3001` (kit-db owns `:3000`, this prevents collision).
- **CSRF / origin:** production `start` script sets `ORIGIN=http://localhost:3001`.
- **PWA shell:** `static/manifest.webmanifest` configured so Chrome/Safari "Install as app" gives a standalone window with its own dock icon. `document.title` drives window title → pending-count badge lives there.

## 🎨 Design Language

Same house style as kit-db: apple.com aesthetic — system font stack, Lucide icons, neutral grays + Apple blue accent, generous whitespace.

- **No emojis in UI strings** — reserved for docs/chat per NoriCo global standards.
- **Target resolution:** 1440×900 (per MEMORY.md UI preferences).

## 📋 Pipeline

Standard 5-step NoriCo pipeline. Artifacts in `pipeline/`.

1. `/spec` — requirements + wireframes
2. `/plan` — technical plan + SE review + sprint roadmap
3. `/build` — implementation with checkpoints
4. `/review` — code review + fix + test + security
5. `/ship` — final commit + PR

## 🗂️ Backlog

See `planning/BACKLOG.md` (created on first deferred idea).

## 📊 Current Status

- **Branch:** `main`
- **State:** Scaffolded 2026-04-22. Repo + docs only — no SvelteKit code yet.
- **Next step:** `/spec` pass to convert the shelved plan (`~/.claude/plans/i-want-to-spin-structured-crystal.md`) into requirements + wireframes.

## 📐 Locked Decisions (from pre-spec Q&A, 2026-04-22)

Carried forward from the shelved plan so they survive future sessions:

- **Daily task schedule:** every-day OR weekdays-only toggle. No custom weekday picker in v1.
- **EOD review modal:** single configurable hour in Settings; fires on app open/refresh after that hour.
- **Window-title counter:** `tempo` when empty, `(N) tempo` when pending.
- **Notes ↔ tasks:** a note belongs to at most one task; a task can have many notes.
- **Two lists, two tables:** `task_templates` + `task_instances` (daily) and `work_items` (working). No single `kind` column.
- **History:** kept forever (UI filter for decluttering).
- **Icons:** stubs in v1, real branding later.
- **Delivery:** PWA (Level 1). No native Swift wrapper. Revisit only if something concrete is missing after a month of daily use.

## 🐛 Lessons Learned

None yet — inherits kit-db's lessons (migration runner, bunfig.toml, ORIGIN for CSRF, etc.) when the matching code lands.
