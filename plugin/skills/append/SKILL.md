---
name: append
description: >
  Append a finding (decision, bug, convention, gotcha) to the nearest cotesy
  WAL, or promote it to an ancestor scope with --to. Use when the user says
  "cotesy append", "remember this for cotesy", or wants to log a finding
  without immediately updating docs.
allowed-tools: Read, Write, Bash(git *)
argument-hint: "[--to <scope>] <finding>"
---

# cotesy:append

Cheap, inline, write-time capture. Appends one finding to a scope's
`.cotesy/wal.md` (write-ahead log). Does **not** touch docs — that's
`cotesy:sync`'s job, run later by a fresh curator agent.

## WAL entry schema

Reuses `project-docs`' finding taxonomy: `decision`, `bug`, `convention`, `gotcha`.

Each entry is a level-2 heading appended to `.cotesy/wal.md`:

```markdown
## <ISO8601 timestamp> — <type>
**Summary:** one-line statement
**Detail:** rationale / context (the "why")
**Source:** <scope-relative-path>   (only present when written via --to, i.e. promoted from a descendant scope)
```

## Steps

1. **Resolve target scope**:
   - If `--to <scope>` is given, the target is that path. It must already
     contain a `.cotesy/` dir — if not, error clearly: "`<scope>` has no
     .cotesy/ — run cotesy:init there first."
   - Else, resolve the nearest ancestor `.cotesy/` by walking up from cwd
     (same resolution style as git's `.git` lookup). If none is found up to
     filesystem root, error clearly: "No .cotesy/ scope found from `<cwd>` —
     run cotesy:init to create one."
2. **Classify the finding's `type`**: infer `decision`/`bug`/`convention`/
   `gotcha` from the finding text and conversation context. If genuinely
   ambiguous, ask the user to pick one rather than guessing.
3. **Determine `**Source:**`**:
   - If `--to` was used, set `**Source:**` to the *originating* scope — i.e.
     the nearest `.cotesy/` scope from cwd (not the `--to` target), expressed
     relative to the target scope. This is what marks the entry as promoted.
   - If `--to` was not used (appending to the nearest scope directly), omit
     the `**Source:**` line entirely.
4. **Format and append** the entry to `<target-scope>/.cotesy/wal.md`, using
   the current UTC timestamp in ISO8601.
5. **Confirm to the user**: what was written (type + summary) and which
   scope's WAL it landed in.

## Notes

- This skill never edits `docs/ARCHITECTURE.md`, `CLAUDE.md`, `.claude/rules/`,
  or `.cotesy/index.md` — only the WAL. Curation is deferred to `cotesy:sync`.
- Multiple `cotesy:append` calls are cheap and safe — entries just accumulate
  in the WAL until the next sync.
