# Novella Bioreactor / Experiment Schedules

Live scheduling dashboards for Novella's labs. Each site below is an
independent, single-file dashboard (no build step, no dependencies) served
directly by GitHub Pages:

- **Modi'in** (bioreactors) — `/` — https://yonatan-levinson.github.io/bioreactormodiin/
- **Chemo** (bioreactors) — `/chemo/`
- **CPI** (bioreactors) — `/cpi/`
- **ECL** (bioreactors) — `/ecl/`
- **Biology** (flask experiments) — `/biology/`

Every dashboard also has a "Manage stations" (or "Manage platforms") menu
item, so you can rename the placeholder station/line to whatever the real
platform is called at that site — no code changes needed per site.

Each dashboard's **calendar** data is independent — each needs its own Google
Sheet and Apps Script deployment before its own cloud sync will work (see
below). Cloud sync starts **off** on every new site until you connect one.

The **Projects** tab is the one exception: it's shared across all five
dashboards and always reads/writes Modi'in's Sheet specifically, regardless of
which site you opened it from or whether that site's own cloud sync is
connected. So a project added from the Chemo dashboard shows up on Modi'in's
Projects tab too, and vice versa — there's one project list, not five.

## Connecting cloud sync for a new site

Live sharing uses a free Google Apps Script Web App + Google Sheet in your
own Google account. One-time setup per site, about 10 minutes.

1. Go to [sheets.new](https://sheets.new) to create a fresh Google Sheet for
   this site (name it something like "Novella — Chemo Schedule").
2. In the Sheet, go to **Extensions → Apps Script**.
3. Delete any starter code in `Code.gs` and paste in the script below.
4. Click **Deploy → New deployment**. For "Select type," choose **Web app**.
   - Execute as: **Me**
   - Who has access: **Anyone with the link**
5. Click **Deploy**, then authorize the script when Google prompts you
   (it's your own script running in your own account — this is expected).
6. Copy the resulting Web App URL — it ends in `/exec`.
7. Open the dashboard for that site, click the **⋯** menu → **Connect cloud
   sync…**, paste the URL, and save.

Updating the dashboard code later (pushing a new `index.html`) never touches
data already stored in the Sheet — the backend just stores whatever JSON
it's given, keyed by field name, so unrelated fields are untouched by a
schema change.

### Code.gs

```javascript
/**
 * Novella Bioreactor Schedule — cloud sync backend
 * Runs as a Google Apps Script Web App. Stores the whole schedule as JSON
 * in a single cell of a Google Sheet. The dashboard reads (GET) and writes (POST).
 *
 * Setup is in the SETUP_GUIDE. You do NOT need to edit anything below.
 */

var SHEET_NAME = 'data';
var CELL = 'A1';

function _sheet() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sh = ss.getSheetByName(SHEET_NAME);
  if (!sh) sh = ss.insertSheet(SHEET_NAME);
  return sh;
}

// GET  -> returns the stored schedule JSON
function doGet(e) {
  var raw = _sheet().getRange(CELL).getValue();
  var out = raw ? String(raw) : '{"stations":[],"runs":[],"tasks":[]}';
  return ContentService.createTextOutput(out)
    .setMimeType(ContentService.MimeType.JSON);
}

// POST -> overwrites the stored schedule JSON with the request body
function doPost(e) {
  var lock = LockService.getScriptLock();
  lock.waitLock(20000); // avoid two writers clobbering each other
  try {
    var body = e.postData && e.postData.contents ? e.postData.contents : '';
    // validate it parses before storing
    JSON.parse(body);
    _sheet().getRange(CELL).setValue(body);
    return ContentService.createTextOutput('{"ok":true}')
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput('{"ok":false,"error":"' + err + '"}')
      .setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}
```

## Repo conventions

- Single file per site. No bundler, no framework, no `package.json`.
- No CDN dependencies — nothing that requires network beyond each site's own
  cloud-sync endpoint.
- See `CLAUDE.md` for architecture details and conventions for future changes.
