# Plugin structure
Repo layout and skill/agent discovery conventions. See README's "Layout" section for the directory tree itself.

## Decisions
- Repo layout (`.claude-plugin/marketplace.json` at root + `plugin/` for the actual plugin) mirrors the `qrec`/`knowhub` convention.
- `plugin.json` lists `skills` explicitly by path. Agents are **not** listed there — `plugin/agents/` is auto-discovered by directory convention (confirmed via `knowhub`'s `absorb-agent.md`, which has no `plugin.json` reference).
- Skill dirs are named after the command slug: `plugin/skills/<name>/` → `/cotesy:<name>`.
- `README.md` is user-facing only (what cotesy is, commands, install); dev/maintainer content (local dev install, release-please internals, repo layout tree) lives in `CONTRIBUTING.md` instead. README previously mixed both, leaving no real install instructions for new users.

## Gotchas
- Registering the plugin in `~/.claude/settings.json` (`extraKnownMarketplaces` + `enabledPlugins`) does not hot-load it into an already-running session — `/cotesy:*` commands only resolve after a session restart.
