# Scopes
What a `.cotesy/` scope is and how nesting/promotion works. See README for the base definition ("a scope is any folder with its own `.cotesy/`").

## Decisions
- `cotesy:init` can target any subfolder, not just repo root (e.g. `.claude/plans/<name>`), which is how nested scopes get created.

## Gotchas
- `cotesy:append --to <ancestor>` only makes sense when run from *inside* a nested descendant scope; if cwd itself isn't inside any `.cotesy/` scope, there's no "originating scope" to attribute in the `**Source:**` line.
