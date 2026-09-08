# SOVC Live Dashboard — v2

Single static page (`index.html`), no build step, no backend. Reads your Google
Sheet directly in the browser and recalculates every statistic from the live rows —
nothing is hard-coded.

## Repository structure (required for Vercel)

```
SOVC-Demo-dashboard-/
├── index.html      ← must be exactly this name, lowercase, in the repo root
├── vercel.json
└── README.md
```

If you previously had a file named `Index.Html.txt` or similar, delete it —
Vercel serves whatever sits at the repo root as `/`, and a misnamed file causes
the 404 you saw.

## Step 1 — Publish your Google Sheet as CSV

1. Open your Google Sheet.
2. **File → Share → Publish to web**.
3. First dropdown: select the specific sheet tab (e.g. `2026 SEP`) — not "Entire document".
4. Second dropdown: **Comma-separated values (.csv)**.
5. Click **Publish**, confirm, then copy the link (`https://docs.google.com/spreadsheets/d/e/.../pub?output=csv`).

**Why this step is required:** a static site hosted on Vercel has no server and no
Google credentials — it can only read data that's already public. Publishing to
web makes just that one tab readable as plain text; it does **not** make the
sheet editable by others, and you can un-publish it any time from the same menu.
If you need the data to stay fully private, the alternative is a small serverless
function (e.g. a Vercel Function using a Google service account) that fetches the
sheet server-side — meaningfully more setup, and only worth it if "published but
read-only" isn't acceptable for your data.

## Step 2 — Point the dashboard at your sheet

Open `index.html`, find the `CONFIG` block near the top of the `<script>` section:

```js
const CONFIG = {
  CSV_URL: "PASTE_YOUR_PUBLISHED_CSV_LINK_HERE",
  REFRESH_INTERVAL: 5 * 60 * 1000
};
```

Paste your link into `CSV_URL`. That's the only edit required.

## Step 3 — Deploy / redeploy on Vercel

If the project is already connected to this GitHub repo, committing the change
above triggers an automatic redeploy. Otherwise: vercel.com → **Add New → Project
→ Import Git Repository** → select this repo → Deploy (no framework/build
settings needed, it's static).

## How the field mapping works

Your sheet's actual headers are: `Date, Product category, Product Name, AOV,
Customer No., Demo status, Order status, Agent Name, department`.

The dashboard reads whatever headers are actually in row 1 and matches them to
internal fields by name (case-insensitive), so it keeps working even if you
reorder columns. Confirmed mapping for your current sheet:

| Dashboard field | Reads from your column | Status |
|---|---|---|
| Date | `Date` | ✅ |
| Department | `department` | ✅ |
| Agent | `Agent Name` | ✅ |
| Customer | `Customer No.` | ✅ |
| Product | `Product Name` | ⚠️ present but empty in every row — dashboard automatically falls back to `Product category` for all product analysis until you start filling it in |
| Demo status | `Demo status` | ✅ (only `Completed` / `Pending` values exist today — see note below) |
| Order/Sale status | `Order status` | ✅ (only `Sold` / `Pending` values exist today) |
| Sale amount | `AOV` | ⚠️ column exists but blank in every row — revenue-based KPIs will read 0 until this is filled in |
| Time | — | ❌ not in your sheet — no time-of-day feature is shown |
| Remarks | — | ❌ not in your sheet |

## Important: "Missed" demos

Your `Demo status` dropdown currently only offers `Completed` and `Pending` — there's
no `Missed` option, so the dashboard's Missed-demo KPI, the "Agents Requiring
Attention" section, and the red/yellow/green alert system will show 0 / empty
until you add a `Missed` (or similar) value to that column's dropdown in Google
Sheets. The dashboard already knows how to read it the moment it appears — no
code change needed on your end.

## What updates automatically

- Every KPI, chart, table, and section recalculates from whatever rows exist —
  add a row in the sheet and it appears on next refresh (every 5 minutes, or
  click "Refresh Data" for an instant pull).
- All filters (date range, department, agent, product, search) are computed
  live from the same live dataset — nothing is precomputed or cached beyond the
  5-minute refresh window.
- Department and agent lists in the filter dropdowns are read directly from the
  sheet — nothing is hard-coded.

## Data quality handling

- Blank rows are skipped.
- Missing agent/department/product names fall back to "Unknown Agent" /
  "Unassigned" rather than breaking the page.
- Status text is normalized regardless of capitalization (`Sold`, `sold`, `SOLD`
  all count the same way) and trailing/leading spaces are trimmed automatically
  (your sheet currently has some, e.g. `"Sold "` — this is handled, but worth
  cleaning at the source since not every tool downstream will be as forgiving).
- Invalid or unparseable dates are excluded from date-filtered views rather than
  crashing the page.
- One bad row never breaks the whole dashboard — each row is processed
  independently.

## If the sheet can't be reached

If `CSV_URL` is missing or the fetch fails, the dashboard shows a clear red
error banner explaining what's wrong — it does **not** fall back to fake data.
