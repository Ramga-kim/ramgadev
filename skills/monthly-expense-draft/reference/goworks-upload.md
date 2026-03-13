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

- add rows first
- set text inputs in a single browser-side evaluation
- use Playwright `selectOption` for select fields
- trigger blur on amount inputs so totals recalculate
- snapshot `ref` values are not DOM attributes; do not turn them into CSS selectors inside `page.evaluate` or `browser_run_code`

## Attachment behavior

- Goworks stores selected files in client-side `insertFileList`
- actual upload occurs only after the approval submit flow calls `PerformUpload`
- absence of an immediate network upload request after file selection is normal
- deleting the source file before final submit breaks lazy file access and can drop the attachment

## Validation points

- attachment filename is visible in the form
- attachment size is visible in the form
- `insertFileList` contains `File` instances with `hasFile: true`
- prefer targeted field checks or `page.evaluate` over full-page snapshots when validating filled rows or attachment state

## Efficiency notes

- copy the zip into an allowed upload path before the first upload attempt instead of failing once and retrying
- avoid full-page snapshots after every step; reserve them for login uncertainty, DOM discovery, or final user-facing confirmation
- on Windows, prefer PowerShell-native file operations over `mv` or `cp` fallbacks for rename/copy steps
- treat totals, summary cells, and other readonly or auto-calculated fields as verification targets, not input targets
- after login redirects or page transitions, reacquire locators instead of trusting older element references
- for file upload, if the chooser closes after a failed attempt, click the upload button again before retrying `browser_file_upload`
