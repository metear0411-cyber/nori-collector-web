# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, no-build web tool distributed to 노리주간보호센터 staff so each employee can self-report
their own job tasks ("직무기술서" = job description). Staff pick tasks from a preset list for their job
title, optionally add custom tasks, then submit. The center director later assigns importance/ownership —
staff only select what they actually do.

There is no server-side code in this repo, no package manager, no build step, and no test suite. The
entire application — markup, CSS, data, and JS — lives in `index.html` (~440 lines, one very long line
containing the embedded JSON task catalog).

## Running / deploying

- **Local preview**: just open `index.html` in a browser (or serve it statically, e.g. `python3 -m http.server`).
- **Distribution**: this file is handed to staff as a static page (no auth) — treat it as public-facing;
  `<meta name="robots" content="noindex, nofollow">` is already set to keep it out of search results.
- **No build/lint/test commands exist** — there is no `package.json`, `Makefile`, or CI config in this repo.
  Editing means editing `index.html` directly.
- **Deploy = commit + push to `origin/main`** (`git@github.com:metear0411-cyber/nori-collector-web.git`),
  then re-publish/re-copy the static file wherever it's hosted for staff.

## Architecture (all inside `index.html`)

**Flow:** start screen (name + job pick) → two-column task picker (pool of preset tasks | "내 직무기술서"
list of picked tasks) → preview/print view → save (download JSON + optional POST to Google Sheets).

- **`PRESET` (line ~214)** — a single giant inline JSON object: `PRESET.jobs[jobName].tasks[]`, one entry
  per job title (`간호사`, `사무원`, `사회복지사`, `시설장`). Each task has
  `{id, task, cycle, area, systems, desc, aliases, publish, status, note}`.
  - `cycle` is one of `daily/weekly/monthly/quarterly/semiannual/yearly/adhoc` (labels in `CYCLES`/`CL`).
  - `systems` references the external tools a task touches (케어포, 사통망, 다해솔루션, 네이버웍스, 밴드,
    인스타, 블로그, 공단, 건강보험EDI, 푸른씨앗, 희망이음) — shown as tags in the UI.
  - `status: "active"` vs `"deprecated_candidate"` (deprecated-candidate styling exists in CSS via `.item.dep`
    / `.t-dep`, though no tasks currently use that status — it was pruned in an earlier commit).
  - `publish: true` marks tasks flagged for public-facing display (★공시 tag).
  - When editing this data, **preserve valid JSON on that one line** — it's easiest to extract, edit, and
    reinsert with a script (e.g. `python3 -c "import json,re; ..."`) rather than hand-editing inline, since
    the line is 40k+ characters.

- **App state** (`state = {staff, job, picked, custom}`) is persisted to `localStorage` under key
  `nori-jd-collector` (`persist()`), so a staff member can resume a partial submission (`resumeBox` on the
  start screen offers to continue a saved session).

- **Rendering**: `renderPool()` draws the filterable/searchable task list (filtered by `filterCycle` and
  `searchQ`, grouped by cycle); `renderMine()` draws the picked list grouped by cycle, supports drag-and-drop
  (`mineList.ondrop`) and manual custom-task entry (`addCustom()`).

- **Export path** (`saveBtn` click handler near line 376–392):
  1. Always downloads a JSON file named `직무기술서_{staff}_{date}.json` (`buildExport()` shape:
     `{version, staff, job, savedAt, selected[], custom[]}`) — this is the backup/fallback.
  2. If `SHEET_ENDPOINT` (a Google Apps Script web-app URL, hardcoded near the bottom of the script) is set,
     also POSTs the same JSON there with `mode: "no-cors"` (fire-and-forget — success/failure is inferred
     from whether the fetch promise resolves, since no-cors responses are opaque) for centralized collection
     into a Google Sheet.
  - `exportExcel()` provides an alternate `.xls`-compatible HTML-table export, and `showPreview()` renders a
    printable HTML summary (also used by the browser print dialog via the `@media print` CSS rules that hide
    everything except `#preview`).

## Things to know before editing

- No frameworks/libraries — vanilla DOM APIs only (`document.createElement`, `.onclick`, etc.).
- Korean-language UI/data throughout; keep new strings in Korean and consistent with existing tone
  (staff-facing, casual-but-respectful "-습니다" register).
- `reduced-motion` is explicitly respected (see the `@media (prefers-reduced-motion: reduce)` block) — keep
  any new transitions/animations gated the same way.
- Mobile layout collapses to a single column with a fixed bottom bar (`#mobileBar`) below 820px width —
  check both desktop and mobile breakpoints when changing layout/CSS.
