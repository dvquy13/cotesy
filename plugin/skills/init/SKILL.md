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
4. **Read `../../shared/placement-scope.md`** (resolved relative to this
   skill's own base directory) — it defines the `docs/ARCHITECTURE.md`
   skeleton's purpose (Doc Targets section) and the index-generation
   convention (Index generation section) used below.
5. **Condensed first-pass audit of `<scope>`** (same spirit as `project-docs`
   Phase 1 full audit, scoped to this folder only):
   - Check whether `<scope>/CLAUDE.md`, `<scope>/docs/ARCHITECTURE.md`, and
     `<scope>/.claude/rules/` exist.
   - If `<scope>/docs/ARCHITECTURE.md` does **not** exist, seed it with the
     skeleton (do not invent content — leave sections empty except a one-line
     description if inferable from a README or package manifest in scope).
     Per the shared reference, this file stays a short index — components as
     one-liners, not deep content:
     ```markdown
     # <scope name>
     > One-line description
     ## Components
     ## Data Flow
     ## Decisions
     ## Dependencies
     ```
   - Scan `<scope>` root and key subdirs (one level of `ls`/`Glob`), config
     files (`package.json`, `Cargo.toml`, `pyproject.toml`, etc.), and obvious
     entry points, to fill in `Components` if evident (one line per
     component/feature found — no `docs/<topic>.md` files exist yet at init
     time, so nothing to link to). Skip anything not directly observable —
     no speculation.
6. **Generate `<scope>/.cotesy/index.md`** per the shared reference's Index
   generation convention, from whatever docs exist or were just seeded:
   ```markdown
   # cotesy index — <scope>
   | Topic | File | Summary |
   |---|---|---|
   | Architecture | docs/ARCHITECTURE.md | <one-liner> |
   ```
   If nothing exists yet beyond the just-seeded `ARCHITECTURE.md`, the table
   has one row.
7. **Report**: list exactly what was created (`.cotesy/wal.md`,
   `.cotesy/index.md`, and `docs/ARCHITECTURE.md` if seeded), and remind the
   user these are plain files meant to be committed.

## Notes

- Never overwrite an existing `docs/ARCHITECTURE.md` or `CLAUDE.md` — only
  seed when absent.
- This skill does not call the curator agent — it only scaffolds. Curation
  happens on `cotesy:sync`.
