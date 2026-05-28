# sentry-workflows-plugin

> Community **preview** — not yet an official Sentry channel. Workflows live here while we shape them, with an eye toward upstreaming into the [official `sentry-mcp` plugin](https://github.com/getsentry/sentry-mcp/tree/main/plugins) once they've earned their place.

Opinionated Sentry workflows for [Claude Code](https://claude.com/claude-code) — built on top of the official Sentry MCP at `mcp.sentry.dev`.

## Quickstart

```bash
/plugin marketplace add jshchnz/sentry-workflows-plugin
/plugin install sentry-workflows@sentry-workflows
```

On first use Claude Code opens a browser to authorize `mcp.sentry.dev` (OAuth). No tokens to paste or rotate.

Then in any session:

```
/sentry-workflows:fix-issue
/sentry-workflows:groom-stale
/sentry-workflows:install-routines
```

See [`plugin/README.md`](plugin/README.md) for skill documentation and configuration.

## What's in the box

- **`/sentry-workflows:fix-issue`** — Sweep open Sentry issues, pick one that's fixable in the current repo, open a draft PR. Uses a scorer subagent fanned out in parallel and a fix-implementer subagent for the actual change.
- **`/sentry-workflows:groom-stale`** — Two-pass triage: close stale issues and re-open regressions. Caps each pass at 50, supports `--dry-run`. (Owner assignment was removed in 0.2.0 — see CHANGELOG.)
- **`/sentry-workflows:install-routines`** — Prints ready-to-paste prompts for Claude Routines so you can schedule the above or trigger them from GitHub / API.

## Why a remote MCP

This plugin connects to Sentry's official MCP at `https://mcp.sentry.dev/mcp` over HTTP. That means:

- **No tokens to manage** — OAuth handles auth, scoped to your Sentry user.
- **Same auth in routines** — Claude Routines authenticate Sentry via the claude.ai Sentry connector, which is the same backend. The workflow prompts work identically in cloud sessions.
- **Always current** — tool surface tracks `mcp.sentry.dev`, not a pinned npm version.

## Repository layout

```
.
├── .claude-plugin/marketplace.json   # marketplace manifest
└── plugin/                           # the plugin itself
    ├── .claude-plugin/plugin.json
    ├── .mcp.json
    ├── skills/
    ├── agents/
    ├── routines/                     # standalone routine prompts
    │   ├── fix-issue.routine.md
    │   └── groom-stale.routine.md
    └── README.md
```

## Local development

```bash
git clone https://github.com/jshchnz/sentry-workflows-plugin
cd sentry-workflows-plugin
claude --plugin-dir ./plugin
```

Validate the manifest before committing:

```bash
claude plugin validate ./plugin --strict
```

## Requirements

- Claude Code v2.1.143 or later (for `displayName`).
- A Sentry account with access to the org you want to operate on.
- The `gh` CLI installed and authenticated (for `fix-issue` only — `groom-stale` doesn't touch git).

## License

Apache-2.0. See [LICENSE](LICENSE).
