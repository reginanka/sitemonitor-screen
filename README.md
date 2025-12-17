# sitemonitor-screen

Automated website visual change monitoring using GitHub Actions, Playwright, and Telegram notifications.

## Features

- Simulates real user visits through Playwright's headless Chromium browser with full page load until `networkidle`
- Extracts key text content (warnings and update dates) from HTML for notifications
- Captures screenshots of a specific page region between two text markers and compares MD5 hashes with the previous state
- Sends notifications to a Telegram work channel with text, screenshot, and a "Subscribe" button when changes are detected
- Sends detailed technical logs to a separate log channel after each run (successful or failed)

> Current average GitHub Actions runtime for this workflow is about 46 seconds, which is acceptable for a 5-minute cron schedule and leaves enough room for retries or temporary slowdowns.

## Why Screenshot + Hash Comparison

The target website does not provide stable text or images that can be reliably parsed and compared between runs. The area of interest consists of many dynamic blocks with colors and variables loaded from external sources, making DOM/text-based comparison fragile even when actual content hasn't changed.

The chosen approach:

- Full page render in Playwright browser
- Identifies visual region boundaries using static text anchors (e.g., "Дата оновлення інформації" and last occurrence of "робіт")
- Captures a screenshot of this zone and calculates MD5 hash of the image bytes for comparison with `last_hash.json`

This provides stable "visual regression" monitoring: the tool reacts only to actual visual changes, not internal technical markup modifications.

## Why Two Playwright Sessions

The script implements two independent Playwright sessions with different responsibilities:

### 1. Semantic pass (`get_schedule_content`)

- Loads the page, retrieves HTML, and extracts required text (warnings, dates, metadata) using BeautifulSoup
- Provides clean parsing logic without dependencies on element coordinates or scroll operations

### 2. Visual pass (`take_screenshot_between_elements`)

- Separately opens the page, finds anchor elements via Playwright locators, and constructs a screenshot rectangle between them
- Runs in an isolated browser lifecycle to ensure previous actions don't affect layout and positioning during screenshot capture

**This separation:**

- Reduces code coupling: each function solves its own task (content vs. visual) and can evolve independently
- Makes behavior more deterministic in CI environments, where side effects from one scenario can cause flaky failures in visual tests
- Provides clear architecture instead of a "monolithic" scenario with hard-to-debug mixed logic

## Project Structure

- **`monitor.py`** – main monitoring logic:
  - content retrieval (`get_schedule_content`)
  - screenshot capture and MD5 hash calculation (`take_screenshot_between_elements`)
  - state persistence in `last_hash.json`
  - Telegram channel and log channel messaging
  - global exception handler for convenient debugging

- **`last_hash.json`** – stores previous state: message hash, text, date, screenshot hash, and timestamp of last successful run

- **`.github/workflows/monitor.yml`** – GitHub Actions workflow:
  - `ubuntu-latest` environment with Python 3.11, dependency installation, and Playwright setup
  - scheduled runs (cron with minimum 5-minute intervals per GitHub Actions limitations) and manual trigger via `workflow_dispatch`
  - commits updated `last_hash.json` back to repository, skipping re-trigger with `[skip ci]` in commit message

## Setup

### 1. GitHub Secrets

Create the following secrets in your repository (`Settings` → `Secrets and variables` → `Actions`):

- `URL` – target page for monitoring
- `TELEGRAM_BOT_TOKEN` – Telegram bot token
- `TELEGRAM_CHANNEL_ID` – main work channel for notifications
- `TELEGRAM_LOG_CHANNEL_ID` – log channel for technical logs
- `SUBSCRIBE` – URL or deep link for the subscribe button
- `PAT_TOKEN` – personal GitHub token with push permissions to this repository

  
### 2. Local Run (Optional)
```bash
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows
pip install -r requirements.txt
playwright install chromium
python monitor.py

```
For local runs, environment variables (`URL`, Telegram tokens, etc.) can be passed via `.env` file or export/set in terminal.

## License

This project is licensed under the MIT License – see the [LICENSE](LICENSE) file for details.
