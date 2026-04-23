# Study Planner — Claude Context

This file gives Claude full context about this project so it can make changes accurately without needing re-explanation each session.

---

## What this project is

A **single-file Progressive Web App (PWA)** — a universal learning schedule planner. It has no backend, no framework, no build step. Everything runs in the browser from one `index.html` file.

It is deployed on **GitHub Pages** and works on Windows, iPhone, and Android. Users can install it to their home screen like a native app.

---

## Repository structure

```
/
├── index.html          ← The entire app. All HTML, CSS, and JS in one file.
├── manifest.json       ← PWA manifest (app name, theme colour, display mode)
├── sample_plan.json    ← Example plan file showing the correct JSON schema
└── CLAUDE.md           ← This file
```

There is no `package.json`, no build pipeline, no node_modules. Do not add a framework or build step without being explicitly asked.

---

## How the app works

### Core concept
The app imports **plan.json files** — structured learning plans that Claude (in chat) generates for any topic. Each plan contains modules with hours, tags, and resource links. The app turns these into a dynamic calendar schedule.

### Data flow
1. User imports a `plan.json` → app stores it in `localStorage`
2. User sets start date, weekday hours/day, weekend hours/day
3. App distributes modules across days based on available hours
4. Skipped/rescheduled days push modules forward automatically
5. User taps a day → modal popup shows module detail, resource links, status picker, notes

### Schedule building logic
The core function is `buildSch()` in `index.html`. It:
- Iterates day by day from `startDate`
- Skips days with status `skip` or `reschedule` (modules carry forward)
- Fills each day with modules until available hours are consumed
- Tracks `carryHrs` when a module spans multiple days
- Returns an array of day objects: `{ dateStr, status, modules[], hrs, isWeekend, notes }`

### State management
All state lives in `localStorage` under the key `studyplanner_v2`. Structure:

```json
{
  "plans": {
    "plan_1234567890": {
      "title": "...",
      "modules": [...],
      "total_hours": 44,
      "state": {
        "startDate": "2025-04-23",
        "wdHrs": 2,
        "weHrs": 4,
        "dayStates": {
          "2025-04-25": { "status": "done", "notes": "Covered X and Y" },
          "2025-04-26": { "status": "skip", "notes": "" }
        }
      }
    }
  },
  "active": "plan_1234567890"
}
```

---

## plan.json schema

This is the format Claude (in chat) generates when a user asks for a new study plan. The app imports this file.

```json
{
  "title": "Short plan name",
  "description": "One-line description",
  "version": "1.0",
  "total_hours": 44,
  "modules": [
    {
      "id": 1,
      "title": "Full module title",
      "short": "Short label for calendar cell (≤18 chars)",
      "detail": "What this module covers — shown in modal and list view",
      "hours": 3,
      "tag": "light",
      "resources": [
        {
          "title": "Resource display name",
          "url": "https://...",
          "type": "docs"
        }
      ]
    }
  ]
}
```

**Tag values:** `light` | `moderate` | `heavy` | `buffer`

**Resource type values:** `video` | `article` | `docs` | `course` | `tool` | `practice` | `other`

The app normalises `hours` and `hrs` fields — both work. `total_hours` is auto-calculated if missing.

---

## Design system

The app uses a **dark theme** with CSS custom properties defined in `:root`. Do not hardcode colours — always use the variables.

### Colour palette

| Variable | Value | Use |
|---|---|---|
| `--bg` | `#0e0e11` | Page background |
| `--surface` | `#16161a` | Cards, panels |
| `--surface2` | `#1e1e24` | Inputs, inner cards |
| `--border` | `#2a2a33` | Default borders |
| `--border2` | `#3a3a46` | Hover/focus borders |
| `--text` | `#e8e8f0` | Primary text |
| `--muted` | `#7a7a8e` | Secondary text, labels |
| `--accent` | `#7c6dfa` | Purple — primary accent |
| `--accent2` | `#a78bfa` | Purple — lighter accent |
| `--green` | `#34d399` | Completed status |
| `--amber` | `#fbbf24` | Rescheduled / warning |
| `--red` | `#f87171` | Skipped / error |
| `--teal` | `#22d3ee` | Weekday study days |

Each colour also has `--colour-bg` (10% opacity fill) and `--colour-b` (30% opacity border) variants, e.g. `--green-bg`, `--green-b`.

### Typography

- **`--sans`**: `'Syne', sans-serif` — headings, UI labels
- **`--mono`**: `'DM Mono', monospace` — dates, stats, badges, code-like UI elements
- Both loaded from Google Fonts in the `<head>`

### Status colours

| Status | Colour |
|---|---|
| `planned` | accent (purple) |
| `done` | green |
| `skip` | red |
| `reschedule` | amber |

