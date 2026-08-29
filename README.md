# cotesy

Decouples **capture** from **curation** for project knowledge bases.

- `cotesy:append` — cheap, inline, write-time. Appends a finding (decision,
  bug, convention, gotcha) to a scope's `.cotesy/wal.md` write-ahead log. No
  doc mutation, no thinking about placement — just log it.
- `cotesy:sync` — periodic curation. Spawns a fresh, memory-less agent that
  drains the WAL, dedupes and audits it against a scope's existing docs
  (`docs/ARCHITECTURE.md`, `CLAUDE.md`, `.claude/rules/`), applies the
  updates, regenerates `.cotesy/index.md`, and clears the WAL. Never
  auto-commits — changes are left unstaged for human review via `git diff`.
- `cotesy:init [path]` — scaffold a `.cotesy/` scope (default: cwd), seeding
  `docs/ARCHITECTURE.md` if absent.
- `cotesy:retire <scope> --to <ancestor>` — one-time distillation: fold a
  scope's docs into an ancestor scope's docs, then (with confirmation)
  delete the retired scope. For decommissioning a scope entirely (e.g. a
  completed plan folder), not routine curation.
- `cotesy:tidy [path]` — housekeeping-only pass: dedup, remove stale
  entries, resolve contradictions, fix misplaced content in a scope's
  existing docs. Never reads the WAL and never adds anything new.

A "scope" is any folder with its own `.cotesy/` — not just repo root. Use
`cotesy:append --to <ancestor>` to promote a finding from a nested scope up to
an ancestor's WAL, tagged with a `**Source:**` provenance line.

Everything is plain committed markdown — no database, no hidden state.

## Install (local dev)

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
