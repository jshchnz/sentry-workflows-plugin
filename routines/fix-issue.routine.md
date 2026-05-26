# Routine prompt — Sentry fix-issue

Paste this whole block into the **Instructions** box at https://claude.ai/code/routines. It is self-contained and does not depend on the plugin being installed in the cloud session — it relies on the Sentry connector enabled on your claude.ai account and the `gh` CLI available in the Default environment.

---

You are running as a Claude Routine. Sweep open Sentry issues, pick one with high confidence it is fixable in this repository, implement the fix, and open a **draft** pull request.

## Configuration (edit these before saving the routine)

- Sentry organization slug: `<ORG_SLUG>`
- Sentry project slug (or `all`): `<PROJECT_SLUG_OR_ALL>`
- Maximum candidates to score: 10
- Minimum score to fix: 4 (out of 5)
- Optional Sentry query filter to AND into the search: `<EXTRA_FILTER_OR_EMPTY>`

## Workflow

1. Confirm `gh auth status` and a clean working tree. If either fails, stop and explain in the run summary.
2. Get the repo's `owner/repo` via `gh repo view --json nameWithOwner -q .nameWithOwner`.
3. Call the Sentry tool `search_issues` with:
   - `organizationSlug` = the org slug above
   - `projectSlugOrId` = the project slug (omit if `all`)
   - `query` = `is:unresolved is:unassigned has:stack sort:freq <EXTRA_FILTER_OR_EMPTY>` (literal Sentry syntax)
   - `limit` = 10
4. For each issue, call `get_sentry_resource` with `resourceId` set to the issue short_id to read the top 10 stack frames, breadcrumbs, and event count.
5. For each candidate, score in parallel (one subagent per candidate). Each scorer returns JSON `{score, fixable, reasoning, suspected_files}` and is read-only — it must verify suspected files exist in this repo via `ls`.
6. Filter for `fixable: true AND score >= 4 AND at least one suspected_file exists in this repo`. If none qualify, write a short report explaining each skip and stop.
7. For the winner, implement the fix:
   - `git checkout -b claude/sentry-fix-<short-id-lowercased>`
   - Edit the smallest set of files needed (≤ 2). Prefer guards/null-checks over refactors.
   - Run the project's test command (probe `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`). If your change breaks tests, revert and stop.
   - Commit with message `fix: <one-line> (<SHORT-ID>)` and a `Closes <SHORT-ID>` line.
   - `git push -u origin claude/sentry-fix-<short-id-lowercased>`
   - `gh pr create --draft --title "fix: <summary> (<SHORT-ID>)" --body "<body>" --base <default-branch>` with a body that links the Sentry issue and lists the test command + result.
8. Call Sentry `update_issue` to assign the issue to the authenticated user (from `whoami`). Do NOT resolve it.
9. Print one final line: `<SHORT-ID> → <PR URL>`.

## Hard rules

- Never push to a branch whose name does not start with `claude/`. Cloud routines enforce this; respect it locally too.
- Never resolve a Sentry issue. Only assign.
- If suspected files are outside this repo (`node_modules`, `vendor/`, paths that don't exist), mark the issue unfixable.
- If the fix expands beyond 2 files, abort with `too-broad` and report.
- Never run `git push --force`, `--no-verify`, or edit lockfiles / generated artifacts.
- Stop on the first error rather than retrying with looser criteria.

## Trigger suggestions

- **Schedule**: weekdays at 09:00 local. Pairs well with a morning review.
- **GitHub**: `pull_request.opened` filtered to label `sentry-fix` if you want a manual lever (open an empty PR labelled `sentry-fix`, the routine fires and proposes a real fix).
- **API**: wire to your alerting tool — POST the alert body as the `text` field, the routine picks the right issue from context.
