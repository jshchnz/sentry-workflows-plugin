# Changelog

## 0.2.0 — correctness, reliability, drop unimplementable features

### Critical fixes

- **`groom-stale` Pass 1 was inverted.** The query
  `lastSeen:-${STALE_AGE_DAYS}d` matched issues seen **within** the last N
  days — the opposite of "stale." Pass 1 now uses an absolute ISO 8601
  cutoff (`lastSeen:<<computed-date>>`) since Sentry's filter syntax has no
  `+duration` shorthand for "older than."
- **`groom-stale` Pass 3 (owner assignment) is removed.** It depended on a
  member-lookup tool that the Sentry MCP does not expose. The skill no
  longer attempts user-level assignment; the digest no longer has an
  `Assigned` section.
- **`install-routines` now finds the routine files.** Routines moved from
  the repo's top-level `routines/` into `plugin/routines/` so they ship
  with the cached install. The skill reads
  `${CLAUDE_PLUGIN_ROOT}/routines/*.md`.
- **`/schedule` example syntax fixed.** The previous example
  (`/schedule daily at 9am: paste-prompt-here`) wasn't valid input; the
  skill now points at the `/schedule` skill's conversational interface and
  lists equivalent cron expressions for self-management.
- **Dropped the Slack integration claim.** No Slack MCP is wired into the
  plugin; the digest goes to stdout only. Pipe it to your own webhook in a
  routine follow-up step if you want notification.

### Reliability fixes

- **Real `--dry-run` plumbing.** The `DRY_RUN` flag is now checked at every
  `update_issue` call site, with explicit `(dry-run; skipped)` markers in
  the digest.
- **Preflight pass.** `groom-stale` now calls `whoami` and `find_projects`
  before any data passes so OAuth and org-access failures surface up front
  with actionable error messages.
- **Documented event-count verification.** Pass 1 and Pass 2 now use
  `search_events` with explicit time windows for counts, and read the
  `activity[]` field from `get_sentry_resource` for resolve timestamps.
- **Partial-progress digest.** Per-issue errors are collected into a new
  `Errors` digest section instead of aborting the run.
- **Stable digest schema.** Every digest section renders, even when empty
  (with `(0)` count and `_None._` body), so downstream consumers can parse
  it.
- **Branch-collision preflight in `fix-issue`.** Before invoking the
  implementer, the skill checks whether `claude/sentry-fix-<short-id>`
  already exists locally and whether a PR is already open for it. Handles
  the "stale branch from a failed previous run" case without auto-deleting.
- **Test baseline in `fix-implementer`.** The agent runs tests on the
  clean branch before editing, then compares post-fix failures against
  that baseline. Eliminates the unreliable "is this failure related to my
  change?" judgment call.
- **Expanded vendored/build exclusions in `issue-scorer`.** Now excludes
  `.next/`, `dist/`, `build/`, `out/`, `target/`, `__pycache__/`, `.tox/`,
  `.gradle/`, `bin/`, `obj/`, and respects `git check-ignore`. Also adds
  generated-file patterns (`*.min.js`, `*.bundle.js`, `*.generated.*`,
  protobuf outputs).
- **Source-map prefix normalization in `issue-scorer`.** Strips
  `webpack:///./`, `webpack-internal:///`, `app:///`, `~/`,
  `/_next/static/chunks/` from stack-frame paths before checking
  existence. This fixes the silent skip-everything behavior on JS/TS
  projects with bundlers.

### Out of scope (deferred to 0.3.0)

- JSON digest output for routine integration.
- Stale draft-PR cleanup.
- Configurable commit co-author.
- Slack webhook integration (opt-in via userConfig).
- Full source-map resolution (using actual source-map files, not just
  prefix stripping).

## 0.1.0

Initial preview release.
