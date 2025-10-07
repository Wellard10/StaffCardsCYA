## Staff Card Clean v2 — Model Health Report

This report is an automated, read-only analysis of the exported Power BI report and semantic model in this repository. It lists inventory items, notable findings, and a prioritized set of safe recommendations you can apply (with validation steps). I did not change any model or report files while generating this.

---

## What I inspected
- Semantic model folder: `Staff Card Clean v2.SemanticModel/definition/` (includes `model.tmdl`, `relationships.tmdl`, `database.tmdl`, `tables/*.tmdl`, `roles/*.tmdl`).
- DAX queries: `Staff Card Clean v2.SemanticModel/DAXQueries/`
- Report manifest and pages: `Staff Card Clean v2.Report/definition/report.json` and `.../pages/pages.json`.
- Underlying data workbook (source for many partitions): `CYA Staff Card POW BI V2.xlsx` (present at repo root; the model loads it via SharePoint Web.Contents in many table partitions).

## Quick inventory (automatically collected)
- .tmdl files discovered: 96 (tables, roles, cultures, relationships, model, database entries).
- Key model files: `model.tmdl`, `relationships.tmdl`, `definition/*.tmdl` (per-table definitions under `definition/tables/`).
- DAX: at least one exported DAX query `DAXQueries/Query 1.dax` and multiple measures embedded in table `.tmdl` files (example: `TE - Data` contains 'Count of Event value (Latest per Attendee)').
- Report pages: 6 pages listed in `pages/pages.json` (page order visible).

## Notable findings (concrete examples)
- Excel as source: many table partitions use Excel.Workbook(Web.Contents("https://cyacumbria.sharepoint.com/.../CYA Staff Card POW BI V2.xlsx")) — the authoritative data source is a SharePoint-hosted Excel file. See e.g. `Staff ID.tmdl`, `TE - Data.tmdl`.
- DAX example: `DAXQueries/Query 1.dax` contains measure `DailyContractedHours` which uses LOOKUPVALUE and RELATED; such logic depends on clean relationships (see note below).
- Relationships: `relationships.tmdl` shows numerous joins to `DateTable.Date` (Event date, Sessions Date, StaffSchedule.Date, Calendar.Start Date). Multiple relationships are marked `isActive: false` (AutoDetected_*). Several relationships use `crossFilteringBehavior: bothDirections` and `toCardinality: many` — these can create ambiguous filter propagation and performance issues.
- Duplicate / exported copies: many tables include `(2)` in their filenames (for example `CYA People Report (2).tmdl`, `Barriers (2).tmdl`) and `.bak` files exist (e.g., `StaffSchedule.tmdl.bak`). These are likely exported duplicates or backups and should be reviewed before consolidation.
- Role-based RLS: Role `ViewerRLS` is present and applies a table-level filter to `Staff ID` using `Viewer Table` and `USERPRINCIPALNAME()`; changes to `Viewer Table` or `Staff ID` must be validated carefully.

## Problems that commonly reduce health or performance (observed here)
- Multiple active/bi-directional relationships between fact and lookup tables can cause ambiguous filtering and slower queries.
- Inactive relationships (AutoDetected_*) may indicate the model has duplicate join keys or slightly mismatched values (string vs int, trimmed spaces, multiple date formats).
- Heavy use of LOOKUPVALUE / RELATED in calculated columns or measures is fragile if relationships are inactive or when duplicate keys exist.
- Duplicated tables / `(2)` exports increase model size and risk users connecting to the wrong table in visuals.
- Using Excel on SharePoint as the single large source can be OK, but performance depends on how the workbook is structured (large sheets, many transformations in Power Query, splits/expansions). Consider moving to a more robust storage (CSV/Parquet or a database) for large datasets.

## Quick wins (low risk, prioritize first)
1. List and archive duplicates and `.bak` files — do NOT delete; move to `archive/` or rename with `.archived` suffix. Files: any `*.tmdl.bak` and `*(2).tmdl`. Why: reduces confusion when opening model files or Tabular Editor.
   - Validation: open `definition/tables/` and confirm only canonical table(s) remain referenced in `model.tmdl`/`relationships.tmdl`.

2. Replace unnecessary bothDirections relationships with single-direction where the relationship only needs to flow from lookup -> fact. Why: improves query plan and reduces filter ambiguity.
   - Example candidate: relationships with `crossFilteringBehavior: bothDirections` between `Strat2.Activity` and `'Sessions + Activities (2)'.Activity` (see `relationships.tmdl`).
   - Validation: run visuals that rely on those relationships in Power BI Desktop and confirm expected filters after change.

