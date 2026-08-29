---
name: tidy
description: >
  Housekeeping-only pass over a scope's existing docs — dedup, remove stale
  entries, resolve contradictions, fix misplaced content — without
  consuming any new knowledge. Use when the user wants to "clean up docs",
  "dedupe", "audit existing docs", tidy a scope without adding anything, or
  just upgraded the cotesy plugin and wants existing docs re-audited against
  the current placement rules.
allowed-tools: Read, Agent, Bash(git *)
argument-hint: "[path]"
---

# cotesy:tidy

Pure housekeeping. Spawns a fresh, memory-less `cotesy-groundskeeper` agent
that audits a scope's existing docs against present-day reality and against
each other — dedup, staleness, contradictions, misplacement — and fixes what
it finds. Unlike `cotesy:sync` and `cotesy:retire`, it never reads the WAL
and never adds content: it can only remove, merge, or relocate what's
already there.

## Steps

1. **Resolve scope**: the `path` argument if given, else the nearest
   ancestor `.cotesy/` from cwd (same resolution as `cotesy:append`). Error
   clearly if none found.
2. **Spawn the groundskeeper**: invoke the `cotesy-groundskeeper` agent,
   passing the resolved `<scope>` absolute path plus the absolute path to
   this skill's `../../shared/placement-scope.md` (resolved relative to this
   skill's own base directory). Do not pass along conversation context,
   summaries, or findings yourself — the agent derives everything from
   reading the scope's own docs and that shared reference.
3. **Surface the agent's summary** to the user verbatim (counts of
   removed/merged/relocated entries, or "already clean" if nothing changed).
4. **Remind the user**: any changes made are unstaged — review with `git
   diff` before committing.

## Notes

- Complements `cotesy:sync`, which is the only place new WAL-sourced content
  enters docs. `cotesy:tidy` never touches the WAL and never adds anything —
  safe to run anytime, including on a scope with no pending WAL entries.
- Unlike `cotesy:sync`, there's no "nothing to do" WAL check up front — the
  groundskeeper always does the audit and simply reports if it found nothing
  to fix.
- **Run this after upgrading the cotesy plugin.** Plugin updates (e.g. a
  changed `plugin/shared/placement-scope.md` routing rule) never touch a
  downstream scope's docs by themselves — `cotesy:sync` only re-audits when
  it has WAL entries to drain, so a scope with an empty WAL won't pick up a
  rules change on its own. `cotesy:tidy` is the reliable way to re-apply
  current rules to docs written under an older version.
