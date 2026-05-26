# Sharing this plugin

This repo is a Claude Code plugin marketplace. Anyone with Claude Code v2.1.143+ can install the workflows directly from GitHub — no npm publish, no Sentry-side review needed.

## To share

The repo is at `jshchnz/sentry-workflows-plugin`. Send testers the two commands in the next section.

## What testers run

```bash
# In any Claude Code session:
/plugin marketplace add jshchnz/sentry-workflows-plugin
/plugin install sentry-workflows@sentry-workflows

# On first MCP use, Claude Code opens a browser to mcp.sentry.dev for OAuth.
# Enter your Sentry org slug when Claude prompts (it's required userConfig).

# Then try a workflow:
/sentry-workflows:groom-stale --dry-run
```

## What testers need

- A Sentry account with access to the org they want to operate on. The OAuth flow at `mcp.sentry.dev` handles auth — no tokens to paste.
- `gh` CLI installed and authenticated, **only** if they want to test `/sentry-workflows:fix-issue` (which opens PRs). `groom-stale` doesn't need git.
- For routine testing: the Sentry connector enabled at https://claude.ai/customize/connectors (this is independent of the Claude Code plugin install — both paths use the same OAuth backend).

## Reporting back

Ask testers for:
- Was the OAuth flow smooth? (this is the riskiest UX leg)
- Did `groom-stale --dry-run` produce a digest that matched their Sentry backlog?
- Did `fix-issue` find a fixable issue, and was the resulting draft PR usable?
- Did the routine prompt printed by `/sentry-workflows:install-routines` work when pasted into `claude.ai/code/routines`?

## Updating

After pushing changes:

```bash
# Tester refreshes their local marketplace clone:
/plugin marketplace update sentry-workflows
```

Bump the `version` field in `plugin/.claude-plugin/plugin.json` so testers get the update notification. Without an explicit bump, Claude Code uses the commit SHA — every push counts as a new version.

## Limits during preview

- Marketplace name (`sentry-workflows`) and plugin name are unofficial. Don't list this on the Anthropic community marketplace yet — that's reserved for when it upstreams into `getsentry/sentry-mcp`.
- The `Sentry (community preview)` author string in the manifest is intentional — keep it until this lands officially so testers know it's not endorsed by Sentry yet.
