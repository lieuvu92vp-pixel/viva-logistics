# VIVA Logistics Portal

Single-page dashboard for tracking import/export shipments. Data lives in a
Google Sheet; there is no backend/database in this repo.

## Repo shape

- `index.html` — the entire app (~2,950 lines): HTML + CSS + JS in one file.
  No build step, no bundler, no `package.json`. Editing this file *is* the
  deploy artifact.
- No CI workflows, no tests. Verification is manual (open the file / deployed
  URL in a browser and click through).
- Hosting: **production is `https://viva-logistics.lieuvu-92-vp.workers.dev`,
  deployed by manually uploading `index.html` to Cloudflare via the
  dashboard** — this repo's GitHub history is not the deploy source for that
  Worker. A separate Cloudflare Pages project (`viva-logistics.pages.dev`)
  *is* Git-connected and auto-deploys from this repo, but it is not the URL
  staff/customers actually use — treat it as unused/stale, not production.
  This split happened deliberately: an earlier attempt at GitHub-driven
  auto-deploy reportedly broke something related to Apps Script (GAS),
  suspected to be origin/CORS allowlisting rejecting Cloudflare Pages'
  per-deploy preview URLs. Details of that failure aren't documented, so
  **don't try to reconnect Git-based auto-deploy for production** without
  first confirming with the user — the safe workflow is: edit `index.html`
  in this repo, hand the finished file to the user, they upload it to
  Cloudflare by hand.

## Data flow

```
Browser (index.html) --fetch(GET)--> Cloudflare Worker proxy --> Google Apps Script --> Google Sheet
```

- `API_URL` (index.html:823) = `https://viva-proxy.lieuvu-92-vp.workers.dev`.
  The worker and Apps Script live outside this repo — not editable here.
- All calls go through `api(action, params, _method, auth)` (index.html:1255).
  Everything is sent as a **GET with query params** (including writes) —
  `_method` is currently unused/legacy. Auth token (from login) is appended
  as `&token=...` when `auth` is true.
- Known `action` values (grep `api('` to find call sites / add new ones):
  `login`, `getDashboard`, `getImports`, `getExports`, `addImport`,
  `addExport`, `updateImport`, `updateExport`, `updateArrival`,
  `importExcel`.
- Response contract: `{ ok: boolean, data?, error? }`. `api()` throws on
  `!ok`, non-2xx, or non-JSON body — callers should wrap in try/catch and
  use `showToast(msg, 'error')` on failure (existing pattern throughout).

## Auth & permissions

- Login (`doLogin`, index.html:1143) posts username/password, stores the
  returned session (`session.token`, `session.role`, `session.permission`)
  in `localStorage['viva_session']`.
- Permission string drives UI, not just the backend: `ADMIN` / `EDIT` →
  `window._canEdit = true`; `ADMIN` alone → `window._isAdmin = true`
  (`renderUserInfo`, index.html:1197). `role === 'KHO'` hides the data-entry
  nav (index.html:1178). Treat this as **display-only** gating — the real
  authorization must happen server-side in Apps Script, since anyone can
  read the client JS.

## UI structure

- Single-page shell: login screen → `#appShell` (sidebar + topbar + page
  panels). Client-side hash routing via `navTo(id, el)` (index.html:1230);
  valid page ids: `dashboard`, `import`, `alerts`, `export`, `upcoming`,
  `entry`.
- i18n is manual: `LANG.vi` / `LANG.en` dictionaries (index.html:826) +
  `data-vi` / `data-en` attributes on elements + `applyLang()`
  (index.html:851), which also has hardcoded VI/EN maps for card titles and
  table headers (`cardTitles`, `tblHeaders`) — **any new page/label needs
  entries added in three places**: the dictionary, the DOM attributes, and
  those maps if it's a card title or table header.
- Tables (`import`, `export`) support client-side sort (`sortImport`/
  `sortExport`), text filter (`filterImport`/`filterExport`), and per-column
  dropdown filters (`cf*` functions, index.html:2762+, state keyed by
  `table`+`field`).
- Excel import/export uses SheetJS (`XLSX` global) to parse `.xlsx` client
  side (`handleFile`, `handleExportFile`) before POSTing parsed rows via
  `importExcel` / `addExport`.
- Charts via Chart.js (`renderChart`, `renderCustChart`).

## Conventions when editing

- Keep changes inside `index.html` unless deliberately splitting the file —
  it's a single-file app by design so far.
- Match the existing inline-CSS-variable style (`--navy`, `--teal`, etc. in
  `:root`) rather than introducing new colors ad hoc.
- Vietnamese is the primary language in UI strings and code comments; keep
  new user-facing strings bilingual (vi default, en via `data-en`/`LANG.en`).
- No linter/formatter configured — match surrounding style (2-space indent,
  minified-ish CSS, semicolon-terminated JS).
- Since there's no test suite, sanity-check changes by reasoning through the
  actual DOM ids/classes referenced (`document.getElementById(...)`) — typos
  here fail silently at runtime, not at build time.

## Working branch

Development happens on `claude/web-portal-sheets-github-4wov3y`. See git log
for `main` vs. this branch's history.
