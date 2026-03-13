# Goworks Upload Reference

## Form URL

- `https://www.goworks.co.kr/Approval/NewForm?templateid=2679`

## Table notes

- main table id: `Table3`
- row cell id pattern: `{row}_{column}`
- common columns:
  - `_1`: date
  - `_2`: amount
  - `_3`: vendor
  - `_5`: account item select
  - `_8`: expense type select

## Bulk fill pattern

- add rows first in one batched browser action
- set text inputs in a single browser-side evaluation
- set select fields in a batched browser action first; fall back to Playwright `selectOption` only if Goworks change handling fails
- trigger blur on amount inputs so totals recalculate
- snapshot `ref` values are not DOM attributes; do not turn them into CSS selectors inside `page.evaluate` or `browser_run_code`

## Mandatory strategy

1. add rows in one batched browser action
2. fill text inputs by row and cell index inside `#Table3`
3. set select fields in one batched browser action when possible
4. verify with targeted checks instead of rediscovering the DOM
5. use per-field browser actions only as fallback after the batched strategy fails

## Avoid

- trying CSS selectors like `#1_1` for numeric-leading ids
- turning snapshot refs into DOM selectors
- using `select.value = '...'` as the primary strategy for Goworks select fields
- re-inspecting the whole table structure on every retry once the verified pattern is known
- using repeated per-row `browser_click` or repeated per-row `browser_select_option` as the default strategy

## Attachment behavior

- final attachment artifact is a zip file that contains receipt PDFs
- image receipts should be converted to per-receipt PDFs before packaging
- source PDFs stay as PDFs; a single PDF may represent one or multiple receipts at extraction time
- Goworks stores selected files in client-side `insertFileList`
- actual upload occurs only after the approval submit flow calls `PerformUpload`
- absence of an immediate network upload request after file selection is normal
- deleting the source file before final submit breaks lazy file access and can drop the attachment

## Validation points

- attachment filename is visible in the form
- attachment size is visible in the form
- `insertFileList` contains `File` instances with `hasFile: true`
- typical structure is `{ file: File, ... }`, so validation should read `item.file.name` and `item.file.size`
- prefer targeted field checks or `page.evaluate` over full-page snapshots when validating filled rows or attachment state

## Efficiency notes

- copy the zip into an allowed upload path before the first upload attempt instead of failing once and retrying
- treat the allowed upload path as a hard precondition before opening the file chooser
- avoid full-page snapshots after every step; reserve them for login uncertainty, DOM discovery, or final user-facing confirmation
- on Windows, prefer PowerShell-native file operations over `mv` or `cp` fallbacks for rename/copy steps
- treat totals, summary cells, and other readonly or auto-calculated fields as verification targets, not input targets
- after login redirects or page transitions, reacquire locators instead of trusting older element references
- for file upload, if the chooser closes after a failed attempt, click the upload button again before retrying `browser_file_upload`
