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
| `data/attention.json` | The needs-attention queue: what each open PR needs next, whether a bot can act on it, what is parked and what needs you. Zeus works from this file. |
| `attention.py` | Builds `data/attention.json` during the refresh (stdlib only). |
| `config/queue.json` | Reviewer trust tiers, check failures to ignore, the nudge switch and the open-PR cap. Edit by hand. |
| `config/parked.json` | PRs set aside on purpose, with a reason and the conditions that wake them up. Edited by hand and by Zeus. |
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
- **Needs attention:** rendered from `data/attention.json`. Each open PR gets one state, most
  urgent first: changes requested, conflicting, checks failing, reply owed, merge unknown, ready
  but quiet, FYI, AI note. Rows a bot has already handled at the current head commit, with no new
  comment since, are dimmed. Parked PRs are left out until a wake condition fires.
- **Ledger:** click a column header to sort; the Status column shows the merge state of open PRs.
- **Shareable views:** the pinned agent, ledger filter, search, sort and log filter are kept in the
  URL hash, for example `#ledger=action&sort=-updated`.
- A banner appears when the snapshot is more than 3 hours old, which means the refresh
  workflow is failing.

## The attention queue

`attention.py` reads each open PR's comments, reviews and check runs, then decides:

- **Who spoke last and whether it counts.** `config/queue.json` sorts reviewers into `maintainer`,
  `substantive` and `noise`. A comment from a noise login, or one that opens with "AI code review",
  is an AI note, never a reply owed. A comment that @-mentions other people but not the author
  (such as "@maintainer please review") is FYI. Unknown logins count as `default`.
- **Whether checks really failed.** Failing check runs matching `expectedFailures` (regexes, such as
  the fork arm64 Docker build) are ignored.
- **Whether a bot already acted.** The autoresolver and Zeus start every comment with
  `<!-- hermes-autotriage -->` and a line such as `<!-- action=rebased pr=123 head=abc1234 at=… -->`.
  An item is `actionable` only when there is no such marker, the head commit changed since, a
  non-noise human commented since, or (for conflicts) main kept moving after a rebase. An
  `action=escalated` marker sets `needsHuman` until you comment on the PR yourself.
- **Whether it is parked.** An entry in `config/parked.json` hides the PR until one of its `wake`
  conditions fires: `onNewHumanComment` (default true), `after` (an ISO date) or `paths` (exact
  file or directory paths on upstream `main`, no globs, checked for commits since `parkedAt`).

GitHub computes merge states lazily (often 20 s or more), so PRs that first read as `unknown` are
re-read in up to four rounds, about two minutes at most. Nudging quiet, mergeable PRs is off by default (`nudge.enabled`).

## Refreshing the data

The workflow runs every hour at :17, though GitHub delays or skips scheduled runs when it is
busy, so Zeus triggers it with `gh workflow run refresh.yml` whenever the queue is over 90 minutes
old. It runs `refresh_snapshot.py` with the workflow's own
token (about 400 API calls, roughly 5 minutes, retrying transient errors), commits `data/` when it changed, and deploys
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
