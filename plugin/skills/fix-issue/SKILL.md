---
name: fix-issue
description: Find one Sentry issue you are confident you can fix in the current repository and open a draft pull request that fixes it. Use when the user says "fix something from Sentry", "find a Sentry issue to fix", "auto-fix a Sentry bug", or when invoked autonomously from a Claude routine. Requires the Sentry MCP and `gh` CLI to be available.
---

# fix-issue

End-to-end: pick one fixable Sentry issue, write the patch, push to a `claude/`-prefixed branch, and open a draft PR.

## Inputs

The plugin already configured these via `userConfig`; read them from the environment as `CLAUDE_PLUGIN_OPTION_*`:

- `SENTRY_ORG_SLUG` — required
- `SENTRY_DEFAULT_PROJECT_SLUG` — optional; if set, scope the issue search to it

Auth is handled by the remote MCP at `mcp.sentry.dev` via OAuth — on first use Claude Code opens a browser to authorize. Do not prompt the user for tokens.

If the user passed `$ARGUMENTS`, treat it as a free-form Sentry query filter to append (e.g. "javascript stacktrace.module:checkout/").

## Workflow

### 1. Confirm preconditions

- `gh auth status` must succeed. If not, stop and tell the user to run `gh auth login`.
- `git status` must show a clean working tree. If dirty, stop and ask the user to stash.
- Determine the GitHub `owner/repo` via `gh repo view --json nameWithOwner -q .nameWithOwner`.

### 2. Pull candidate issues

Call the Sentry MCP `search_issues` tool with:

- `organizationSlug`: `SENTRY_ORG_SLUG`
- `projectSlugOrId`: `SENTRY_DEFAULT_PROJECT_SLUG` (omit if unset)
- `query`: `is:unresolved is:unassigned has:stack sort:freq ${ARGUMENTS}` — literal Sentry-syntax filter; append any user filter
- `limit`: 10

(`search_issues` accepts either a `query` field with literal Sentry syntax OR a `naturalLanguageQuery` field. Always pass `query` here — it works whether or not the server has an `EMBEDDED_AGENT_PROVIDER` configured.)

If the call fails with an auth error, tell the user: "The Sentry MCP isn't authorized. Re-run `/plugin enable sentry-workflows` and complete the browser OAuth flow when it opens." and stop.

### 3. Score candidates in parallel

For each candidate, spawn the `issue-scorer` agent (one Agent call per candidate, all in one assistant message so they run in parallel). Each scorer call gets:

- The issue short_id, title, culprit, and platform
- A `get_sentry_resource` result for the issue (so the agent sees the top stack frames and breadcrumbs without re-fetching). Pass the issue short_id as `resourceId` and `SENTRY_ORG_SLUG` as `organizationSlug`.

Each scorer returns JSON `{score: 1-5, fixable: bool, reasoning: string, suspected_files: string[]}`.

### 4. Pick the winner

Filter for `fixable: true` and `score >= 4` AND at least one `suspected_files` entry that exists in the current repo (`ls` or `test -f`). If none qualify, post a one-paragraph summary explaining why each was skipped and exit cleanly. **Never widen the criteria silently.**

### 5. Implement the fix

Hand off to the `fix-implementer` agent with:

- The full issue payload (stack trace, breadcrumbs, tags, event count)
- The scorer's `suspected_files` and `reasoning`
- Branch name: `claude/sentry-fix-<issue-short-id-lowercased>`
- Repo `owner/repo`

The agent implements, runs the project's tests if it can identify them, commits, pushes, and opens a **draft** PR. The PR body must include:

- A link to the Sentry issue (`https://sentry.io/organizations/<org>/issues/<short-id>/`)
- A short root-cause explanation
- What was changed and why
- Test plan (commands run + result)
- A `Closes <issue-short-id>` line (informational; Sentry's GitHub integration links it)

### 6. Update Sentry

Call the Sentry MCP `update_issue` tool to assign the issue to the `whoami` result (the authenticated user). Do not resolve the issue — wait for the PR to merge.

### 7. Report

Print a one-line summary: `<issue-short-id> → <PR URL> (branch: <branch>)`.

## Hard rules

- **Never push to `main`, `master`, or any branch that does not start with `claude/`.** Routines enforce this server-side; respect it locally too.
- **Never run `git push --force` or `--no-verify`.**
- **Never resolve a Sentry issue from this skill** — only `update_issue` to assign. Resolution happens when the PR merges.
- If the issue's stack trace references files outside the current repo, mark it unfixable and move on. Do not invent file paths.
- If you cannot find a clean fix within the scorer's `suspected_files`, abort and report — do not refactor broadly to make the fix work.

## Optional Seer hint

If after step 3 every candidate scored 3 or below and the user invoked the skill with `--use-seer` in `$ARGUMENTS`, call `analyze_issue_with_seer` on the top-frequency issue, then re-run the scorer with Seer's suspected files added to the input. Seer takes 2–5 minutes; warn the user before calling it.
