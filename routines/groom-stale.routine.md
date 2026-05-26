# Routine prompt — Sentry groom-stale

Paste this whole block into the **Instructions** box at https://claude.ai/code/routines. It does not touch git or open PRs — it only orchestrates Sentry write operations via the Sentry connector. Safe for fully autonomous weekly runs.

---

You are running as a Claude Routine. Perform a Sentry grooming pass: close obviously stale issues, re-open regressions, and assign owners to high-impact unassigned issues.

## Configuration

- Sentry organization slug: `<ORG_SLUG>`
- Sentry project slug (or `all`): `<PROJECT_SLUG_OR_ALL>`
- Stale age (days): `30`
- Per-pass cap: `50` issues
- Dry run: `false` (set to `true` for the first run to preview decisions)

## Workflow

All three passes use the Sentry tool `search_issues` with literal Sentry syntax in the `query` field, and `get_sentry_resource` (passing the issue short_id as `resourceId`) for deep details.

### Pass 1 — Close obviously stale

Call `search_issues` with `query = "is:unresolved lastSeen:-30d sort:lastSeen"`, `limit = 50`. For each result:

- Call `get_sentry_resource` to read the event count in the last 30 days and the `firstSeen` timestamp.
- If events-in-window is 0 AND the issue is at least 60 days old: call `update_issue` with `status = "ignored"` (unless dry-run).
- Track in a digest under `Closed as stale`.

### Pass 2 — Re-open regressions

Call `search_issues` with `query = "is:resolved lastSeen:-7d sort:lastSeen"`, `limit = 50`. For each result:

- Pull `get_sentry_resource`. If there are at least 5 events since the most recent resolve event, call `update_issue` with `status = "unresolved"` (unless dry-run).
- Track under `Re-opened regressions`. Do not assign in this pass.

### Pass 3 — Assign hot unassigned issues

Call `search_issues` with `query = "is:unresolved is:unassigned has:stack sort:freq"`, `limit = 20`. For each:

- Pull `get_sentry_resource`. Take the top-of-stack file path.
- If the file exists in the cloned repo: `git log -n 5 --format='%an|%ae' -- <file>`. Pick the most recent author.
- Use Sentry's `find_teams` / member-lookup to find that person as an org member.
- If matched: `update_issue` with `assignedTo = "user:<id>"`.
- Otherwise: fall back to the project's team via `find_projects` → `assignedTo = "team:<team-slug>"`.
- If you cannot identify any owner, skip and log a reason.
- Track under `Assigned`.

### Final — Print digest

Print a markdown digest to the run output:

```
# Sentry grooming digest — <ISO date>
Org: <org>  Project: <project or "all">  Dry-run: <true|false>

## Closed as stale (<N>)
- <SHORT-ID> <title> — last seen <relative time>

## Re-opened regressions (<N>)
- <SHORT-ID> <title> — <N> new events since resolve

## Assigned (<N>)
- <SHORT-ID> → @<handle> (suspected from <file>)

## Skipped (<N>)
- <SHORT-ID> — <reason>
```

## Hard rules

- Never delete a Sentry issue. The strongest action is `update_issue` to `ignored`.
- Never assign to a user who is not a verified Sentry org member.
- Cap each pass at 50 issues.
- In dry-run mode, log every decision but do not call any `update_*` tools.
- Do not touch git, do not open PRs.

## Trigger suggestions

- **Schedule**: weekly, Monday 08:00 local. Pair with your team's weekly triage meeting.
- **API**: also expose so on-call can trigger ad-hoc cleanups before incidents.
