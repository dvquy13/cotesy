# cotesy
> Decouples capture (cheap, inline, write-time) from curation (fresh-agent, periodic) for project knowledge bases.

## Components
- **Plugin structure** — repo layout and skill/agent discovery conventions. See [docs/plugin-structure.md](plugin-structure.md).
- **Scopes** — what a `.cotesy/` scope is and how `cotesy:init`/`cotesy:append --to <ancestor>` work across nested scopes. See [docs/scopes.md](scopes.md).
- **Curation (`cotesy:sync`)** — spawns a fresh `cotesy-curator` agent to drain a scope's WAL into its docs. See [docs/curation.md](curation.md).
- **Retire (`cotesy:retire`)** — one-time distillation of a scope's existing docs into an ancestor scope's docs via `cotesy-retirer`, then (with confirmation) deletes the retired scope.
- **Tidy (`cotesy:tidy`)** — housekeeping-only pass via `cotesy-groundskeeper`; never reads the WAL, plan limited to REMOVE/UPDATE(merge)/RELOCATE, never ADD.
- **Evals** — skill evals use a self-rolled JSON format (`evals/evals.json` per skill: `{skill_name, evals: [{prompt, expected_output, expectations}]}`), not the official `claude plugin eval` `prompt.md`/`graders/` convention — that CLI is early-access and wasn't enabled in this org at build time. The self-rolled format needs no special access and has precedent in `qrec`; upgrades cleanly if/when `claude plugin eval` access is granted.
- **Index generation** — `.cotesy/index.md` indexes more than the three fixed doc targets: also any other markdown file sitting directly in a scope's root (e.g. a plan file the scope was pointed at) — indexing only `CLAUDE.md`/`ARCHITECTURE.md`/rules would silently omit a scope's primary content when the scope isn't a normal code repo. This surfaced during the self-hosted sanity check (`cotesy:init` on `.claude/plans/mvp/` didn't index `mvp-plan.md` until fixed).
- **Release process** — release-please based. See [docs/release.md](release.md).

## Data Flow
`cotesy:append` → entry appended to `<scope>/.cotesy/wal.md` → `cotesy:sync` reads WAL + existing docs → spawns `cotesy-curator` (fresh agent, no memory) → curator drains all WAL entries as one batch, dedupes against each other and against existing docs, routes each via Placement Scope (code comment > `.claude/rules/` > `CLAUDE.md` > `docs/ARCHITECTURE.md`) → regenerates `.cotesy/index.md` → truncates WAL back to schema-header only.

`cotesy:retire <scope> --to <ancestor>` → guards that `<scope>`'s WAL is fully drained (else tells the caller to `cotesy:sync` first) → spawns `cotesy-retirer` (fresh agent, no memory) with both scope paths → retirer reads `<scope>`'s docs (never its WAL beyond the guard check), distills/dedupes/audits into `<ancestor>`'s docs via the same Placement Scope, regenerates `<ancestor>`'s index → skill asks the human whether to delete `<scope>` → if confirmed, `git rm -r <scope>` (staged, unlike doc edits which stay unstaged).

`cotesy:tidy [path]` → spawns `cotesy-groundskeeper` (fresh agent, no memory) with the scope path → groundskeeper reads the scope's existing docs only (never the WAL), audits every entry via the shared Audit Criteria & Tags, builds a plan containing only REMOVE/UPDATE/RELOCATE lines → applies it (or reports "already clean" if the plan is empty) → regenerates the index if topics changed.

## Decisions
- **The audit/dedup/placement logic (Audit Criteria & Tags, Placement Scope routing table, Plan+Dedup convention) is factored into one file, `plugin/shared/placement-scope.md`, rather than duplicated inline in each agent** — `cotesy-curator`, `cotesy-retirer`, and `cotesy-groundskeeper` all read it at invocation time, so this spans all three curation-type agents. The spawning skill resolves the shared file's absolute path relative to its own base directory (`../../shared/placement-scope.md`) and hands that path to the agent, since agents are memory-less and don't otherwise know their own install location.

## Dependencies
- No runtime dependencies — everything is plain committed markdown, read/written via the standard Claude Code tool set (Read/Write/Edit/Glob/Grep/Bash).
