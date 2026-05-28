---
name: groom-stale
description: Weekly Sentry grooming pass. Closes stale unresolved issues and re-opens resolved issues that regressed. Use when the user says "groom Sentry", "clean up Sentry issues", "weekly Sentry triage", or when invoked autonomously from a routine.
---

# groom-stale

Two-pass triage that uses only the Sentry MCP — no git operations, no PRs. Safe for unattended runs.

## Inputs

- `SENTRY_ORG_SLUG` — required
- `SENTRY_DEFAULT_PROJECT_SLUG` — optional
- `STALE_AGE_DAYS` — default 30 (from `userConfig`)

If `$ARGUMENTS` contains `--dry-run`, log every decision but **do not call `update_issue`**. Default behavior is to apply changes.

## Workflow

Compute these once at the start of the run and reuse in every pass:

- `DRY_RUN = "--dry-run" in $ARGUMENTS`
- `STALE_CUTOFF_ISO = (now − STALE_AGE_DAYS days) formatted as YYYY-MM-DDTHH:MM:SS` (no trailing `Z`)
- `AGE_CUTOFF_ISO   = (now − 60 days)` in the same format
- Maintain three accumulators: `closed[]`, `reopened[]`, `errors[]`. Append to them as you go; the digest is built from these at the end.

All passes use `search_issues` with the literal `query` field. To pull deeper details on a single issue, use `get_sentry_resource` with the issue short_id as `resourceId`. To count events in a window, use `search_events` (the documented tool for counts).

### Pass 0 — Preflight

Before touching any data:

1. Call `whoami`. If it errors, abort the run with `Sentry MCP is not connected — re-run \`/plugin enable sentry-workflows\` and complete OAuth.` and exit cleanly (digest with `Errors: 1`).
2. Call `find_projects(organizationSlug=SENTRY_ORG_SLUG)`. On 403, abort with `The authenticated user has no access to org \`<SENTRY_ORG_SLUG>\` — re-authenticate or change SENTRY_ORG_SLUG.` and exit.
3. If `SENTRY_DEFAULT_PROJECT_SLUG` is set, confirm it appears in the `find_projects` result. If not, abort.

### Pass 1 — Close obviously stale

Note on syntax: Sentry's date filters accept either `-duration` (e.g. `-30d` = "within the last 30 days") or an absolute ISO 8601 timestamp with `<` / `>`. There is **no** `+duration` shorthand for "older than." To find issues that have **not** been seen recently, use the ISO form.

Call `search_issues` with:

- `query = "is:unresolved lastSeen:<${STALE_CUTOFF_ISO} firstSeen:<${AGE_CUTOFF_ISO}"`
- `sort = "date"`
- `limit = 50`

This returns unresolved issues whose last event is older than `STALE_AGE_DAYS` and whose first event is older than 60 days (so we don't ignore issues that are simply brand-new and idle).

For each result:

1. Call `search_events(organizationSlug=SENTRY_ORG_SLUG, dataset="errors", query="issue:<short_id>", statsPeriod="${STALE_AGE_DAYS}d", fields=["count()"], limit=1)`. Read the `count()` value from the single returned row. If it is **not** zero, the index lagged between calls — skip this issue and append to `errors[]` with reason `unexpected-activity`.
2. If `DRY_RUN`, append `<short_id>` to `closed[]` with status `(dry-run; skipped)` and do not call `update_issue`.
3. Otherwise call `update_issue(issueId=<short_id>, status="ignored", reason="Auto-closed by groom-stale: no events in ${STALE_AGE_DAYS}d, first seen >60d ago")`. On error, append to `errors[]` and continue. On success, append to `closed[]`.

### Pass 2 — Re-open regressions

Call `search_issues` with:

- `query = "is:resolved lastSeen:-7d"`
- `sort = "date"`
- `limit = 50`

The `-7d` here is correct: it finds resolved issues whose most recent event is *within* the last week (i.e., events arrived *after* the resolve), which is the regression signal. Contrast with Pass 1, where the goal was the opposite (issues with **no** recent events) — that's why Pass 1 uses an absolute ISO cutoff.

For each result:

1. Call `get_sentry_resource(resourceType="issue", organizationSlug=SENTRY_ORG_SLUG, resourceId=<short_id>)`. Inspect the activity feed (the field is typically `activity[]`, with entries of `type: "set_resolved"` carrying a `dateCreated`). Take the most recent `set_resolved` timestamp as `RESOLVE_TIME`. If no `set_resolved` entry exists, append to `errors[]` with `reason = no-resolve-timestamp` and skip.
2. Call `search_events(organizationSlug=SENTRY_ORG_SLUG, dataset="errors", query="issue:<short_id> timestamp:>${RESOLVE_TIME}", statsPeriod="30d", fields=["count()"], limit=1)`. Read the `count()` value. If `< 5`, skip (too noisy to call a regression — not an error). (Pin `statsPeriod` to `30d` so the embedded agent doesn't default to a shorter window that pre-trims the absolute timestamp filter.)
3. If `DRY_RUN`, append to `reopened[]` with status `(dry-run; skipped)`.
4. Otherwise call `update_issue(issueId=<short_id>, status="unresolved", reason="Auto-reopened by groom-stale: <N> events since resolve at <RESOLVE_TIME>")`. On error, append to `errors[]` and continue. On success, append to `reopened[]`.

Do **not** also assign — assignment is out of scope for this skill (the Sentry MCP has no member-lookup tool).

### Idempotency

`update_issue` is naturally idempotent — setting `status: ignored` on an already-ignored issue, or `status: unresolved` on an already-unresolved issue, is a no-op. Re-runs are safe. Don't pre-check the status; just call.

### Final — Print digest

Print this exact structure to stdout. Sections are always present even when empty so downstream consumers can parse a stable schema.

```
# Sentry grooming digest — <ISO date>
Org: <SENTRY_ORG_SLUG>  Project: <SENTRY_DEFAULT_PROJECT_SLUG or "all">  Dry-run: <true|false>

## Closed as stale (<len(closed)>)
- <SHORT-ID> <title> — last seen <relative time>[ (dry-run; skipped)]

## Re-opened regressions (<len(reopened)>)
- <SHORT-ID> <title> — <N> events since resolve[ (dry-run; skipped)]

## Errors (<len(errors)>)
- <SHORT-ID or "(pass)"> — <reason>
```

If any list is empty, render the heading with the `(0)` count and the literal line `_None._` under it.

## Hard rules

- **Never delete issues.** `update_issue` to `ignored` is the strongest action.
- **Cap each pass at 50 issues.** Enforced via `search_issues(limit: 50)`.
- `--dry-run` produces the same digest but skips every write call. The `DRY_RUN` flag is checked at every `update_issue` call site — not at the top of the workflow.
- If a Sentry call fails, append to `errors[]` and continue. A pass-level fatal error (e.g., 403 from the initial `search_issues`) aborts that pass; subsequent passes still run.
- Do not attempt user-level assignment. The Sentry MCP exposes no member-lookup tool; team-only assignment is deferred to a future version.
