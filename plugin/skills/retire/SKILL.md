---
name: retire
description: >
  Distill a scope's docs into an ancestor scope's docs, then (with
  confirmation) delete the retired scope. Use when a scope — like a
  completed plan folder — has run its course and its knowledge should be
  folded upward into permanent project docs before the scope is removed.
allowed-tools: Read, Write, Glob, Grep, Bash(git *), Agent, AskUserQuestion
argument-hint: "<scope> --to <ancestor>"
---

# cotesy:retire

One-time, whole-scope distillation — a different operation from
`cotesy:sync`'s routine WAL drain. `cotesy:sync` drains what was appended;
`cotesy:retire` migrates everything a scope's own docs currently say into an
ancestor scope's docs, because the scope itself is going away.

## Steps

1. **Resolve arguments**: `<scope>` (the scope being retired) and `--to
   <ancestor>` (the target scope its knowledge folds into) are both
   required. Both must already contain `.cotesy/` — if either doesn't, error
   clearly naming which one is missing and suggesting `cotesy:init`.
2. **Guard against unsynced findings**: read `<scope>/.cotesy/wal.md`. If it
   contains any `## ` entries (not just the schema header), **stop** and
   tell the user to run `cotesy:sync <scope>` first — retiring with pending
   WAL entries would silently lose them.
3. **Spawn the `cotesy-retirer` agent**, passing the resolved `<scope>`
   (source) and `<ancestor>` (target) absolute paths, plus the absolute path
   to this skill's `../../shared/placement-scope.md` (resolved relative to
   this skill's own base directory). Do not pass along conversation context,
   summaries, or findings yourself — the agent derives everything from
   reading the two scopes' own files and that shared reference, per its
   no-conversation-memory contract.
4. **Surface the agent's summary** to the user verbatim (what migrated,
   where it landed, confirmation the source scope was left untouched).
5. **Ask the user** (via `AskUserQuestion`, or directly if that tool isn't
   available in context) whether to delete `<scope>` now that its content is
   promoted:
   - If yes: run `git rm -r <scope>` — this both deletes the source scope's
     files and stages the removal (unlike doc edits, which stay unstaged;
     deletion is a distinct, deliberate action worth staging for a clean
     review). **Do not run `git commit`.**
   - If no: leave `<scope>` in place and tell the user it's safe to retire
     later — the migrated docs in `<ancestor>` already reflect its content.
6. **Remind the user**: the target scope's doc changes are unstaged, the
   removal (if performed) is staged — review both with `git status`/`git
   diff` before committing.

## Notes

- Never use this for routine curation — that's `cotesy:sync`. Use `retire`
  only when a scope is being decommissioned entirely (a finished plan, a
  folder that's being deleted or merged elsewhere, etc.).
- Safe to run against a scope with sparse or no docs — the retirer will
  report there's nothing worth migrating.
