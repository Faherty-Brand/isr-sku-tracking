# ISR — SKU Tracking

A single-page web app used by Faherty Brand's Allocation & Store Planning team to manage seasonal SKU allocation planning, replenishment configuration, newness review, Late List management, and Initial Allocation (IA) volume forecasting.

It replaces what used to be a multi-file Excel workflow with one shared tool: everyone works from the same published dataset, and overrides, notes, and review decisions sync in real time across users.

> **Owner:** Christian Schmidt (Allocation & Store Planning)
> **Companion tool:** Store Matrix (Google Apps Script), owned by Lauren

---

## Contents

- [What the tool does](#what-the-tool-does)
- [Getting access](#getting-access)
- [Tabs at a glance](#tabs-at-a-glance)
- [Seasons: CS / PS / NS](#seasons-cs--ps--ns)
- [Data sources](#data-sources)
- [Refreshing data](#refreshing-data)
- [Architecture](#architecture)
- [Repository layout](#repository-layout)
- [Making changes](#making-changes)
- [Troubleshooting](#troubleshooting)
- [Roadmap](#roadmap)

---

## What the tool does

At a high level, ISR pulls together planning, inventory, sales, WIP, flow, and Syrup data into one SKU-level grid, then layers planner decisions (tier, delivery, end-of-life dates, notes) on top. Those decisions feed a **Configs Generator** that produces the allocation and replenishment configurations loaded into **Syrup**.

Typical uses:

- Seeing every SKU in a season with its tier, delivery, on-hand, WIP, and lifecycle status in one place
- Setting season-scoped overrides (tier, delivery, notes) that everyone on the team can see
- Reviewing new arrivals and how deeply they were allocated at drop time
- Catching SKUs that are past their computed end date but still selling, and extending them so they stay in replenishment
- Tracking late WIP and forecasting IA volume

## Getting access

1. Open the GitHub Pages URL for this repository (see the repo's **About** panel / Pages settings).
2. Sign in with your Faherty email and password. Accounts are managed in Supabase — ask Christian if you need one created.
3. On load, the app reads the most recently **published snapshot**, so you don't need any local files to view data.

Only admins load raw source files and publish; most users just sign in and work.

## Tabs at a glance

| Tab | Purpose |
| --- | --- |
| 📋 **SKU Tracking** | The main grid. One row per SKU for the working season, with tier, delivery, inventory, WIP, lifecycle dates, overrides, and notes. Exportable to CSV. |
| ⚠ **Late List** | WIP running behind its planned delivery. Switchable between CS / PS / NS and exportable. |
| 🎯 **NF Review** | Newness/Fit review. In **Released** mode it calculates allocation depth *at drop time* (drop date, drops, drop stores, reach, dropped units, depth, per-size) rather than using today's on-hand. Depth is based on each SKU's largest ("primary") drop, since follow-up drops are usually small top-ups. Sales are intentionally excluded. |
| ⏳ **EOL Review** | End-of-lifecycle review. Bulk-review SKUs near or past their computed end date and override it with a date, `+Nw` (extend N weeks), or `INDEF` (indefinite). The Configs Generator picks these up so SKUs that are still selling can stay in replenishment. Includes tier and notes popovers, column filters, multi-SKU lookup, and a toggle to sink out-of-season / markdown SKUs. |
| 📦 **IA Volume** | Initial Allocation volume forecasting. |
| 🏬 **Store Matrix** | Tier-to-store mappings pulled from Lauren's Store Matrix tool, with a season toggle (e.g., FA26 and HO26 side by side). |

## Seasons: CS / PS / NS

The tool tracks rolling seasons (e.g., SP26, SU26, FA26, HO26) using three roles:

- **CS** — current season (the "working season")
- **PS** — previous season
- **NS** — next season

Overrides for tier, delivery, and notes are **season-scoped**, so a change made for one season doesn't leak into another.

**Seasonal changeover, in practice:**

- Next-season newness SKUs get Syrup configs *before* the season flips, so depth recommendations can be reviewed ahead of allocation.
- Carry-forward SKUs with tier changes wait until the new season starts; store groups are adjusted through replenishment.
- The tool's working season is flipped once the new season officially starts.

## Data sources

Admins load these files from a single folder. The loader matches files by name, so keep the names consistent.

| File | What it provides |
| --- | --- |
| `PST.xlsx` | Planning Spreadsheet Template — the seasonal SKU plan |
| `ISR INV Pull.csv` | Inventory (from 42 Reporting) |
| `ISR LCW Sales.csv` | Last-completed-week sales (from 42 Reporting) |
| `Size Sets.xlsx` | Size set definitions |
| `Newness Log.xlsx` | Newness drops (supports multiple drops per SKU) |
| `Flow Cal.xlsx` | Flow calendar |
| `WIP.xlsx` | WIP tracker export |
| `Store Codes.xlsx` | Store code reference |
| `Safety Stock.csv` | Safety stock settings |
| `syrup IA.csv` | Syrup Initial Allocation export |
| `syrup RP.csv` | Syrup Replenishment export |
| `syrup configs.csv` | Current Syrup configurations |

Store Matrix data (tier-to-store mappings, group category) is fetched live from Lauren's Apps Script tool and stored with the published snapshot.

## Refreshing data

1. **Refresh the source files.** Most of the folder can be refreshed automatically with `isr_update.py` (see below). PST, Newness Log, Flow Cal, Store Codes, Size Sets, and Safety Stock are maintained manually.
2. **Load the folder** in the app (folder picker).
3. **Spot-check** the grid and console for warnings (e.g., missing or duplicate headers).
4. Click **Publish data → everyone**. This pushes the loaded dataset as the new shared snapshot. Other users pick it up on their next load.

### `isr_update.py`

A separate Python script that refreshes:

- `ISR INV Pull.csv` and `ISR LCW Sales.csv` from 42 Reporting emails
- `syrup IA.csv` and `syrup RP.csv` from links in Syrup export emails
- `WIP.xlsx` from the local WIP Backlog folder

It authenticates to Gmail via Google OAuth. It's packaged as a Claude skill (**isr-update-setup**) that builds a personalized setup kit — Python, dependencies, and Gmail API credentials — so other team members can run it independently. Each user authenticates with their own Google account; don't share OAuth tokens.

## Architecture

```
            ┌───────────────────────────┐
 Source     │  Folder of source files   │  ← isr_update.py + manual files
 files ───▶ │  (admin's machine)        │
            └─────────────┬─────────────┘
                          │ load + parse in browser
                          ▼
            ┌───────────────────────────┐
            │  index.html (GitHub Pages)│  single-file HTML/CSS/JS
            │  SheetJS · PapaParse      │
            └──────┬──────────────┬─────┘
       publish /   │              │  fetch tier→store
       read        ▼              ▼
   ┌────────────────────┐   ┌────────────────────────┐
   │ Supabase           │   │ Store Matrix           │
   │ Auth · Postgres ·  │   │ (Apps Script, Lauren)  │
   │ Realtime           │   └────────────────────────┘
   └────────────────────┘
                          │ Configs Generator output
                          ▼
                      ┌────────┐
                      │ Syrup  │
                      └────────┘
```

- **Frontend:** one self-contained `index.html` — HTML, CSS, and JavaScript with SheetJS inlined, plus PapaParse. No build step.
- **Hosting:** GitHub Pages under the Faherty Brand organization.
- **Backend:** Supabase for authentication, persistence, and realtime sync. Shared state lives in these tables:
  - `snapshots` — the published dataset everyone reads
  - `overrides` — season-scoped tier / delivery / EOL overrides
  - `notes` — SKU notes
  - `nf_review` — NF Review decisions
  - `store_matrix` — Store Matrix payloads per season
- **Realtime:** each of the above has a realtime channel, so edits made by one user appear for others without a reload.

## Repository layout

```
/
├── index.html   # the entire app
└── README.md    # this file
```

## Making changes

The app is maintained as one file, and every release is a **drop-in replacement** of `index.html`.

**Build stamps.** Each module logs a build stamp to the browser console on load, for example:

```
[EOL] End-of-lifecycle module ready [eol-review p7z4]
[SM] Store Matrix module ready [store-matrix-v1 + p7z11 season/cov-min]
```

After deploying, open DevTools → Console and confirm the stamp matches the build you shipped. GitHub Pages can take a minute or two to update, and browsers cache aggressively — hard refresh if you still see the old stamp.

**Before shipping a change:**

1. Syntax-check every inline `<script>` block (e.g., extract and run `node --check`).
2. Run the regression harness (Node `vm.runInContext` with DOM stubs). Using a shared VM context matters: top-level `let` / `const` in separate `<script>` blocks don't attach to `window`, and the harness reproduces that.
3. Check that business-logic changes are applied everywhere they're read — grid, Late List, NF Review, CSV export, side panel. Prefer shared resolver functions over duplicated logic.
4. Bump the build stamp and note what changed.
5. Commit the new `index.html` with a message that includes the build stamp.

**Conventions worth knowing:**

- **Resolve columns by header name**, not position, with fallbacks and collision warnings. Source exports drift.
- **Be careful with dates.** Mixed formats, two-digit years, and stale fallback dates have all caused bugs. The "infinity" sentinel is Excel serial `73050` = 12/31/2099.
- **When migrating logic from Excel,** the live workbook's formulas are the source of truth.
- **Prefer simple, predictable precedence rules** with documented tradeoffs over clever reconciliation.

## Troubleshooting

| Symptom | Likely cause / fix |
| --- | --- |
| Data looks out of date | The latest snapshot hasn't been published, or your browser cached an old build. Hard refresh, then check the console build stamp. |
| Can't sign in | You may not have a Supabase account yet — contact Christian. |
| Edits from teammates aren't appearing | Check the console for realtime subscription errors and reload. |
| A column is blank after loading new files | A source export's header may have been renamed. Check the console for header warnings. |
| Store tiers missing | The Store Matrix fetch may have failed or the selected season has no matrix yet. Check the season toggle on the Store Matrix tab. |
| Exported CSV has odd characters | SKU Tracking export sanitizes special characters; Late List and NF Review exports may not yet. |

## Roadmap

Ideas under consideration (not committed):

- Migrating the Allocation Log and Replenishment Minimums processes from Excel
- An Allocation Log feature that ingests raw allocation/replenishment feeds directly (with attention to file size)
- Pinning the Supabase client library version
- CSV sanitation for Late List and NF Review exports
- Supporting handoffs to upcoming forecasting software partners

---

Questions, bugs, or feature requests: reach out to Christian Schmidt or open an issue in this repository.
