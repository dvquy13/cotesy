# Contributing

## Local dev install

Registered via `~/.claude/settings.json`:
```json
"extraKnownMarketplaces": { "cotesy": { "source": { "source": "directory", "path": "/Users/dvq/frostmourne/cotesy" } } },
"enabledPlugins": { "cotesy@cotesy": true }
```

## Releasing

Versioning and changelog generation are automated with
[release-please](https://github.com/googleapis/release-please). Commits to
`main` must use [Conventional Commits](https://www.conventionalcommits.org/)
prefixes (`feat:`, `fix:`, `feat!:` / `BREAKING CHANGE:` footer, `chore:`,
`docs:`, `refactor:`, ...) — these determine the next semver bump.

On every push to `main`, the `release-please` workflow opens or updates a
standing "release PR" containing the version bump (applied in lockstep to
`plugin/.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`)
and the `CHANGELOG.md` entry. Merging that PR tags the release (`vX.Y.Z`)
and publishes a GitHub Release — no manual version editing or changelog
writing required. See `release-please-config.json`.

## Layout

```
.claude-plugin/marketplace.json
plugin/
  .claude-plugin/plugin.json
  agents/curator.md
  agents/retirer.md
  agents/groundskeeper.md
  shared/placement-scope.md
  skills/{init,append,sync,retire,tidy}/SKILL.md (+ evals/evals.json)
```
