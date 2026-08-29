# mvp
> One-line description
## Structure
## Key Concepts
## Entry Points
## Data Flow
## Decisions
- `cotesy:init` seeds `docs/ARCHITECTURE.md` even for non-code scopes (e.g. plan folders), rather than branching by scope type. One code path handles both repo roots and narrow scopes like a `plans/` dir, at the cost of a placeholder `ARCHITECTURE.md` sitting next to a plan file — accepted as the simpler option (simplicity-first).

## Gotchas
- `.cotesy/index.md` must include non-doc-target markdown files already sitting in scope root (e.g. `mvp-plan.md`), not just the three fixed doc targets (`CLAUDE.md`/`ARCHITECTURE.md`/rules). Indexing only the fixed targets means a scope's own reference content never appears in `index.md` even though it's the primary content of the scope — fixed by scanning scope-root markdown files as an additional index source.
## Dependencies
