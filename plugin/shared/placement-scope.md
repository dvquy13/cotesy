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

## Read README.md before writing

If the scope has a `README.md`, read it before adding or updating any doc
target — not to edit it (it's not a routing target), but as dedup context.
Apply the same "non-obvious only" filter from above at write-time, not just
at capture-time: if a fact (or its mechanism/detail) already lives in
README.md, don't restate it in `CLAUDE.md`/`docs/ARCHITECTURE.md`/topic
docs/rules — add only what's incremental (rationale, gotchas, non-obvious
depth) and cross-reference README for the rest. This is a **Duplicate** per
the Audit Criteria below, just checked against one more source.

## Doc targets

Each target exists for a different combination of **how often it's needed**
(does it justify occupying space a session loads automatically or scans by
default?) and **how specific** the content is (one function, one component,
or the whole repo). Get the target right and length stays self-limiting;
get it wrong and content piles up in whichever target has no stated purpose.

- **Code comment** — specific to one function/block; a developer reading
  that code would hit this. Not managed by cotesy agents directly, but
  always wins when applicable — check before routing to any doc.
- **`.claude/rules/<topic>.md`** — rules specific to a directory/file
  pattern; needs `paths` frontmatter to trigger on relevant files. Like
  `CLAUDE.md`, this is **push**: it auto-injects into context the moment a
  session touches a matching file, without anyone asking for it. Use it for
  a constraint or convention that must land *before* someone acts on a
  matching file (e.g. "don't hand-edit these version fields, they're
  bot-managed"). This is a push/pull distinction, not a depth one — don't
  route here just because content is topic-scoped; that's what
  `docs/<topic>.md` below is for.
- **`CLAUDE.md`** — auto-loaded into every session in this scope, so
  admission is gated by the **frequency test**: would a session doing
  *unrelated* work still benefit from knowing this exists, even just as a
  one-line pointer? Passing content: build/test/lint commands, the repo's
  folder map, and other frequently-needed procedures (e.g. "how to
  release") — stated as pointers/one-liners to the relevant topic doc, not
  the full mechanics. Ruthless about length: target under ~50 lines, ~200 is
  a hard ceiling. If it fails the frequency test, it belongs in a topic doc
  or `docs/ARCHITECTURE.md` instead, not here.
- **`docs/<topic>.md`** — one component or feature's decisions, gotchas, and
  implementation detail. This is **pull**: it never auto-loads, only read
  when someone deliberately looks it up (via `.cotesy/index.md`, or when an
  agent is deduping). Right home for rationale/gotchas that are valuable to
  have on hand but don't need to interrupt a session that merely happens to
  touch a related file. This is where most non-trivial findings end up.
  Doesn't exist for every topic from day one — see **Topic graduation**
  below for when one gets created.
- **`docs/ARCHITECTURE.md`** — a short, high-altitude index: what the
  components/features are (one line each, linking to `docs/<topic>.md` once
  a topic has graduated) and how they interact (data flow, use cases).
  Holds only decisions that genuinely span ≥2 components — a decision or
  gotcha scoped to a single component belongs in that component's topic
  doc, not here. Never accumulates a generic catch-all Decisions/Gotchas
  list of everything — that defeats the point of it being an index.

## Topic graduation

A topic starts as a single bullet under `docs/ARCHITECTURE.md`'s Components
section (name — one-line description, no decisions/gotchas inline). It
**graduates** to its own `docs/<topic>.md` once it accumulates 2 or more
non-trivial facts (decisions, gotchas, or implementation detail beyond the
one-liner) — whichever finding causes the second fact to exist triggers the
graduation as part of the same update:

1. Create `docs/<topic>.md` with the topic's one-liner plus every fact about
   it (existing inline ones plus the new one) under `## Decisions` and/or
   `## Gotchas` sections, as applicable.
2. Replace the ARCHITECTURE.md bullet with a one-liner linking to the new
   file: `- **<Topic>** — <one-line summary>. See [docs/<topic>.md](<topic>.md).`

Until graduation, a single fact about a topic stays inline as part of its
Components bullet — don't create a topic file for a component that has only
one thing worth saying about it yet.

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
   `git log`; if unverifiable, prefer the more specific target (see Doc
   Targets above) or the more recent source.
4. **Misplaced**: does this entry violate Doc Targets/Topic graduation above
   (e.g. a topic-scoped decision sitting in `docs/ARCHITECTURE.md`'s
   generic prose, a component with ≥2 facts still inline instead of
   graduated, content in `CLAUDE.md` that fails the frequency test, or a
   push/pull mismatch — a non-urgent gotcha sitting in `.claude/rules/`
   that should be pull-only in `docs/<topic>.md`, or a must-know-before-
   acting constraint buried in `docs/<topic>.md` that should push via
   `.claude/rules/`)?

Tag each existing entry: `KEEP`, `STALE → remove`, `DUPLICATE → merge into
<entry>`, `CONTRADICTION → resolve to <X>`, `RELOCATE → <target>`.

**Don't default to KEEP** — an unverifiable or stale claim sitting in a doc
costs attention at every future read, more than no claim at all. When
genuinely unsure, say so in the plan rather than silently keeping it.

## Plan + dedup convention

For each new/incoming item and each tagged existing entry, assign it to
exactly one line: `[FINDING] → [TARGET] (ADD/UPDATE/REMOVE/RELOCATE)`. `KEEP`
entries produce no line. A finding that triggers topic graduation is one
`RELOCATE` line per fact being moved into the new topic doc, plus one
`UPDATE` line for the ARCHITECTURE.md bullet becoming a pointer.

Before executing the plan, scan it for:
- The same fact assigned to multiple targets → keep only the most specific.
- Near-duplicates within a target → merge into one entry.
- Contradictions with existing docs → resolve per Audit Criteria above.

Only proceed to execution with the deduplicated plan. When reporting the
result, lead with subtraction counts (stale removed, duplicates merged,
contradictions resolved, relocated) before addition counts — it's easy to
under-report cleanup next to a list of additions.

## Index generation

`.cotesy/index.md` is the full topic → file → one-liner map for the scope,
regenerated any time the doc set changes. Include one row per: `CLAUDE.md`
(if present), `docs/ARCHITECTURE.md`, every `docs/<topic>.md` file, every
file under `.claude/rules/`, and any other markdown file sitting directly in
scope root not already covered (e.g. a plan file the scope was pointed at).
One-liner summaries come from each file's own description line/first
heading — don't restate the file's full content.