Status colours are applied as CSS classes: `.st-done`, `.st-skip`, `.st-reschedule` on calendar cells and list items.

---

## Key functions reference

| Function | Location | What it does |
|---|---|---|
| `buildSch()` | `<script>` | Builds full schedule array from plan + state |
| `renderAll()` | `<script>` | Top-level render — calls all sub-renders |
| `renderCal(sch)` | `<script>` | Renders monthly calendar grid |
| `renderList(sch)` | `<script>` | Renders day-by-day list view |
| `renderSummary(sch, plan)` | `<script>` | Renders the 4 stat cards |
| `renderProgress(sch)` | `<script>` | Renders progress bar and stats |
| `openModal(ds, dayData)` | `<script>` | Opens day detail popup |
| `saveDay()` | `<script>` | Saves status + notes to localStorage |
| `readFile(file)` | `<script>` | Parses imported plan.json |
| `exportSch()` | `<script>` | Exports schedule as JSON download |
| `save()` / `load()` | `<script>` | localStorage persistence |

---

## Modal popup behaviour

Tapping any study day opens a bottom sheet modal (`#dayOverlay`) that contains:

1. **Date + module title** in the header
2. **Status picker** — 4 buttons: Planned / Completed / Skip / Reschedule
3. **Module cards** — one per module scheduled that day, each showing title, detail, hours, and resource links
4. **Notes textarea** — free text, persisted per day
5. **Save & close** button — writes to `localStorage` and re-renders

Status changes drive the schedule: `skip` and `reschedule` cause modules to push forward to the next available day. `done` locks the day and shows green. `planned` is the default.

---

## Multi-plan support

The app supports multiple plans simultaneously. Plans are stored as a dictionary in `APP.plans` keyed by `plan_<timestamp>`. The active plan is tracked in `APP.active`.

The plan bar at the top shows a chip per plan. Clicking a chip switches the active plan. Each plan has its own independent `state` (start date, hours, day statuses, notes).

The built-in Vibe Coding plan (`plan_builtin`) is pre-loaded on first launch if no plans exist.

---

## Deployment

**Platform:** GitHub Pages — free static hosting.

**URL pattern:** `https://<username>.github.io/<repo-name>`

**All three files must be in the repo root:**
- `index.html` — the app
- `manifest.json` — PWA metadata
- `sample_plan.json` — example plan for reference

**To deploy a change:** Edit `index.html` in VS Code → commit → push to `main`. GitHub Pages auto-deploys within 60 seconds. No build step needed.

**PWA install:** On iPhone, open in Safari → Share → Add to Home Screen. On Android, open in Chrome → three-dot menu → Add to Home Screen.

---

## What Claude should NOT do

- Do not add a JavaScript framework (React, Vue, Angular, etc.)
- Do not add a package.json or build pipeline
- Do not add a backend or server-side code
- Do not split the app into multiple JS/CSS files (keep it single-file)
- Do not use `localStorage` workarounds — it works fine on GitHub Pages
- Do not change the colour scheme or typography without being asked
- Do not use `position: fixed` for new UI elements — use in-flow layout
- Do not add external JS libraries unless explicitly asked

---

## Common change requests and how to handle them

### Adding a new feature to the modal
Add HTML inside `.modal-body` in `index.html`. Wire up JS in the `openModal()` and `saveDay()` functions. Persist any new fields in `plan.state.dayStates[ds]`.

### Adding a new status type
1. Add a new `.sbtn` button in the status grid HTML
2. Add a `sel-<status>` CSS class following the existing pattern
3. Add the status to the `renderStBtns()` function
4. Add a CSS class `.st-<status>` for calendar cells and list items
5. Add a dot class `.dot-<status>` for calendar cell dots
6. Update `buildSch()` if the new status should push modules forward

### Changing schedule logic
Edit `buildSch()` only. All rendering functions consume its output — they don't need to change if the day object shape stays the same.

### Adding a new field to plan.json
1. Update the schema section in this file
2. Handle the field in `readFile()` (normalise if needed)
3. Render it where appropriate (modal, list, or calendar)

### Supporting a new resource type
Add the type to `RES_ICON` object in the script. Add it to the schema table above.

---

## Owner context

The owner of this project is a software developer with 7 years of experience currently studying the **Vibe Coding Competency — Level 1 (Beginner to Fundamental)** course. The course has 8 modules covering AI tools, prompt engineering, code review, security, professional workflow, and testing AI-generated code.

The planner was built to track this course and has been extended to be a universal tool for any learning plan. The owner uses Claude (chat) to generate `plan.json` files for new topics and imports them into this app.

The owner works primarily on **Windows**, deploys via **GitHub Pages**, and accesses the app on Windows, iPhone, and Android.
