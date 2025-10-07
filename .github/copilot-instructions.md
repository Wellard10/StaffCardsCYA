## Purpose
Short, actionable guidance for an AI coding agent working on this repository (Power BI report + semantic model). Focus on what to change, where to look, and safe workflows.

## Big picture (what this repo is)
- This repository stores an extracted Power BI report and semantic model for "Staff Card Clean v2". The repo contains two main artifacts:
  - `Staff Card Clean v2.Report/` — the report definition and pages exported as JSON (visuals under `pages/.../visuals/<GUID>/`). See `definition/report.json` and `definition/pages/pages.json` for the top-level structure.
  - `Staff Card Clean v2.SemanticModel/` — the tabular model files (model definition `.pbism` and per-table `.tmdl` files), DAX queries and roles under `DAXQueries/` and `roles/`.

Why this matters: editing the JSON or `.tmdl` files can change visuals, theme and model metadata, but the authoritative runnable artifact is the `.pbix`/`.pbit` that you must open in Power BI Desktop or repackage with tooling. There is no automated packaging script in the repo.

## Key files and where to look (examples)
- Report: `Staff Card Clean v2.Report/definition/report.json`
- Pages: `Staff Card Clean v2.Report/definition/pages/pages.json` and per-page folders like `.../08fd67cfd00b4a33029c/page.json` and `.../visuals/<GUID>/`.
- Theme: `Staff Card Clean v2.Report/definition/StaticResources/RegisteredResources/CYA_Theme23762211597342953.json` and `SharedResources/BaseThemes/CY24SU10.json`.
- Semantic model: `Staff Card Clean v2.SemanticModel/definition.pbism` and `Staff Card Clean v2.SemanticModel/definition/model.tmdl`.
- DAX: `Staff Card Clean v2.SemanticModel/DAXQueries/Query 1.dax` (single exported DAX file shown).
- Tables/roles: `.../tables/*.tmdl`, `.../roles/ViewerRLS.tmdl` — these show table schemas, relationships and role-level filters.

## Typical developer workflows (how to make safe changes)
- Iterative edit (recommended):
  1. Make small edits to JSON or `.tmdl` files (for e.g., theme colour or captions in `report.json`).
  2. Validate by opening the authoritative `*.pbit`/`*.pbip` in Power BI Desktop and re-applying the modified assets (or re-exporting). There is no repo script to repackage automatically.
  3. For semantic model edits, prefer Tabular Editor or SQL Server Data Tools: open `definition.pbism` or the `.tmdl` files in your model editor, apply changes, then validate by loading the `*.pbit` in Power BI Desktop.

Example PowerShell commands (run locally on Windows):
```powershell
# Search for a string across exported JSON/TMDL files
Get-ChildItem -Recurse -File | Select-String -Pattern 'SomeText' -SimpleMatch

# Open a file with the default app (Power BI Desktop, Tabular Editor, or JSON editor)
Start-Process -FilePath '.\Staff Card Clean v2.Report\Staff Card Clean v2.pbit'
```

## Patterns and repository conventions
- Visuals are grouped by GUID: change-level edits to visuals live under `.../visuals/<GUID>/`.
- The extracted layout is descriptive — prefer editing human-readable JSON keys like `title/text`, `format` or `dataBinding` only when you understand the effect. Small UI label and theme changes are low risk; changing dataset/table/column names is high risk.
- Backups and duplicates exist (filenames including `(2)` or `.bak`). Treat `*.bak` and files with `(2)` as historical copies — do not propagate them unless explicitly required.

## Integration points & external dependencies
- Power BI Desktop — primary tool to author and validate the report. There is no CI system here that rebuilds PBIX files.
- Tabular Editor / SSAS tools — useful for editing `.pbism`/`.tmdl` files and DAX expressions in `DAXQueries/`.
- Theme JSON files under `StaticResources/` are consumed by the report during open-time in Power BI Desktop.
- Role-level security is defined in `.../roles/*.tmdl`; changes should be validated with a report open and role testing.

## What an AI agent should and should not change
- Safe: small textual edits (labels/captions), color tweaks in theme JSON, comments in JSON, non-structural metadata updates, or documenting findings.
- Avoid: renaming tables/columns, changing relationships, or making large structural DAX/model changes unless the agent can also run and validate Power BI Desktop/Tabular Editor locally; those require a human-in-the-loop.

## Pull request guidance
- Keep changes small and focused (one area per PR). Include screenshots or a short validation checklist when the change affects visuals or model behavior.

## If you need to repackage or automate
- There is no packaging script in the repo. To automate packaging you can either:
  - Use Power BI Desktop manually to import the modified JSON and save as `*.pbix`/`*.pbit`.
  - Use third-party tools (Tabular Editor CLI, PowerBI API, or community pack/unpack scripts) — add new scripts to the repo and document them in README when you do so.

## Quick checklist for reviewers
- Confirm edits were made to the intended JSON/tmdl file and not to a `.bak` or duplicate.
- Validate locally by opening `*.pbit` in Power BI Desktop and exercising changed visuals/roles.

---
If anything here is unclear or you'd like the instructions to include automation examples (repackaging scripts or Tabular Editor CLI snippets), tell me which tooling you prefer and I'll update this file.
