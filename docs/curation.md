# Curation (`cotesy:sync`)
Spawns a fresh `cotesy-curator` agent to drain a scope's WAL into its docs. See README for the command's user-facing description.

## Decisions
- The curator is spawned fresh on every sync with no conversation memory — it derives everything from reading the scope's own WAL and existing docs, so its behavior is reproducible independent of how/when findings were appended.
- The curator never auto-commits: it writes/edits files and leaves them unstaged, matching `project-docs`' git-status-before-edit / review-via-diff pattern. `Bash(git *)` in its toolset is for read-only staleness checks (`git status`/`log`/`diff`) only.
