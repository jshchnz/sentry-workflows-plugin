# sentry-workflows — Claude Code plugin (preview)

Opinionated Sentry workflows that ride on top of Sentry's official MCP at `mcp.sentry.dev`.

| Skill                                  | What it does                                                                                |
| -------------------------------------- | ------------------------------------------------------------------------------------------- |
| `/sentry-workflows:fix-issue`          | Picks one fixable open Sentry issue in this repo and opens a **draft** PR with the fix.    |
| `/sentry-workflows:groom-stale`        | Weekly grooming: close stale, re-open regressions, assign hot unassigned issues.           |
| `/sentry-workflows:install-routines`   | Prints copy-paste routine prompts so you can run the above on a schedule or GitHub trigger. |

## Install

```bash
/plugin marketplace add jshchnz/sentry-workflows-plugin
/plugin install sentry-workflows@sentry-workflows
```

The first time the Sentry MCP is touched in a session, Claude Code opens a browser to `mcp.sentry.dev` for OAuth. Approve once, you're done — no tokens to paste or rotate.

Claude Code will also prompt for:

- `SENTRY_ORG_SLUG` — e.g. `sentry`. Required.
- `SENTRY_DEFAULT_PROJECT_SLUG` — optional, to scope workflows to a single project.
- `STALE_AGE_DAYS` — default 30.

## Use

Run a skill from any session:

```
/sentry-workflows:fix-issue
/sentry-workflows:groom-stale
/sentry-workflows:groom-stale --dry-run
```

The `fix-issue` skill requires the `gh` CLI installed and authenticated (`gh auth login`). It will never push to a branch that doesn't start with `claude/`, never force-push, and never resolve a Sentry issue.

## Run on a schedule with Claude Routines

1. Enable the Sentry connector at https://claude.ai/customize/connectors.
2. From any CLI session, run `/sentry-workflows:install-routines` to print routine prompts you can paste into https://claude.ai/code/routines.
3. Recommended cadence:
   - `fix-issue`: weekdays at 09:00 OR `pull_request.opened` with label `sentry-fix`.
   - `groom-stale`: weekly, Monday 08:00.

See [`../routines/`](../routines/) for the standalone prompt files.

## Architecture

```
plugin/
├── .claude-plugin/plugin.json   # manifest + userConfig (no secrets)
├── .mcp.json                    # connects to https://mcp.sentry.dev/mcp via HTTP+OAuth
├── skills/
│   ├── fix-issue/
│   ├── groom-stale/
│   └── install-routines/
└── agents/
    ├── issue-scorer.md          # read-only fixability scorer
    └── fix-implementer.md       # writes the change, opens the PR
```

`fix-issue` fans out one `issue-scorer` subagent per candidate (in parallel), then hands the winner to `fix-implementer`.

## Hard guardrails

- All git work happens on `claude/`-prefixed branches.
- PRs are always opened as **drafts** for human review.
- `groom-stale` caps each pass at 50 issues, never deletes, only assigns to verified org members.
- `--dry-run` is supported on `groom-stale` for safe first runs.

## Why a remote MCP?

Two reasons:

1. **No token management.** OAuth via `mcp.sentry.dev` scopes to your Sentry user and stays current with whatever permissions you have.
2. **Routine compatibility.** Claude Routines authenticate Sentry through the same backend via the claude.ai connector. The routine prompts in `../routines/` work without any plugin install in the cloud session.

## License

Apache-2.0. See [LICENSE](../LICENSE).
