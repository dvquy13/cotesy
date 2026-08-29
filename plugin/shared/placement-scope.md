# cotesy shared reference — audit & placement

This file is the single source of truth for how any cotesy agent audits
existing docs, routes content to a target, and plans changes without
stepping on itself. It's handed to agents by path at invocation time (skills
resolve the path relative to their own base directory and tell the agent to
`Read` it) rather than duplicated inline in every agent definition — keeps
the routing/audit rules in exactly one place as they evolve.

## Verified-only / non-obvious filters

Apply these before anything else is written to a doc target:

- **Verified only**: only capture what was actually verified or completed.
  Skip anything framed as a future plan, proposed design, or approach not
  yet taken.
- **Non-obvious only**: skip anything self-evident from reading the code or
  README. Every entry should answer "why" or "gotcha", not "what".

## Audit criteria & tags

Go entry-by-entry through every existing doc target being touched, against
present-day reality — don't trust a doc's own claim, verify it:

1. **Stale**: does this entry reference a file, function, command, or
   pattern that no longer exists or was replaced? Verify via `Grep`/`Glob`/
   `Read` rather than trusting the doc.
2. **Duplicate**: is the same fact stated more than once — within a target,
   across targets, or restated by incoming content?
3. **Contradiction**: do two entries (existing, or existing vs. incoming)
   disagree? Resolve to whichever is confirmed by the current codebase or
   `git log`; if unverifiable, prefer the more specific target (see
   Placement Scope) or the more recent source.
4. **Misplaced**: does this entry violate Placement Scope below (e.g. a
   code-comment-level detail sitting in `CLAUDE.md`)?

Tag each existing entry: `KEEP`, `STALE → remove`, `DUPLICATE → merge into
<entry>`, `CONTRADICTION → resolve to <X>`, `RELOCATE → <target>`.

**Don't default to KEEP** — an unverifiable or stale claim sitting in a doc
costs attention at every future read, more than no claim at all. When
genuinely unsure, say so in the plan rather than silently keeping it.

## Placement Scope

Route each finding/entry to **exactly one** target, most specific wins:

**Code comment** > **`.claude/rules/<topic>.md`** > **`CLAUDE.md`** >
**`docs/ARCHITECTURE.md`**

- **Code comment** — specific to one function/block; a developer reading
  that code would hit this.
- **`.claude/rules/<topic>.md`** — rules specific to a directory/file
  pattern; needs `paths` frontmatter to trigger on relevant files.
- **`CLAUDE.md`** — build/test/lint commands, project-wide conventions,
  workflow rules, important constraints needed from session start. Ruthless
  about length: target under ~50 lines, ~200 is a hard ceiling.
- **`docs/ARCHITECTURE.md`** — structure, key concepts, entry points, data
  flow, architectural decisions + rationale, dependencies.

Each fact lives in exactly one target — if it fits as a code comment, it
doesn't also go in `CLAUDE.md`.

## Plan + dedup convention

For each new/incoming item and each tagged existing entry, assign it to
exactly one line: `[FINDING] → [TARGET] (ADD/UPDATE/REMOVE/RELOCATE)`. `KEEP`
entries produce no line.

Before executing the plan, scan it for:
- The same fact assigned to multiple targets → keep only the most specific.
- Near-duplicates within a target → merge into one entry.
- Contradictions with existing docs → resolve per Audit Criteria above.

Only proceed to execution with the deduplicated plan. When reporting the
result, lead with subtraction counts (stale removed, duplicates merged,
contradictions resolved, relocated) before addition counts — it's easy to
under-report cleanup next to a list of additions.
