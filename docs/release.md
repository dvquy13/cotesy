# Release process
Release-please based versioning and changelog generation. See README's "Releasing" section for the day-to-day mechanism and Conventional Commits workflow.

## Decisions
- Releases use `release-please` (googleapis/release-please-action@v4) rather than semantic-release, because there's no Node/npm toolchain in this repo — semantic-release typically expects an npm-based host project, while release-please's GitHub Action is self-contained.

## Gotchas
- `release-please-config.json` needs `"include-component-in-tag": false`, or it expects tags formatted `<package-name>-vX.Y.Z` instead of plain `vX.Y.Z`. Setting `package-name: "cotesy"` (needed for readable PR titles) makes release-please default to component-prefixed tags; this repo's baseline tag was cut manually as `v0.1.0`, and without this flag release-please couldn't find that release and errored.
- New GitHub repos default to blocking Actions-created pull requests, which makes release-please fail with "GitHub Actions is not permitted to create or approve pull requests." Fix via `gh api -X PUT repos/<owner>/<repo>/actions/permissions/workflow -f default_workflow_permissions=write -F can_approve_pull_request_reviews=true` — a repo-level setting change, not a code change, worth remembering when bootstrapping release-please on any new repo.
