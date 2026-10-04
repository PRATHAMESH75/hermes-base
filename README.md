# Hermes Base

A Clash-of-Clans-style dashboard for the Hermes issue autoresolver's pull requests on
[`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent).

Live page: https://prathamesh75.github.io/hermes-base/

## What's here

| File | Purpose |
|---|---|
| `index.html` | The page. It fetches `data/snapshot.json` and every battle-log file listed in `data/feed/index.json`. |
| `refresh_snapshot.py` | Rebuilds `data/` from the GitHub API with the `gh` CLI (stdlib only). |
| `seed-snapshot.json` | Hand-verified PR outcomes the refresh keeps: all 218 PRs as of Oct 4, 2026, with every closure classified. |
| `data/snapshot.json` | The current snapshot: every PR plus the last 14 days of battle-log events. |
| `data/feed/*.json` | Battle-log history: one file per month up to Sep 2026, then one per day. |
| `.github/workflows/refresh.yml` | Hourly refresh, commit and GitHub Pages deploy. |

## How the base maps to the data

- **Agents (buildings)** are the areas of the codebase, taken from each PR's conventional-commit
  scope and then its `comp/*` label: Core Sage (agent loop, providers), Gateway Herald
  (gateway and platforms), Desktop Artisan (desktop, TUI, dashboard), CLI Smith (CLI, config,
  install), Tool Tinkerer (tools, skills, MCP) and Cron Keeper (cron, kanban).
- **Purple pill:** open PRs. **Gold pill:** PRs that landed, either merged or salvaged
  (cherry-picked into a maintainer PR).
- **Troops marching to the town hall:** open PRs with active review threads.
- **Reviewers at the gate:** everyone who commented or reviewed in the battle-log window.
- Hover or tap a building to see what that agent has worked on, what it is addressing and
  what is in review. Click to pin the dossier.

## Refreshing the data

The workflow runs every hour at :17. It runs `refresh_snapshot.py` with the workflow's own
token (about 320 API calls, roughly 4 minutes), commits `data/` when it changed, and deploys
the site. Trigger it by hand from the Actions tab with **Run workflow**. A failed refresh keeps
the last committed data and still deploys.

To refresh locally where `gh auth status` succeeds:

```bash
python3 refresh_snapshot.py
python3 -m http.server 8000
```

Each refresh writes one `data/feed/<YYYY-MM-DD>.json` file for every whole day in its 14-day
window, so a day's battle log is kept after it ages out of the snapshot. The page merges the
history with the snapshot's feed, preferring exact-time events over the monthly files'
day-level entries for the same person, PR and day.
