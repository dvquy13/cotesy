# cotesy
> Decouples capture (cheap, inline, write-time) from curation (fresh-agent, periodic) for project knowledge bases.

## Structure
- Repo root: `.claude-plugin/marketplace.json` (marketplace listing) + `plugin/` (the actual plugin), mirroring the `qrec`/`knowhub` convention.
- `plugin/.claude-plugin/plugin.json` lists `skills` explicitly by path. Agents are **not** listed there — `plugin/agents/` is auto-discovered by directory convention (confirmed via `knowhub`'s `absorb-agent.md`, which has no `plugin.json` reference).
- Skill dirs are named after the command slug: `plugin/skills/<name>/` → `/cotesy:<name>`.

## Key Concepts
- A **scope** is any folder with its own `.cotesy/` — not just repo root. `cotesy:init` can target any subfolder (e.g. `.claude/plans/<name>`), and scopes nest: `cotesy:append --to <ancestor>` promotes a finding from a descendant scope up to an ancestor's WAL with a `**Source:**` provenance line.
- Capture (`cotesy:append`) only ever writes to `.cotesy/wal.md` — it never touches docs. Curation (`cotesy:sync`) is the only thing that reads/writes `docs/ARCHITECTURE.md`, `CLAUDE.md`, `.claude/rules/`, and `.cotesy/index.md`.
- The curator (`cotesy-curator` agent) is spawned fresh on every sync with no conversation memory — it derives everything from reading the scope's own WAL and existing docs, so its behavior is reproducible independent of how/when findings were appended.
- The audit/dedup/placement logic (Audit Criteria & Tags, Placement Scope routing table, Plan+Dedup convention) is factored into one file, `plugin/shared/placement-scope.md`, rather than duplicated inline in each agent — `cotesy-curator`, `cotesy-retirer`, and `cotesy-groundskeeper` all read it at invocation time. The spawning skill resolves the shared file's absolute path relative to its own base directory (`../../shared/placement-scope.md`) and hands that path to the agent, since agents are memory-less and don't otherwise know their own install location.

## Entry Points
- `cotesy:init [path]` — scaffold a scope.
- `cotesy:append [--to <scope>] <finding>` — log a finding to a WAL.
- `cotesy:sync [path]` — drain a scope's WAL by spawning `cotesy-curator`.
- `cotesy:retire <scope> --to <ancestor>` — one-time distillation: fold a
  scope's *existing docs* (not its WAL) into an ancestor scope's docs via
  the `cotesy-retirer` agent, then (with confirmation) delete the scope.
  Distinct from `sync`: it's a scope-decommission operation, not routine
  periodic curation, and its input is a doc tree rather than WAL entries.
- `cotesy:tidy [path]` — housekeeping-only pass via the `cotesy-groundskeeper`
  agent: audits a scope's existing docs (dedup, staleness, contradictions,
  misplacement) and fixes them, but never reads the WAL and never adds new
  content — its plan can only contain REMOVE/UPDATE(merge)/RELOCATE, never
  ADD. Complements `sync`, which is the only entry point where new content
  actually enters docs.

## Data Flow
`cotesy:append` → entry appended to `<scope>/.cotesy/wal.md` → `cotesy:sync` reads WAL + existing docs → spawns `cotesy-curator` (fresh agent, no memory) → curator drains all WAL entries as one batch, dedupes against each other and against existing docs, routes each via Placement Scope (code comment > `.claude/rules/` > `CLAUDE.md` > `docs/ARCHITECTURE.md`) → regenerates `.cotesy/index.md` → truncates WAL back to schema-header only.

`cotesy:retire <scope> --to <ancestor>` → guards that `<scope>`'s WAL is fully drained (else tells the caller to `cotesy:sync` first) → spawns `cotesy-retirer` (fresh agent, no memory) with both scope paths → retirer reads `<scope>`'s docs (never its WAL beyond the guard check), distills/dedupes/audits into `<ancestor>`'s docs via the same Placement Scope, regenerates `<ancestor>`'s index → skill asks the human whether to delete `<scope>` → if confirmed, `git rm -r <scope>` (staged, unlike doc edits which stay unstaged).

`cotesy:tidy [path]` → spawns `cotesy-groundskeeper` (fresh agent, no memory) with the scope path → groundskeeper reads the scope's existing docs only (never the WAL), audits every entry via the shared Audit Criteria & Tags, builds a plan containing only REMOVE/UPDATE/RELOCATE lines → applies it (or reports "already clean" if the plan is empty) → regenerates the index if topics changed.

## Decisions
- **Curator never auto-commits.** It writes/edits files and leaves them unstaged, matching `project-docs`' git-status-before-edit / review-via-diff pattern. `Bash(git *)` in its toolset is for read-only staleness checks (`git status`/`log`/`diff`) only.
- **Evals use a self-rolled JSON format** (`evals/evals.json` per skill: `{skill_name, evals: [{prompt, expected_output, expectations}]}`), not the official `claude plugin eval` `prompt.md`/`graders/` convention — that CLI is early-access and wasn't enabled in this org at build time. The self-rolled format needs no special access and has precedent in `qrec`. If/when `claude plugin eval` access is granted, these cases upgrade cleanly (same coverage, richer grading).
- **`.cotesy/index.md` indexes more than the three fixed doc targets.** It also includes any other markdown file sitting directly in a scope's root (e.g. a plan file the scope was pointed at) — indexing only `CLAUDE.md`/`ARCHITECTURE.md`/rules would silently omit a scope's primary content when the scope isn't a normal code repo (e.g. a scope pointed at a plans folder). This surfaced during the self-hosted sanity check (`cotesy:init` on `.claude/plans/mvp/` didn't index `mvp-plan.md` until fixed).

## Gotchas
- Registering the plugin in `~/.claude/settings.json` (`extraKnownMarketplaces` + `enabledPlugins`) does not hot-load it into an already-running session — `/cotesy:*` commands only resolve after a session restart.
- `cotesy:append --to <ancestor>` only makes sense when run from *inside* a nested descendant scope; if cwd itself isn't inside any `.cotesy/` scope, there's no "originating scope" to attribute in the `**Source:**` line.

## Dependencies
- No runtime dependencies — everything is plain committed markdown, read/written via the standard Claude Code tool set (Read/Write/Edit/Glob/Grep/Bash).
