# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

RNGD Look-Ahead — a crew scheduling / production-tracking board (PWA) for construction project DPW547 (job 2124013), deployed on Netlify. A static single-file React app syncs a shared board across devices through one Netlify function backed by Netlify Blobs. An offline Python pipeline (`analysis/trucking/`) is the reference implementation for the in-app trucking-document analysis.

## Commands

```sh
npm install                          # once — devDeps are only needed to run the e2e
node tools/dev-server.mjs            # local dev at http://127.0.0.1:8888 with /api/state emulated in-memory
node tools/dev-server.mjs --no-api   # simulate non-Netlify hosting (app must degrade to local-only)
BOARD_KEY=secret node tools/dev-server.mjs   # exercise the 401/key flow
npm run e2e                          # THE test suite (tools/e2e-docs.mjs); exit 0 = pass
node tools/e2e-docs.mjs --headed     # same, watching the browser
npm run fixtures                     # regenerate synthetic xlsx fixtures (tools/make-fixtures.mjs)
```

- There is no build step (`npm run build` is an echo — Netlify publishes the repo root as-is), no linter, and no unit-test runner. The whole suite is `npm run e2e` plus the in-app domain selftest.
- Domain selftest: open `/?selftest=1` (green badge `#dpwselftest` with `data-pass="1"`) or call `window.__DPW.selftest()` in the console. The e2e asserts it too. When you touch any `dpw*` function, extend `dpwSelftest()` alongside it.
- The e2e is a plain node script, not a runner: it boots the dev server, drives headless Chromium via `playwright-core` (`/opt/pw-browsers/chromium`, override with `PW_CHROMIUM`), and serves the app's CDN scripts from `node_modules` with all other third-party requests aborted. There is no way to run "a single test"; steps run in sequence and later steps depend on earlier state. Screenshots/artifacts land in `tools/e2e-out/` (gitignored).
- Python pipeline: `cd analysis/trucking && python3 run.py` (pandas/numpy/openpyxl/xlsxwriter). It aborts unless the invoice workbook reconciles. It needs real workbooks in `analysis/trucking/data/`, which are **not in the repo** — data and output are gitignored because the repo root is a publicly served site. Real tracker/timesheet files must never be committed; the e2e uses synthetic fixtures only.

## Architecture

### The single file

`index.html` (~8,300 lines) is the entire frontend: React 18 UMD + Babel Standalone loaded from unpkg, one `<script type="text/babel">` block, JSX compiled in the browser. No modules, no imports — everything is top-level in one script, roughly in this order:

