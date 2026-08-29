---
name: sync
description: >
  Drain a cotesy scope's WAL into its docs by spawning a fresh curator agent.
  Use when the user says "cotesy sync", "curate cotesy", or wants pending
  cotesy findings folded into docs/ARCHITECTURE.md, CLAUDE.md, or rules.
allowed-tools: Read, Agent, Bash(git *)
argument-hint: "[path]"
---

# cotesy:sync

Curation layer. Spawns a fresh, memory-less `cotesy-curator` agent to drain a
scope's `.cotesy/wal.md` into its docs, then regenerates `.cotesy/index.md`
and clears the WAL. The curator does its own reading of the WAL and docs —
this skill's job is only to resolve the scope, decide whether there's
anything to do, and spawn the agent.

## Steps

1. **Resolve scope**: the `path` argument if given, else the nearest
   ancestor `.cotesy/` from cwd (same resolution as `cotesy:append`). Error
   clearly if none found.
2. **Check for pending work**: read `<scope>/.cotesy/wal.md`. If it contains
   only the schema-header comment (no `## ` entries), tell the user there's
   nothing to sync and **stop** — do not spawn the curator agent.
3. **Spawn the curator**: invoke the `cotesy-curator` agent, passing the
   resolved `<scope>` absolute path plus the absolute path to this skill's
   `../../shared/placement-scope.md` (resolved relative to this skill's own
   base directory). Do not pass along conversation context, summaries, or
   findings yourself — the agent must derive everything from reading the
   scope's own files and that shared reference, per its
   no-conversation-memory contract.
4. **Surface the agent's summary** to the user verbatim (counts of
   added/updated/removed/relocated entries, which doc targets changed).
5. **Remind the user**: the changes are unstaged — review with `git diff`
   before committing.

## Notes

- This skill never edits docs directly — all doc mutation happens inside the
  curator agent's own tool calls.
- Safe to run repeatedly; an empty WAL is a fast no-op.