3. Consolidate date relationships: ensure the model uses a single Date table (DateTable) with a clear active relationship per fact-date. If multiple date columns exist (Start Date, Event Date, Session Date), prefer using inactive relationships for secondary dates and use USERELATIONSHIP in measures when needed.
   - Example: many facts link to `DateTable.Date` (TE - Data Event date, Sessions Date, StaffSchedule.Date) — confirm which should be active.

4. Convert large calculated columns that are only used in visuals to measures where possible (reduces storage cost). Candidate columns: calculated columns in `DateTable` used as flags (IsHoliday is fine in date table) but look for repeated text-to-number transformations in fact tables.

5. Review DAX using LOOKUPVALUE or RELATED (e.g., `DailyContractedHours`): where feasible, replace LOOKUPVALUE with measures that use relationships or with proper join keys. Why: robust to changes and often faster as measures.

## Medium / higher-risk items (require validation in Power BI Desktop or Tabular Editor)
- Rewire relationships to use canonical keys (e.g., make `Staff ID` the single join point for staff names/ids across tables). This may require deduplicating or transforming keys in the source Excel workbook or in Power Query.
- Remove redundant tables from the model (merging duplicated exports) — requires careful step-by-step validation and test reports.
- Update RLS logic (`ViewerRLS`) only with strict testing since it controls visible rows per user.

## Suggested prioritized plan (practical next steps)
1. (Day 0) Create an `archive/` folder and move `*(2).tmdl` and `*.bak` files there (non-destructive). Commit the archive move. This reduces developer confusion.
2. (Day 1) Produce a relationship-change PR that converts selected bothDirections relationships to single direction. Keep changes minimal (1–3 relationships per PR), include a validation checklist and screenshots in the PR description.
3. (Day 2) Replace obvious calculated columns implemented in fact tables with measures where they are used only in aggregations.
4. (Week) If model size / performance remains a problem, extract heavy Excel sheets into CSV/parquet or load them into a database and change the model partitions to use that source.

## Commands & safe automation snippets (how to apply small changes)

PowerShell pack/unpack (example — replace `report.json` inside a `.pbit`):

```powershell
# backup
Copy-Item '.\Staff Card Clean v2.pbit' '.\Staff Card Clean v2.pbit.bak'

# extract
$tmp = Join-Path $env:TEMP 'pbit-unpack'
Remove-Item $tmp -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Path $tmp | Out-Null
Expand-Archive -Path '.\Staff Card Clean v2.pbit' -DestinationPath $tmp

# replace file(s) from repo
Copy-Item -Path '.\Staff Card Clean v2.Report\definition\report.json' -Destination (Join-Path $tmp 'Report\definition\report.json') -Force

# recompress
Compress-Archive -Path (Join-Path $tmp '*') -DestinationPath '.\Staff Card Clean v2.updated.pbit'
Start-Process -FilePath '.\Staff Card Clean v2.updated.pbit'
```

Tabular Editor CLI (example template — adapt for TE2/TE3 installation):

```powershell
# TE3 example
$te = 'C:\Program Files\Tabular Editor 3\TabularEditor.exe'
$script = '.\scripts\change-relationship-direction.csx' # C# script to change relationship properties
& $te -Script $script -InputModel '.\Staff Card Clean v2.SemanticModel\definition\model.tmdl' -OutputModel '.\Staff Card Clean v2.SemanticModel\definition\model.tmdl'
```

Note: Tabular Editor scripts should be idempotent and make minimal changes; always backup `model.tmdl` first.

## Safety & validation checklist (to include in PRs)
- Create backups of `.pbit`, `.pbix`, `model.tmdl` before edits.
- Make one small, focused change per PR.
- In PR description include: files changed, why, manual validation steps (open `*.pbit` -> check page X and visual Y), and RLS role checks.
- Attach screenshots showing before/after visuals or the model diagram if relationships changed.

## Deliverables I will create if you ask next
- `tools/pack-pbit.ps1`: safe pack/unpack script with dry-run mode. (I can add and demonstrate a dry-run.)
- `scripts/change-relationship-direction.csx` + `tools/run-tabular-editor.ps1`: small Tabular Editor script to toggle relationship cross-filtering or direction (TE3 example). (I will not run Tabular Editor here; I will generate the script and explain how to run it locally.)
- A follow-up PR with a small, non-destructive change (for example, archiving duplicate files) and validation checklist.

---

If you'd like, I will now create `MODEL_HEALTH_REPORT.md` in the repository (this file) and then implement the first low-risk change: move `*(2).tmdl` and `*.bak` into `archive/` and open a PR with that single change. Reply with one of:

- "Archive duplicates" — I will create `archive/`, move duplicates there, commit, and push a branch + PR.
- "Add scripts" — I will add `tools/pack-pbit.ps1` and a TE3 example script (I will not run Tabular Editor locally; I will provide run instructions).
- "Stop" — keep the report only.

Report created: `MODEL_HEALTH_REPORT.md` (this file).
