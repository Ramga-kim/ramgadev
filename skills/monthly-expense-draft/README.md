# monthly-expense-draft

Workflow skill for drafting a monthly Goworks expense approval from receipt images.

## What it does

- opens the Goworks expense form
- checks login state and waits for manual login when needed
- identifies the target receipt folder for the previous month
- extracts receipt date, amount, and vendor from images
- fills the expense table in bulk
- zips the receipt images and attaches the archive

## Prerequisites

- Playwright plugin enabled in the agent environment
- access to `https://www.goworks.co.kr/Approval/NewForm?templateid=2679`
- a local receipt archive organized by month

## Safety boundaries

- the skill must never click the final submit button
- credentials stay as placeholders unless the installer fills them in locally
- temporary upload copies must not be deleted before submit is complete and attachment presence is verified later

## Files

- `SKILL.md`: runtime instructions for the agent
- `reference/goworks-upload.md`: DOM and upload behavior notes kept out of the main skill body
