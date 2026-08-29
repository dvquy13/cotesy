# Cotesy MVP Implementation Plan

## Context
`~/frostmourne/cotesy` is currently an empty directory. The design ([[Cotesy]]) and MVP scope ([[Cotesy - MVP]]) are fully specified in the vault but nothing is built yet. Cotesy decouples **capture** (cheap, inline, write-time — `cotesy:append` writing to a WAL) from **curation** (a fresh, memory-less agent that periodically drains the WAL and updates a repo's knowledge base — `cotesy:sync`). It generalizes `project-docs`' audit/dedupe/placement logic to work on any folder scope (not just repo root), ships as a portable Claude Code plugin, and keeps everything as plain committed markdown.

Studied for reuse:
- `~/.claude/skills/project-docs/SKILL.md` — source of the audit/dedupe/contradiction/placement logic the curator agent will reuse (condensed inline, since cotesy must be portable to repos/users that don't have this personal skill installed).
- `~/frostmourne/qrec` and `~/frostmourne/knowhub` — existing plugin skeletons. Confirmed conventions to mirror:
  - Repo root: `.claude-plugin/marketplace.json` (lists `{"name","source":"./plugin",...}`) + `plugin/` subfolder containing the actual plugin.
  - `plugin/.claude-plugin/plugin.json` — `name`, `version`, `description`, explicit `skills` array (paths to skill dirs). Agents are **not** listed in plugin.json — `knowhub/plugin/agents/absorb-agent.md` exists with no plugin.json reference, so a plugin-level `agents/` directory is auto-discovered by directory convention.
  - Skill dirs are named after the command slug and invoked as `/<plugin>:<dir-name>` (e.g. `knowhub/plugin/skills/capture` → `/knowhub:capture`). So `cotesy:init/append/sync` map directly to `skills/init/`, `skills/append/`, `skills/sync/`.
  - Agent frontmatter format (from `.claude/agents/*.md` examples): `name`, `description`, `tools:` comma-separated list. This is what encodes the curator's restricted toolset.
  - Local dev registration: `~/.claude/settings.json` → `extraKnownMarketplaces.<name>.source = {source:"directory", path:"<repo root>"}` + `enabledPlugins["<plugin>@<marketplace>"] = true`.

Decisions confirmed with user:
- Curator agent does **not** auto-commit — it writes/edits files and leaves them unstaged for manual review (matches `project-docs`' existing git-status-before-edit, review-via-diff pattern).
- This plan includes registering cotesy locally in `~/.claude/settings.json` so the commands are actually invokable for dogfooding.
- Skill eval is included in the spec from the start (per user request). Researched via `claude-code-guide`:
  - The official `claude plugin eval` CLI is **early access, not enabled in this session/org** — self-test is `claude plugin eval` in an empty dir; "early access" message means not enabled, "No eval cases found" means it is.
  - Its convention (for when enabled): `evals/<case-name>/prompt.md` (YAML frontmatter: `name`, `tags`, `runs`, `allowed_tools`, etc.) + `graders/<name>.md` (frontmatter `type: regex|tool_used|tool_order|file_exists|llm|baseline` + rubric body) + optional `case.yaml` with `context.scaffold_script` to seed filesystem state before each isolated run. This maps well onto cotesy's file-mutating skills: scaffold `.cotesy/wal.md`/`.cotesy/index.md` state, then grade with `file_exists` / `regex` (`target: {source: file, path: ...}`) graders on the resulting files.
  - `~/frostmourne/qrec/plugin/skills/recall/evals/evals.json` shows a simpler, self-rolled convention already used in this user's own repos (`{skill_name, evals: [{prompt, expected_output, expectations}]}`) that needs no special access and works today.
  - `/skill-doctor` is a post-deployment usage-stats report (skill call counts, cost, "never invoked" flags), not a test harness — skip it for v1, revisit after dogfooding.
  - **Decision**: scaffold `evals/` per skill now using the lightweight self-rolled JSON format (no access gating, precedent in `qrec`), written alongside each SKILL.md rather than deferred. If/when `claude plugin eval` access is granted, these upgrade cleanly to `prompt.md`/`graders/` — same case coverage, richer grading.

### 0. Relocate this plan into the repo
- First action once this plan is approved, before step 1: move this plan file into `~/frostmourne/cotesy/.claude/plans/mvp/mvp-plan.md` — it becomes the tracked, in-repo source of truth for the build (and doubles as the first real content the repo's own `.cotesy/` scope will later index).

## File layout to build

```
~/frostmourne/cotesy/
  .claude-plugin/
    marketplace.json
  plugin/
    .claude-plugin/
      plugin.json
    agents/
      curator.md
    skills/
      init/
        SKILL.md
        evals/
          evals.json
      append/
        SKILL.md
        evals/
          evals.json
      sync/
        SKILL.md
        evals/
          evals.json
  README.md
```

## Steps

### 1. Repo + plugin skeleton
- `git init` in `~/frostmourne/cotesy`.
- Write `.claude-plugin/marketplace.json` (name `cotesy`, owner, one plugin entry `source: "./plugin"`), modeled on `qrec/.claude-plugin/marketplace.json`.
- Write `plugin/.claude-plugin/plugin.json`: `name: cotesy`, `version: 0.1.0`, `description`, `skills: ["./skills/init", "./skills/append", "./skills/sync"]`.

### 2. WAL entry schema + provenance
Define in a shared reference the skills/agent all point to (put it directly in `skills/append/SKILL.md` and mirrored in `agents/curator.md`, since there's no separate docs step for it):
- Reuses `project-docs`' finding taxonomy: `decision`, `bug`, `convention`, `gotcha`.
- Each WAL entry (level-2 heading, plain markdown, appended to `.cotesy/wal.md`):
  ```markdown
  ## 2026-08-28T22:10:00Z — decision
  **Summary:** one-line statement
  **Detail:** rationale / context (the "why")
  **Source:** <scope-relative-path>   (only present when written via --to, i.e. promoted from a descendant scope)
  ```
- `.cotesy/wal.md` starts (post-init, post-sync) with just a header comment describing the schema — no entries.

### 3. `skills/init/SKILL.md` → `cotesy:init [path]`
- Resolve target scope: given `path` or cwd.
- Create `<scope>/.cotesy/wal.md` (empty, schema header only) and `<scope>/.cotesy/index.md`.
- Run a condensed first-pass audit inline (same spirit as `project-docs` Phase 1 "Full audit" branch, scoped to `<scope>`): check for existing `CLAUDE.md`, `docs/ARCHITECTURE.md`, `.claude/rules/` under the scope; if `docs/ARCHITECTURE.md` doesn't exist, seed the skeleton from `project-docs`' format (`Structure/Key Concepts/Entry Points/Data Flow/Decisions/Gotchas/Dependencies`); scan scope root + key subdirs, config files, entry points.
- Generate `<scope>/.cotesy/index.md` from whatever docs exist/were seeded (topic → file → one-liner, table format).
- Report what was created.
- `skills/init/evals/evals.json`: 1-2 cases covering — fresh empty repo (skeleton seeded from scratch) and a repo with existing `docs/ARCHITECTURE.md`/`CLAUDE.md` (index built from existing content, nothing clobbered).

### 4. `skills/append/SKILL.md` → `cotesy:append [--to <scope>] <finding>`
- If `--to <scope>` given, target that path directly (must contain `.cotesy/`, else error).
- Else resolve nearest ancestor `.cotesy/` by walking up from cwd (git-`.git`-style resolution) — error clearly if none found (suggest running `cotesy:init`).
- Prompt/confirm the finding's `type` (decision/bug/convention/gotcha) if not inferable, format per the WAL schema (§2), set `**Source:**` only when `--to` was used (value = the *originating* scope, i.e. cwd's nearest scope, not the target).
- Append to `<target-scope>/.cotesy/wal.md`. Confirm to user what was written and where.
- `skills/append/evals/evals.json`: 2-3 cases covering — nearest-scope resolution (no `--to`), explicit `--to <ancestor>` promotion (checks `**Source:**` provenance line is added), and the "no `.cotesy/` found" error path.

### 5. `agents/curator.md` — the curator agent
- Frontmatter: `name: cotesy-curator`, `description`, `tools: Read, Write, Edit, Glob, Grep, Bash(git *)`.
- Body instructions (condensed, inline copy of the relevant `project-docs` logic — Phase 1 read state / Phase 2 distill+audit / Phase 3 dedup / Phase 4 execute, and the Placement Scope table), adapted to:
  - Operate purely from: the WAL content it's handed + reading `<scope>`'s existing docs. No conversation memory, no session context.
  - Drain **all** entries in `<scope>/.cotesy/wal.md` as one batch — dedupe against each other before touching docs, not just against existing docs.
  - Route each entry via Placement Scope (code comment > `.claude/rules/` > `CLAUDE.md` > `docs/ARCHITECTURE.md`), same audit tags (`KEEP`/`STALE`/`DUPLICATE`/`CONTRADICTION`/`RELOCATE`).
  - Regenerate `<scope>/.cotesy/index.md` after doc updates.
  - Truncate `<scope>/.cotesy/wal.md` back to the empty schema-header state once fully drained.
  - **Do not run `git commit` or `git add`** — `Bash(git *)` is for `git status`/`git log` checks only (staleness verification, dirty-tree check before editing), per the confirmed no-auto-commit decision.
  - Output a summary: counts of added/updated/removed/relocated entries.

### 6. `skills/sync/SKILL.md` → `cotesy:sync [path]`
- Resolve scope (default: nearest `.cotesy/` from cwd, or explicit `path`).
- Read `<scope>/.cotesy/wal.md`; if only the empty header is present, tell the user there's nothing to sync and stop.
- Spawn the curator agent fresh via the Agent tool (`subagent_type: cotesy-curator`), passing only the scope path — the agent does its own Read of WAL + docs, per the "no conversation memory" requirement.
- Surface the agent's summary to the user; remind them the changes are unstaged and to review the diff.
- `skills/sync/evals/evals.json`: 2 cases covering — draining a WAL with a mix of entry types into the right Placement Scope targets (`expectations` check the curator doesn't touch `git commit`/`git add`), and the empty-WAL no-op path (skill reports nothing to sync, doesn't spawn the agent).

### 7. Local registration for dogfooding
- Add to `~/.claude/settings.json`: `extraKnownMarketplaces.cotesy = {"source": {"source": "directory", "path": "/Users/dvq/frostmourne/cotesy"}}` and `enabledPlugins["cotesy@cotesy"] = true`, mirroring the existing `qrec` entry.

### 8. Verification (end-to-end, single scope)
1. In a real target repo (or `~/frostmourne/cotesy` itself, dogfooding), run `cotesy:init` → confirm `.cotesy/wal.md` and `.cotesy/index.md` are created and are plain committed-looking markdown.
2. Run `cotesy:append` 2-3 times with different finding types → confirm entries land correctly formatted in `wal.md`.
3. Run `cotesy:sync` → confirm the curator agent spawns fresh (restricted tools only), updates the right doc targets per Placement Scope, regenerates `index.md`, clears `wal.md`, does not run `git commit`/`git add`, and leaves a clean human-reviewable diff (`git diff`).
4. Nested-scope test: create a second `.cotesy/` scope in a subfolder of the same repo, run `cotesy:append --to <ancestor>` from inside it, confirm the entry lands in the ancestor's WAL tagged with `**Source:**`, and that the ancestor's own `cotesy:sync` drains it normally.
5. **Definition of done — self-hosted sanity check:** run `cotesy:init .claude/plans/mvp/` inside `~/frostmourne/cotesy` itself, scoping cotesy onto the very folder this plan now lives in. Confirm `.claude/plans/mvp/.cotesy/wal.md` + `index.md` are created and `index.md` picks up `mvp-plan.md`. Then `cotesy:append` one real finding from this build (e.g. a decision made during implementation) into that scope and `cotesy:sync` it, confirming the curator agent correctly ingests a finding about cotesy's own build into cotesy's own docs. The plugin isn't done until it can be pointed at its own plan folder and behave correctly end-to-end.
