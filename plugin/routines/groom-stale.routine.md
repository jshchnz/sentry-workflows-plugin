# Routine prompt — Sentry groom-stale

Paste this whole block into the **Instructions** box at https://claude.ai/code/routines. It does not touch git or open PRs — it only orchestrates Sentry write operations via the Sentry connector. Safe for fully autonomous weekly runs.

---

You are running as a Claude Routine. Perform a Sentry grooming pass: close obviously stale issues and re-open regressions.

## Configuration

- Sentry organization slug: `<ORG_SLUG>`
- Sentry project slug (or `all`): `<PROJECT_SLUG_OR_ALL>`
- Stale age (days): `30`
- Per-pass cap: `50` issues
- Dry run: `false` (set to `true` for the first run to preview decisions)

## Setup

Compute these once at the start of the run:

- `STALE_CUTOFF_ISO = (now − Stale age days)` formatted as `YYYY-MM-DDTHH:MM:SS` (no trailing `Z`).
- `AGE_CUTOFF_ISO   = (now − 60 days)` in the same format.
- Maintain three accumulators: `closed[]`, `reopened[]`, `errors[]`.

## Workflow

Both passes use the Sentry tool `search_issues` with literal Sentry syntax in the `query` field, `get_sentry_resource` (passing the issue short_id as `resourceId`) for deep details, and `search_events` for windowed event counts.

### Pass 0 — Preflight

1. Call `whoami`. If it errors, abort with `Sentry MCP is not connected.`
2. Call `find_projects(organizationSlug=<ORG_SLUG>)`. On 403, abort with `No access to org <ORG_SLUG>.`
3. If `<PROJECT_SLUG_OR_ALL>` is a slug (not `all`), confirm it appears in `find_projects`.

### Pass 1 — Close obviously stale

Sentry's date filters accept `-duration` ("within last N") or absolute ISO timestamps with `<`/`>`. There is no `+duration` shorthand for "older than"; use the ISO form.

Call `search_issues` with `query = "is:unresolved lastSeen:<{STALE_CUTOFF_ISO} firstSeen:<{AGE_CUTOFF_ISO}"` (substitute the computed ISO timestamps from Setup; the leading `<` is the Sentry comparison operator), `sort = "date"`, `limit = 50`. For each result:

- Call `search_events(organizationSlug=<ORG_SLUG>, dataset="errors", query="issue:<short_id>", statsPeriod="30d", fields=["count()"], limit=1)`. If the returned `count()` is non-zero, skip and append to `errors[]` with reason `unexpected-activity`.
- If `Dry run` is `true`, append to `closed[]` with `(dry-run; skipped)` and do **not** call `update_issue`.
- Otherwise call `update_issue(issueId=<short_id>, status="ignored", reason="Auto-closed by groom-stale: no events in 30d, first seen >60d ago")`. On error, append to `errors[]`. On success, append to `closed[]`.

### Pass 2 — Re-open regressions

Call `search_issues` with `query = "is:resolved lastSeen:-7d"`, `sort = "date"`, `limit = 50`. The `-7d` is correct here — it finds resolved issues with events arriving *after* the resolve. For each result:

- Pull `get_sentry_resource`. Find the most recent `activity[]` entry with `type: "set_resolved"` and take its `dateCreated` as `RESOLVE_TIME`. If none, append to `errors[]` with `no-resolve-timestamp` and skip.
- Call `search_events(organizationSlug=<ORG_SLUG>, dataset="errors", query="issue:<short_id> timestamp:>{RESOLVE_TIME}", statsPeriod="30d", fields=["count()"], limit=1)`. If `count()` < 5, skip (not a regression).
- If `Dry run` is `true`, append to `reopened[]` with `(dry-run; skipped)`.
- Otherwise call `update_issue(issueId=<short_id>, status="unresolved", reason="Auto-reopened by groom-stale: <N> events since resolve at <RESOLVE_TIME>")`. On error, append to `errors[]`. On success, append to `reopened[]`.

### Final — Print digest

```
# Sentry grooming digest — <ISO date>
Org: <org>  Project: <project or "all">  Dry-run: <true|false>

## Closed as stale (<N>)
- <SHORT-ID> <title> — last seen <relative time>[ (dry-run; skipped)]

## Re-opened regressions (<N>)
- <SHORT-ID> <title> — <N> events since resolve[ (dry-run; skipped)]

## Errors (<N>)
- <SHORT-ID or "(pass)"> — <reason>
```

Empty sections render the count `(0)` and the line `_None._`.

## Hard rules

- Never delete a Sentry issue. The strongest action is `update_issue` to `ignored`.
- Cap each pass at 50 issues.
- In dry-run mode, log every decision but do not call any `update_*` tools. Check the flag at each call site, not just at the top.
- Do not touch git, do not open PRs.
- Do not attempt user-level assignment — the Sentry MCP has no member-lookup tool.

## Trigger suggestions

- **Schedule**: weekly, Monday 08:00 local. Pair with your team's weekly triage meeting.
- **API**: also expose so on-call can trigger ad-hoc cleanups before incidents.
