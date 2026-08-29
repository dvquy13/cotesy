# cotesy

## Conventions

- All cotesy skills (`init`/`append`/`sync`) resolve scope the same way: explicit path/`--to` argument, else nearest ancestor `.cotesy/` from cwd (git-`.git`-style walk-up). Keeps append/sync/init behavior predictable and lets nested scopes compose without special-casing each skill's resolution logic.
