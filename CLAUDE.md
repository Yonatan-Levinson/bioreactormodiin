# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Single-page scheduling dashboard for Novella's bioreactor lab in Modi'in. Live at
https://yonatan-levinson.github.io/bioreactormodiin/. Everything — HTML, CSS, and JS —
lives in `index.html`. No build step, no dependencies, no package.json. GitHub Pages
serves the file directly from `main`; pushing to `main` updates the live site in
about a minute.

**Keep it that way**: do not introduce a bundler, a framework, or a build pipeline.
Do not split the file into multiple files. No CDN dependencies — nothing that
requires network beyond the cloud-sync endpoint.

## Running / testing

There is no build, lint, or test tooling. To work on this locally, just open
`index.html` directly in a browser (or serve the directory with any static file
server) and reload after edits. Verify changes by exercising the UI manually —
add/drag/resize a run, toggle tasks, switch views, and confirm cloud sync still
round-trips (see below).

## Data & sync architecture

All state lives in one in-memory object: `state = {stations, runs, tasks}` (`index.html:234`).

- **Source of truth**: a Google Sheet reached through an Apps Script web app
  (`CLOUD_URL`, `index.html:230`). `cloudPull()` polls it every 5s and merges
  incoming data by comparing a JSON hash of `{stations, runs, tasks}`; `cloudPush()`
  debounces (400ms) and POSTs the full state on every local change.
- **Local cache**: `localStorage` (`KEY = "novella_bioreactor_v2"`) is written on
  every `save()` and used to hydrate state before the first cloud pull completes.
  It is a cache, not a store — cloud sync is normally ON for everyone on the shared
  link, and localStorage-only state is invisible to the rest of the team.
- **Rule**: any new feature that holds team-visible data must round-trip through
  `state` and get picked up by `save()`/`cloudPush()`/`cloudPull()`. A feature that
  writes only to localStorage or a separate key is a bug, not a feature.

## Domain model

- **Runs** (`state.runs`): multi-day bioreactor runs, each tagged with a `station`
  (from `state.stations`, user-managed) and rendered as a bar spanning `start`–`end`
  (ISO date strings). The operative unit displayed on a run is **day-of-run**
  (`d0`, `d1`, `d2`…, computed as offset from `start`), not the calendar date —
  this is how the lab actually thinks about a run in progress. `deps` lets a run
  depend on another finishing first; `cascade()` (`index.html:421`) pushes
  dependents forward whenever a predecessor's dates change.
- **Tasks** (`state.tasks`): single-day items, optionally tied to a `station`
  (empty = "General" lane) and optionally to a `runId`. Independently toggleable
  on/off the calendar via the Tasks button.
- **Stations** (`state.stations`): user-managed list of reactor lines
  (defaults: Dasgip, MF1, MF2, MF3).

## Rendering

Two view modes share the same `state` but render through separate paths, toggled
by `render()` (`index.html:418`):

- `renderMonth()` (`index.html:289`) — grid of week rows; run bars overlaid per
  week with lane-packing to avoid overlap, tasks shown inline per day cell (max 3,
  "+N more" beyond that). Driven by `monthSpan` (0 = full month, else N weeks) and
  `monthRolling` (wheel-scroll rolling window).
- `renderWeek()` (`index.html:364`) — one row per station (plus a "General" row for
  station-less tasks), run bars positioned absolutely by day column, with a
  dependency-arrow SVG overlay (`drawDeps()`, `index.html:416`).

Drag/resize/reschedule interactions (`index.html:423-429`) are pointer-event based
and only active in week view: dragging a bar moves or resizes a run (snapping to
day columns via `colW`), dragging a task moves its date, and `cascade()` runs after
any change that could affect dependents.

Modals (`openRunModal`, `openTaskModal`, `openStationModal`, `openCloudModal`) build
their markup by direct `innerHTML` string assembly and wire handlers imperatively —
follow that pattern rather than introducing a templating approach.

## Styling

All colors are CSS custom properties defined once in `:root` (`index.html:8-15`).
Use them; don't hardcode hex values in new markup or styles. The run/task palette
itself (`PALETTE`, `index.html:231`) is a separate fixed set of six named swatch
colors used for user-assigned coloring, distinct from the `:root` theme variables.

## Who uses it and how

- **Nadav** (Bioreactor Engineer) opens it daily to see what he's running, and adds
  his own runs and tasks. He's the primary daily user — a change that makes his
  morning check slower or more confusing is a regression.
- **Yoni** (Head of Bioprocess) plans the run sequence and owns this codebase.
- Runs map to named upstream projects: Baseline, DOE, Media, Supply, Wash, Incyte.

The schedule (calendar) is the load-bearing feature. After touching shared-state
code, re-verify it still works before moving on.

## In progress: Project management tab

A new tab is being added: a reporting surface where Yoni and Or both enter their
projects, so Shimrit (CSO) has one place to check status instead of asking each of
them separately. Each project is a card (click to open) with a consistent template:
goals, experimental plan, experiments list, timeline, milestones, owning department
(Process Development / Biology — naming TBD). Anyone on the shared link can edit,
same as the rest of the app, so it must persist through the same cloud-sync path.

Design constraint: this is a *reporting* surface for people who don't open the
calendar and read cold for about two minutes — density is a failure mode here, not
a virtue. Don't reuse the calendar's dense-grid patterns for this tab.
