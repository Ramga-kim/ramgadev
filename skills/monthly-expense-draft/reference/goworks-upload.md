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

## Known select values

- `야근주말식대`: `81100`
- `개인`: environment-specific label or value should be verified once and then reused

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
- creating temporary `.ps1` files as a workaround for quoting or encoding issues
- calling `browser_click` or `browser_run_code` while a file chooser modal is already open
- using `browser_run_code` with top-level `document` access instead of wrapping DOM work in `page.evaluate`

## Attachment behavior

- final attachment artifact is a zip file that contains receipt PDFs
- image receipts should be converted to per-receipt PDFs before packaging
- source PDFs stay as PDFs, but their final attachment filenames must still be normalized to `yyyyMMdd.pdf` or `yyyyMMdd_NN.pdf`
- raw source filenames must not remain inside the final zip
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
- verify that the zip contents are normalized to `yyyyMMdd.pdf` or `yyyyMMdd_NN.pdf` before upload

## Efficiency notes

- copy the zip into an allowed upload path before the first upload attempt instead of failing once and retrying
- treat the allowed upload path as a hard precondition before opening the file chooser
- use `business.upload_staging_path` as the default allowed-root staging location instead of `%TEMP%` or `Downloads`
- avoid full-page snapshots after every step; reserve them for login uncertainty, DOM discovery, or final user-facing confirmation
- on Windows, prefer PowerShell-native file operations over `mv` or `cp` fallbacks for rename/copy steps
- treat totals, summary cells, and other readonly or auto-calculated fields as verification targets, not input targets
- after login redirects or page transitions, reacquire locators instead of trusting older element references
- for file upload, if the chooser closes after a failed attempt, click the upload button again before retrying `browser_file_upload`
- while the chooser is open, do not call any browser tool other than `browser_file_upload`