- Sync constants (`SYNC_CORE_KEYS`, `syncLS`) — top of the script
- Baked-in project data: `CATALOG` (bid items; giant one-line arrays — grep, don't read whole-file), `CTC_DATA`, `CD_CYT`, `TRUCK_BASE`, `SCOPES` (cost-code-prefix → scope color system)
- `DPW DOCS DOMAIN` (~line 190–950): pure `dpw*` functions + `dpwSelftest`
- Physical trucking model (`truckModel`, `haulPlan`, `routePlan`)
- `LookAhead` (~line 1500): the root component owning ALL state, persistence, and the sync engine (`applyState` / `pushNow` / `pullNow` / `applyRemote`)
- View components (`WeekTable`/`BoardStrip`, `MobileBoard`, `Trends`, `MilestoneView`, `CatalogView`, `ProgressView`, `InvoicesView`), then modals, PDF drawing (`draw*Pdf`), and finally the CSS in a `const CSS` template string
- Desktop tabs: Board, Trends, Milestones, Catalog / CTC, Progress, Invoices. Mobile is board-only. Trucking is a modal, not a tab.

Navigate with `grep -n "^function Name"` and ranged reads; never read the whole file.

### Board data model

- `crews` (type `crew`/`sub`) and `siteRows` are board rows; `tasks` are day cards: `{id, rid, res, ci, wk, day, qty, sta, loc, ov, dig, ...}` where `ci` indexes `CATALOG`, `wk` is a Monday ISO date, `day` 0–6. Per-card overrides live in `t.ov`.
- A **run** = one multi-day operation; identity is `rid` (see `runKey`, `migrateRuns` for legacy fallback). `deps` maps dependent runKey → predecessor runKeys; delaying a predecessor pushes successors forward (MS-Project style "hold dates, push later"), cascade in `LookAhead`.
- `logs` (EOS actuals) are keyed by task id and merged per-task by newest `at` on remote pulls. `rainDays`, `holidays`, `baseline`, `milestones`, `prodEvents` feed the forecast/variance surfaces.

### Sync (the part that bites)

- Server: `netlify/functions/state.mjs` → `/api/state`, one JSON envelope `{rev, savedAt, device, deviceName, state}` in Blobs store `lookahead`, key `board-v1`. Writes are **rev-guarded**: PUT must carry `baseRev` matching the stored rev or it gets 409 + the current envelope. `savedAt` is display-only — **never arbitrate conflicts by wall clock** (that mistake once cost a day of scheduling). Optional `BOARD_KEY` env var requires an `x-board-key` header. The `x-board: 1` response header is how the client tells the real endpoint from a 404/SW page. A 7-slot weekday ring (`board-bk-0..6`, `GET ?backup=list` / `?backup=<0-6>`) snapshots the outgoing envelope on the first accepted PUT of a new calendar day. Payload cap 2 MB.
- Client engine (inside `LookAhead`): local persistence to `lafable-v3` debounced 600 ms; no push until the boot pull resolves; poll ~25 s while visible; a tab that slept (>2 min without a heartbeat) must pull-and-adopt before it may push; on 409 the local diff is rescued to a backup key, then the server copy is adopted; a pull-echo fingerprint (`SYNC_CORE_KEYS` order) prevents a pull's own re-serialization from being pushed back. Shared slices merge on remote apply (`logs` per-task, `mergeProgress`/`mergeInvAlloc`/`mergeEvents`); everything else is replace.
- **Adding a synced state slice touches five places in lockstep**: the `useState` in `LookAhead`, `SYNC_CORE_KEYS`, both serializations in the save effect (`core` and `json`) plus its dep array, and `applyState`. Miss one and you get silent data loss or a push loop. Keys must keep the save effect's serialization order.
- Raw imported document rows (`docs`) are device-local under `lafable-docs-v1` and must stay out of the sync payload (they're big and re-derivable by re-importing).
- Known divergence: `tools/dev-server.mjs` still resolves PUT conflicts by comparing `savedAt` and doesn't implement the backup ring — the production function moved to the `baseRev` guard. The e2e therefore doesn't exercise the exact production 409 rule. If you touch the sync contract, update the dev server to match production, not the other way around.

### Docs domain (Progress / Invoices tabs)

The `dpw*` functions are a deterministic JS port of `analysis/trucking` (rates → classifier → allocator → rate audit → anomaly flags). All money is **integer cents**; rate-table keys are cent-rounded strings (`"105.00"`) so float equality can never lie. Invoice-number **eras** (`DPW_ERAS_DEFAULT`) partition history; statistics must never cross era boundaries. The port deliberately fixes two upstream Python bugs (the rate-audit verdict ternary at `analyzer.py:156` and era-crossing code-spend stats) — the Python is the reference for behavior, but the JS is the corrected implementation; don't "fix" the JS back to match.

### PWA / deployment

- `sw.js` caches the app shell cache-first. **Any user-visible change to `index.html` requires bumping `VERSION` in `sw.js`** (e.g. `rngd-v73` → `rngd-v74`) or deployed clients keep the old shell. Weather hosts (api.weather.gov, open-meteo.com) are network-first; `/api/` is never cached.
- `netlify.toml`: publishes the repo root, functions from `netlify/functions` bundled with esbuild.
- If a change adds a new CDN script to `index.html`, pin the same file in `devDependencies` and add it to the `cdnCache` map in `tools/e2e-docs.mjs`, or the e2e (which runs with no external network) will break.

## Conventions

- Sections marked `⚠️ LOCKED` (board-trailer card print specs — 5 markers in `index.html`) are owner-approved field artifacts. Do not alter them without the owner's explicit ask.
- Default branch is `mainfc`. Commit messages are single imperative descriptive lines (see `git log`), no conventional-commit prefixes.
- Comment style in `index.html` explains *why* (invariants, field-use rationale), often in block comments above a section — keep that density when editing.
- Print CSS has three modes (report modal / card sheets / board snapshot) that depend on releasing the app's `overflow:hidden` scroll containers — test printing when touching board layout CSS.
