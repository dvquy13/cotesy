---
name: init
description: >
  Initialize a cotesy scope at a path (default: cwd). Creates .cotesy/wal.md
  and .cotesy/index.md, seeding docs/ARCHITECTURE.md if missing. Use when the
  user says "cotesy init", "set up cotesy here", or wants to start tracking a
  folder's knowledge base with cotesy.
allowed-tools: Read, Write, Glob, Grep, Bash(ls *), Bash(git *)
argument-hint: "[path]"
---

# cotesy:init

Initialize a cotesy-tracked scope: a folder (default cwd) that gets its own
`.cotesy/wal.md` (write-ahead log for `cotesy:append`) and `.cotesy/index.md`
(topic → file → one-liner map, kept current by `cotesy:sync`).

## Steps

1. **Resolve target scope**: the path argument if given, else cwd. Resolve to
   an absolute path.
2. **Guard**: if `<scope>/.cotesy/` already exists, tell the user this scope
   is already initialized and stop (don't clobber an existing WAL/index).
3. **Create `<scope>/.cotesy/wal.md`** with just the schema header, no entries:
   ```markdown
   <!--
   cotesy WAL — append-only findings log, drained by cotesy:sync.
   Entry format:
   ## <ISO8601 timestamp> — <decision|bug|convention|gotcha>
   **Summary:** one-line statement
   **Detail:** rationale / context (the "why")
   **Source:** <scope-relative-path>   (only present when promoted via --to)
   -->
   ```
4. **Condensed first-pass audit of `<scope>`** (same spirit as `project-docs`
   Phase 1 full audit, scoped to this folder only):
   - Check whether `<scope>/CLAUDE.md`, `<scope>/docs/ARCHITECTURE.md`, and
     `<scope>/.claude/rules/` exist.
   - If `<scope>/docs/ARCHITECTURE.md` does **not** exist, seed it with the
     skeleton (do not invent content — leave sections empty except a one-line
     description if inferable from a README or package manifest in scope):
     ```markdown
     # <scope name>
     > One-line description
     ## Structure
     ## Key Concepts
     ## Entry Points
     ## Data Flow
     ## Decisions
     ## Gotchas
     ## Dependencies
     ```
   - Scan `<scope>` root and key subdirs (one level of `ls`/`Glob`), config
     files (`package.json`, `Cargo.toml`, `pyproject.toml`, etc.), and obvious
     entry points, to fill in `Structure`/`Entry Points` if evident. Skip
     anything not directly observable — no speculation.
5. **Generate `<scope>/.cotesy/index.md`** from whatever docs exist or were
   just seeded, as a table:
   ```markdown
   # cotesy index — <scope>
   | Topic | File | Summary |
   |---|---|---|
   | Architecture | docs/ARCHITECTURE.md | <one-liner> |
   ```
   Include a row per doc target that exists in scope (`CLAUDE.md`,
   `docs/ARCHITECTURE.md`, each file under `.claude/rules/`). Also include a
   row for any other markdown file sitting directly in scope root (e.g. a
   plan or reference file this scope was pointed at) that isn't already
   covered — one-liner summary taken from its first heading or first
   non-blank line. If nothing exists yet beyond the just-seeded
   `ARCHITECTURE.md`, the table has one row.
6. **Report**: list exactly what was created (`.cotesy/wal.md`,
   `.cotesy/index.md`, and `docs/ARCHITECTURE.md` if seeded), and remind the
   user these are plain files meant to be committed.

## Notes

- Never overwrite an existing `docs/ARCHITECTURE.md` or `CLAUDE.md` — only
  seed when absent.
- This skill does not call the curator agent — it only scaffolds. Curation
  happens on `cotesy:sync`.
