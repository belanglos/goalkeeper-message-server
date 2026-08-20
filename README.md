# goalkeeper-message-server

Automates a weekly "goalkeeper" rotation: publishes the current person on
duty as JSON via GitHub Pages (for a Slack Workflow Builder workflow to fetch)
and posts an announcement to a Microsoft Teams channel via incoming webhook.

## How it works

1. [`goalkeeper-schedule.json`](./goalkeeper-schedule.json) maps ISO year →
   ISO week number → name:
   ```json
   {
     "2026": { "1": "Alice", "2": "Bob" },
     "2027": { "1": "Carol", "2": "Alice" }
   }
   ```
2. The [`goalkeeper` workflow](./.github/workflows/goalkeeper.yml) runs every
   Monday at 07:00 UTC (and can be triggered manually via
   "Run workflow" / `workflow_dispatch`). It:
   - Computes the current ISO week number and ISO week-year (Monday = start
     of week, per ISO 8601) using plain Node.js — no npm packages required.
   - Looks up `goalkeeper-schedule.json[isoYear][isoWeek]`.
   - Writes the result to [`docs/current.json`](./docs/current.json), e.g.
     `{ "year": 2026, "week": 34, "name": "Alice" }`.
   - Commits and pushes the change to `main` as `github-actions[bot]` (a
     no-op if the file didn't change).
   - Posts a message to Microsoft Teams via the incoming webhook URL stored
     in the `TEAMS_WEBHOOK_URL` repository secret, e.g.
     `⚽ This week's goalkeeper is: Alice (week 34)`.
3. GitHub Pages serves the `docs/` folder, so `docs/current.json` is reachable
   at a public URL. A Slack Workflow Builder workflow calls that URL to
   announce the current goalkeeper.

## Updating the schedule

Edit `goalkeeper-schedule.json` directly — add years/weeks as needed, or swap
in real names for the `Alice` / `Bob` / `Carol` placeholders. Week keys are
ISO week numbers as strings (`"1"`–`"52"`, or `"53"` in long ISO years, e.g.
2026). If a given year/week isn't in the schedule, the workflow writes
`"name": "Unassigned"` instead of failing.

## Manual setup required (one-time)

These can't be done via code in this repo — an admin needs to do them once
in the GitHub UI:

1. **Enable GitHub Pages**: repo Settings → Pages → Source: "Deploy from a
   branch" → Branch: `main`, folder `/docs` → Save. Pages will then serve
   `docs/current.json` at `https://<org-or-user>.github.io/<repo>/current.json`.
2. **Confirm Actions can push to the repo**: repo Settings → Actions →
   General → "Workflow permissions" → select "Read and write permissions".
   The workflow itself requests `contents: write`, but this repo-level
   setting must allow it too, or the push step will fail with a permissions
   error.
3. **Point the Slack Workflow Builder workflow** at the Pages URL from step 1
   (an HTTP request / webhook step reading `current.json`) to surface the
   `name` field in your Slack message.
4. **Add the Teams webhook secret**: create an incoming webhook for the
   target Teams channel (Teams → channel → Connectors/Workflows → "Incoming
   Webhook"), then add its URL as a repo secret: Settings → Secrets and
   variables → Actions → New repository secret → name it
   `TEAMS_WEBHOOK_URL`. Without this secret the notify step will fail.
