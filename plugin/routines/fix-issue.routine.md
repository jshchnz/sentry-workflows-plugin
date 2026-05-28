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
5. For each candidate, score in parallel (one subagent per candidate). Each scorer must:
   - Normalize stack-frame paths by stripping bundler prefixes (`webpack:///./`, `webpack-internal:///`, `app:///`, `~/`, `/_next/static/chunks/`) before checking existence.
   - Exclude paths under `node_modules/`, `.venv/`, `vendor/`, `.next/`, `dist/`, `build/`, `out/`, `target/`, `__pycache__/`, `.tox/`, `.gradle/`, `bin/`, `obj/`, and any path matched by `git check-ignore`.
   - Return JSON `{score, fixable, reasoning, suspected_files}`.
6. Filter for `fixable: true AND score >= 4 AND at least one suspected_file exists in this repo`. If none qualify, write a short report explaining each skip and stop.
7. **Branch preflight.** Let `BRANCH = claude/sentry-fix-<short-id-lowercased>`. Run `git show-ref --verify --quiet refs/heads/${BRANCH}`. If it exists, run `gh pr list --head ${BRANCH} --state all --json url,state --jq '.[0]'`:
   - If a PR exists, report `<SHORT-ID> → PR already open: <url>` and stop.
   - If no PR exists, stop with `Local branch ${BRANCH} exists with no PR — delete and retry.` Do not auto-delete.
8. For the winner, implement the fix:
   - `git checkout -b ${BRANCH}`
   - **Capture baseline.** Detect the test command (probe `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`). Run it once on the clean branch to record `BASELINE_FAILURES`.
   - Edit the smallest set of files needed (≤ 2). Prefer guards/null-checks over refactors.
   - Run the test command again. Compute `NEW_FAILURES = current − BASELINE_FAILURES`. If non-empty after one repair attempt, revert with `git restore .` and stop.
   - Commit with message `fix: <one-line> (<SHORT-ID>)` and a `Closes <SHORT-ID>` line.
   - `git push -u origin ${BRANCH}`
   - `gh pr create --draft --title "fix: <summary> (<SHORT-ID>)" --body "<body>" --base <default-branch>` with a body that links the Sentry issue, lists the test command + result, and notes any pre-existing failures.
9. Call Sentry `update_issue` to assign the issue to the authenticated user (from `whoami`). Do NOT resolve it.
10. Print one final line: `<SHORT-ID> → <PR URL>`.

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
