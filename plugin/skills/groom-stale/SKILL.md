---
name: groom-stale
description: Weekly Sentry grooming pass. Closes stale unresolved issues, re-opens resolved issues that regressed, and assigns owners to high-impact unassigned issues. Use when the user says "groom Sentry", "clean up Sentry issues", "weekly Sentry triage", or when invoked autonomously from a routine.
---

# groom-stale

Three-pass triage that uses only the Sentry MCP — no git operations, no PRs. Safe for unattended runs.

## Inputs

- `SENTRY_ORG_SLUG` — required
- `SENTRY_DEFAULT_PROJECT_SLUG` — optional
- `STALE_AGE_DAYS` — default 30 (from `userConfig`)

If `$ARGUMENTS` contains `--dry-run`, log every decision but **do not call `update_issue`**. Default behavior is to apply changes.

## Workflow

All passes use `search_issues` with the `query` field (literal Sentry syntax). To pull deeper details on a single issue, use `get_sentry_resource` with the issue short_id as `resourceId`.

### Pass 1 — Close obviously stale

`search_issues` with `query: "is:unresolved lastSeen:-${STALE_AGE_DAYS}d sort:lastSeen"`, `limit: 50`. For each result where:

- Total events in the last `STALE_AGE_DAYS` days is **zero** (verify via `get_sentry_resource` event count), AND
- The issue is at least 60 days old (`firstSeen` < now − 60d)

call `update_issue` with `status: "ignored"`. Track these in a digest section labelled `Closed as stale`.

### Pass 2 — Re-open regressions

`search_issues` with `query: "is:resolved lastSeen:-7d sort:lastSeen"`, `limit: 50`. These are resolved issues seeing new events in the last week — almost always regressions. For each one:

- Confirm at least 5 new events since the resolve timestamp (visible in `get_sentry_resource`).
- Call `update_issue` with `status: "unresolved"`.

Track these as `Re-opened regressions`. Do **not** also assign — leave that for humans or for Pass 3.

### Pass 3 — Assign hot unassigned issues

`search_issues` with `query: "is:unresolved is:unassigned has:stack sort:freq"`, `limit: 20`. For each:

- Pull `get_sentry_resource` to read the top-of-stack file path.
- Run `git log -n 5 --format='%an|%ae' -- <file>` in the current repo to find the most recent authors. If the file doesn't exist in the current repo, skip — this skill is conservative and won't guess owners across repos.
- Pick the most recent author. Look them up in Sentry via `find_teams` / member lookup; if there's a matching member, call `update_issue` with `assignedTo: "user:<member-id>"`.
- If no match, fall back to the team that owns the project (from `find_projects`) via `assignedTo: "team:<team-slug>"`.

Track these as `Assigned`. Cap this pass at 20 assignments per run to avoid notification storms.

### Final — Post digest

Print a markdown digest to stdout:

```
# Sentry grooming digest — <ISO date>
**Org**: <org>  **Project**: <project or "all">

## Closed as stale (N)
- <SHORT-ID> <title> — last seen <relative time>

## Re-opened regressions (N)
- <SHORT-ID> <title> — <N> new events since resolve

## Assigned (N)
- <SHORT-ID> → @<handle> (suspected from <file>)

## Skipped (N)
- <SHORT-ID> — <reason>
```

If a Slack connector is enabled on the user's claude.ai account, also post the digest to the channel the user has configured (the routine prompt asks for the channel; for interactive use just print).

## Hard rules

- **Never delete issues.** `update_issue` to `ignored` is the strongest action.
- **Cap each pass at 50 issues** to prevent runaway calls.
- **Never assign to a user the skill cannot verify is a Sentry org member.**
- `--dry-run` produces the same digest but skips every write call.
