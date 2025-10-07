Archive leftover backups and `(2)` exports

This PR archives leftover backup files and previously-exported `(2)` table files into `Staff Card Clean v2.SemanticModel/archive/` to reduce confusion and keep the active model tidy. No active `.tmdl` files were deleted; files are moved non-destructively.

Files moved: (high level)
- `*.tmdl.bak` and `*.json.bak` files
- Any files with `(2)` in their filename that were remaining

Reviewer checklist
- Confirm `Staff Card Clean v2.SemanticModel/archive/` contains only backup or historical files.
- Confirm canonical `.tmdl` files are present under `Staff Card Clean v2.SemanticModel/definition/tables/` (no `(2)` suffixes remain on active files).
- Open `Staff Card Clean v2.SemanticModel/definition/model.tmdl` in Tabular Editor and confirm referenced table names match the canonical `.tmdl` files.
- Validate in Power BI Desktop that primary report pages load and visuals are not broken.

If something looks wrong, comment and request a targeted revert of the affected file(s) from `Staff Card Clean v2.SemanticModel/archive/`.

This change is non-destructive — files are preserved in the archive folder.
