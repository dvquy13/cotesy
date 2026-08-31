# cotesy

Decouples **capture** from **curation** for project knowledge bases.

- `cotesy:append` — cheap, inline, write-time. Appends a finding (decision,
  bug, convention, gotcha) to a scope's `.cotesy/wal.md` write-ahead log. No
  doc mutation, no thinking about placement — just log it.
- `cotesy:sync` — periodic curation. Spawns a fresh, memory-less agent that
  drains the WAL, dedupes and audits it against a scope's existing docs
  (`docs/ARCHITECTURE.md` and its per-topic `docs/<topic>.md` files,
  `CLAUDE.md`, `.claude/rules/`), applies the updates, regenerates
  `.cotesy/index.md`, and clears the WAL. Never auto-commits — changes are
  left unstaged for human review via `git diff`.
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

## Install

In Claude Code:

```
/plugin marketplace add dvquy13/cotesy
/plugin install cotesy@cotesy
```

Then, in any project, run `cotesy:init` to scaffold its first `.cotesy/`
scope.

## Upgrading

Update the plugin with `/plugin marketplace update cotesy` (or your usual
plugin-update flow). This never touches a downstream repo's docs by
itself — there's no hook, it's pull-only. `cotesy:sync` only re-audits
existing docs when it has WAL entries to drain, so a scope with an empty
WAL won't notice a rules change at all. **After upgrading, run
`cotesy:tidy` on each `.cotesy/` scope** to re-audit its existing docs
against the current rules and relocate anything that's now misplaced.

See [CONTRIBUTING.md](CONTRIBUTING.md) for local dev setup, releasing, and
repo layout.
