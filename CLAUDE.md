# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A family of single-page scheduling dashboards for Novella's labs, all served from
one GitHub Pages site (https://yonatan-levinson.github.io/bioreactormodiin/):

- `/index.html` — **Modi'in** bioreactors (the original)
- `/chemo/index.html`, `/cpi/index.html`, `/ecl/index.html` — bioreactor dashboards
  for those sites, structurally identical to Modi'in
- `/biology/index.html` — flask experiments (no station/platform concept; "Run" is
  relabeled "Experiment" throughout)

All five have a **Projects** tab — see "Domain model" below for why it's shared
across every site rather than per-site data.

Each is a fully standalone file — HTML, CSS, and JS all in one, no build step, no
dependencies, no package.json. Pushing to `main` updates all of them within about
a minute. See "Multi-site architecture" below before touching more than one.

**Keep it that way**: do not introduce a bundler, a framework, or a build pipeline.
Don't split any single dashboard's HTML/CSS/JS into separate files. No CDN
dependencies — nothing that requires network beyond each site's own cloud-sync
endpoint.

## Running / testing

There is no build, lint, or test tooling. Open the relevant `index.html` directly
in a browser (or serve the directory with any static file server) and reload after
edits. Verify changes by exercising the UI manually — add/drag/resize a run, toggle
tasks, switch views, and confirm cloud sync still round-trips (see below).

## Data & sync architecture

All state for a given dashboard lives in one in-memory object, `state =
{stations, runs, tasks, projects}`. Runs/tasks/stations are **per-site**;
`projects` is **shared across all five sites** — this means each dashboard
runs *two independent sync channels* against *two different Sheets*:

- **Schedule channel** (`stations`/`runs`/`tasks`) — targets `CLOUD_URL`, which
  is each site's *own* Sheet. Starts blank on every site copy except Modi'in;
  connecting cloud sync (via "⋯ → Connect cloud sync…") is a one-time
  per-site, per-browser action documented in `README.md`. `cloudPull()` polls
  every 5s, comparing a JSON hash of `{stations,runs,tasks}`; `cloudPush()`
  debounces (400ms) and POSTs `{stations,runs,tasks}` (no `projects`) on every
  local schedule change.
- **Projects channel** (`projects`) — targets `PROJECTS_CLOUD_URL`, a constant
  hardcoded to **Modi'in's** Sheet URL in all five files (on the Modi'in file
  itself, this is the same URL as `CLOUD_URL`, so no separate channel is
  needed there — `cloudPull`/`cloudPush` already carry `projects` alongside
  its own schedule). On the four site copies, `projectsPull()` polls
  Modi'in's Sheet every 5s independent of whether the site's own `CLOUD_URL`
  is connected; `projectsPush()` (called via `saveProjects()`, not `save()`)
  does a read-modify-write — GET the current remote blob, overwrite only
  `projects`, POST the whole thing back — specifically so a project edit made
  from Chemo can never clobber Modi'in's own `stations`/`runs`/`tasks` (the
  Apps Script backend replaces the entire Sheet cell on every POST; see
  `README.md`).
- **Local cache**: `localStorage`, under a **site-specific key** (`KEY`/`URLKEY`,
  e.g. `novella_bioreactor_chemo_v1`/`novella_cloud_url_chemo`) — this matters
  because all dashboards share one GitHub Pages origin, so a generic key would
  collide across sites in the same browser. It's a cache, not a store.
- **Rule**: any new feature that holds team-visible schedule data must round-trip
  through `save()`/`cloudPush()`/`cloudPull()`; anything belonging to the shared
  Projects tab must go through `saveProjects()`/`projectsPush()`/`projectsPull()`
  instead. Mixing the two channels risks the site's own schedule data leaking
  into Modi'in's Sheet, or vice versa.

## Domain model

- **Runs** (`state.runs`, called "Experiments" in the Biology UI): multi-day runs,
  each tagged with a `station` (from `state.stations`) and rendered as a bar
  spanning `start`–`end` (ISO date strings). The operative unit displayed is
  **day-of-run** (`d0`, `d1`, `d2`…, offset from `start`), not the calendar date.
  `deps` lets a run depend on another finishing first; `cascade()` pushes
  dependents forward whenever a predecessor's dates change.
- **Tasks** (`state.tasks`): single-day items, optionally tied to a `station`
  (empty = "General" lane) and optionally to a `runId`. Independently toggleable
  on/off the calendar via the Tasks button.
- **Stations** (`state.stations`): user-managed list of platforms/lines. On
  Modi'in: Dasgip, MF1–3 (real names, user-editable via "Manage stations"). On
  Chemo/CPI/ECL: seeded with one placeholder ("Line 1") for the user to rename —
  station *names* are deliberately not hardcoded per site, only the dashboard
  instance is. On Biology there is no station UI at all (see below).
- **Projects** (`state.projects`, **shared across all 5 sites**): a reporting
  tab, separate from the calendar, meant as a single cross-site source of truth
  (originally for Yoni/Or's project status, read by Shimrit). It's visible and
  editable from every dashboard, but always reads/writes Modi'in's Sheet
  specifically (see "Data & sync architecture" above) — so there's one project
  list regardless of which site you opened it from, not five fragmented ones.
  Each project has `name`, `owner`, `dept`, `goals`, `plan`, `milestones` (flat
  list), and `workstreams`: an array of `{title, description, rows}` where every
  workstream shares one project-level column definition (`expCols`/
  `expColWidths`, reorderable and resizable) and each row is `{cells: [...],
  start, end}`. The per-row `start`/`end` (not a project-level date range) drive
  a Gantt strip grouped by workstream (`renderGantt()`), computed fresh from
  whatever rows have both dates set.

## Rendering

Two view modes share `state` but render through separate paths, toggled by
`render()`:

- `renderMonth()` — grid of week rows; run bars overlaid per week with
  lane-packing to avoid overlap, tasks shown inline per day cell (max 3, "+N
  more" beyond that). Driven by `monthSpan` (0 = full month, else N weeks) and
  `monthRolling` (wheel-scroll rolling window).
- `renderWeek()` — one row per station (plus a "General" row for station-less
  tasks), run bars positioned absolutely by day column, with a dependency-arrow
  SVG overlay (`drawDeps()`).

Drag/resize/reschedule interactions are pointer-event based and only active in
week view: dragging a bar moves or resizes a run (snapping to day columns via
`colW`), dragging a task moves its date, and `cascade()` runs after any change
that could affect dependents.

Modals build their markup by direct `innerHTML` string assembly and wire handlers
imperatively — follow that pattern rather than introducing a templating approach.
The Projects modal uses a larger, resizable modal variant (`.modal-lg`/
`.scrim-lg`) — other modals stay at the default small size; `closeModal()`
resets the class so it doesn't leak between modal types.

## Styling

All colors are CSS custom properties defined once in `:root`. Use them; don't
hardcode hex values in new markup or styles. The run/task palette itself
(`PALETTE`) is a separate fixed set of six named swatch colors used for
user-assigned coloring, distinct from the `:root` theme variables.

Free-text inputs/textareas (titles, notes, goals, experiment cells, etc.) carry
`dir="auto"` so Hebrew/mixed-direction content aligns itself automatically —
add this to any new free-text field.

## Who uses it and how

- **Nadav** (Bioreactor Engineer) opens the Modi'in dashboard daily to see what
  he's running, and adds his own runs and tasks. He's the primary daily user on
  that site — a change that makes his morning check slower or more confusing is
  a regression.
- **Yoni** (Head of Bioprocess) plans run sequencing and owns this codebase.
- Modi'in runs map to named upstream projects: Baseline, DOE, Media, Supply,
  Wash, Incyte.

The schedule (calendar) is the load-bearing feature on every site. After
touching shared-state code, re-verify it still works before moving on.

## Multi-site architecture

`/chemo`, `/cpi`, `/ecl`, and `/biology` are **copies**, not includes — each
`index.html` is fully self-contained, per the single-file/no-build-step rule.
This means a fix or feature that should apply everywhere (e.g. a calendar bug
fix) currently has to be **manually propagated to all 5 files**. There is no
shared module. When making a cross-cutting change:

1. Make and verify it on the Modi'in root `index.html` first.
2. Port the same diff to `chemo/`, `cpi/`, `ecl/`, and (adapting for the
   Run→Experiment relabeling and missing station UI) `biology/`.
3. If the change touches the Projects tab specifically, remember its sync
   plumbing differs on the site copies (`PROJECTS_CLOUD_URL`/`saveProjects()`
   instead of `CLOUD_URL`/`save()`) — see "Data & sync architecture" above.
   The modal markup/rendering code itself (`openProjectModal`, `renderGantt`,
   etc.) is identical across all 5 files and can be ported verbatim; only the
   save/delete handlers' final call (`saveProjects()` vs `save()`) differs.

Biology's differences from the Chemo/CPI/ECL template are deliberate and
minimal-risk: `state.stations` still exists internally with exactly one
implicit entry (`{id:"exp",name:"Experiments"}`), but the Station `<select>`
in the run/task modals and the "Manage stations" menu item are removed, and new
runs/tasks default to that one id — so the calendar naturally renders as one
ungrouped lane without touching `renderWeek()`/`renderMonth()`'s per-station
loop structure. If Biology ever needs multiple real lanes, that assumption
would need revisiting.

Each site has a violet site-switcher button at the top-left of the header
(fixed position, right after the logo, so it doesn't shift around as other
header controls wrap/hide) with a dropdown linking to the other four
dashboards by relative path.
