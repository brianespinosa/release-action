# .github/CLAUDE.md

Guidance for Claude Code working in the `.github/` directory of `release-action`.

---

## Workflow files

- `workflows/release-workflow.yml` — the reusable workflow (`workflow_call`); this is what consumers reference
- `workflows/release.yml` — the calling workflow; this repo dogfoods its own pipeline

---

## Workflow structure

`release-workflow.yml` has three jobs:

| Job | Runs when |
|-----|-----------|
| `init-check` | always |
| `release` | `inputs.trigger == 'push'` (after `init-check`) |
| `dependabot-major-prefix` | `inputs.trigger == 'pull_request_target' && github.actor == 'dependabot[bot]'` (after `init-check`) |

`init-check` always runs so the repo settings check fires on PRs as well as on push, blocking a bad merge before it lands rather than only after. Both `release` and `dependabot-major-prefix` declare `needs: init-check`.

**Branch protection requirement:** Rules must require the overall workflow status, not individual job statuses. When `init-check` fails, downstream jobs are skipped (neutral gray), not failed. A rule that only requires `release` to pass will not block merges when `init-check` fails.

The actor check for `dependabot-major-prefix` is at the **job level**, not per-step. This is intentional — a job-level `if:` is the correct GitHub Actions pattern for "this job only applies to a specific actor." Do not move it to individual steps.

### `init-check` job steps

1. **Init check** — assert `allow_squash_merge == true` and `allow_merge_commit == false` via `gh api`; fail with descriptive error if not. Note: `allow_rebase_merge` is intentionally not checked — consumers should also disable it, but the workflow does not enforce this at runtime.

### `release` job steps (in order)

1. **Checkout** — `actions/checkout@v4` with `fetch-depth: 0` and `fetch-tags: true`
2. **Write cliff.toml** — bash heredoc writing bundled config to working directory
3. **Get current tag** — `git tag --list 'v*.*.*' --sort=-version:refname | head -1`; logs whether this is a first release or an existing tag
4. **Install git-cliff** — `orhun/git-cliff-action@v4` with `args: --bumped-version`; followed by a **Verify git-cliff installation** step that confirms the binary is present at `$RUNNER_TEMP/git-cliff/bin`
5. **Check if release needed** — direct binary call; stderr captured to temp file (not merged with stdout) to prevent warnings contaminating the version string; sets `skip=true` or `version` output (not both — `skip` is only written when skipping)
6. **Generate changelog** — direct binary call; VERSION passed via env; stderr captured to temp file; multiline output written to `$GITHUB_OUTPUT` using `content<<CLIFF_OUTPUT` delimiter syntax
7. **Create GitHub Release** — `gh release create`; guards against empty VERSION and CHANGELOG before running
8. **Update alias tags** — force-update `vN` and `vN.M` with explicit per-push error handling and recovery instructions

### `dependabot-major-prefix` job steps

1. **Fetch metadata** — `dependabot/fetch-metadata@v2`
2. **Rewrite title for major updates** — only runs when `steps.metadata.outputs.update-type == 'version-update:semver-major'`; bash substitution `${PR_TITLE/fix(deps)/feat(deps)!}`; logs before/after; fails if substitution produces no change; updates via `gh pr edit`

---

## Known gotchas

### git-cliff action outputs are broken

`orhun/git-cliff-action@v4` has a jq bug that causes `outputs.stdout` and `outputs.version` to be unreliable. Do NOT use them. Confirmed broken at `orhun/git-cliff-action@v4`; re-verify when upgrading to a new major version before removing the direct binary call workaround.

- To get the bumped version: export PATH, call `git-cliff --bumped-version` directly
- To generate the changelog: call `git-cliff --unreleased --strip header --tag "$VERSION"` directly; write multiline output to `$GITHUB_OUTPUT` using the `content<<CLIFF_OUTPUT` heredoc delimiter syntax
- PATH to add: `$RUNNER_TEMP/git-cliff/bin` — the binary is NOT added to `$GITHUB_PATH` automatically
- Do NOT use any outputs from the action. The "Generate changelog" step calls the binary directly and writes its own `content` key to `$GITHUB_OUTPUT` using the `content<<CLIFF_OUTPUT` heredoc delimiter. `steps.changelog.outputs.content` refers to that manually-written step output, not anything produced by the git-cliff action itself.

### Alias tags must be excluded

git-cliff and `git tag --list` must both exclude alias tags like `v1` and `v1.2`.

- `tag_pattern = "v[0-9]+\\.[0-9]+\\.[0-9]+"` in cliff.toml restricts git-cliff to full semver tags only
- `git tag --list 'v*.*.*'` glob pattern excludes single-segment and two-segment tags

### Breaking change syntax

git-cliff recognizes `feat(scope)!:` as a breaking change. It does NOT recognize `feat!(scope):`. Always use the `!` after the scope (or after the type if there is no scope).

### pull_request_target injection risk

The `dependabot-major-prefix` job uses `pull_request_target`, which runs with elevated write permissions. Never interpolate untrusted values directly in a `run:` block. Always pass through `env:`.

Two values in this job are treated this way:

- `PR_TITLE` — contains user-controlled commit message content from the Dependabot PR
- `PR_NUMBER` — passed via env for consistency, though not a content-injection risk

The `CHANGELOG` env var in the `release` job's "Create GitHub Release" step applies the same reasoning: changelog content is derived from commit messages (user-controlled). It must stay as `env: CHANGELOG:` and never be inlined as `${{ steps.changelog.outputs.content }}` in the `run:` block.

This job is safe because it never checks out PR code — it only reads event metadata and edits the PR title.

---

## Semver mapping

| Conventional commit | Bump | Changelog group |
|---------------------|------|-----------------|
| `feat` | minor | Features |
| `fix` | patch | Bug Fixes |
| `fix(deps)` | patch | Dependencies |
| `perf` | patch | Performance |
| `refactor` | patch | Refactor |
| `doc` / `docs` | patch | Documentation |
| `feat(scope)!:` or `BREAKING CHANGE` footer | major | — |
| `chore`, `ci`, `style`, `test`, `build` | none (skipped) | — |

`features_always_bump_minor = true` and `breaking_always_bump_major = true` are set explicitly in cliff.toml. `perf`, `refactor`, and `doc` produce patch bumps via git-cliff's default behavior (no explicit bump override, no `skip = true`). The `doc` parser uses a prefix match (`^doc`), so both `doc:` and `docs:` commits are matched.

---

## Squash merge requirement

**All PRs must be merged via squash merge.**

The `dependabot-major-prefix` job rewrites the PR title. That rewritten title becomes the commit message only when squash merging. A regular merge commit preserves original commit messages, causing git-cliff to see `fix(deps):` instead of `feat(deps)!:` and produce the wrong version bump.

The `release` job enforces squash at runtime: it checks `allow_merge_commit == false` via the GitHub API and fails with an error if merge commits are enabled. It does not check `allow_rebase_merge` — consumers should also disable rebase merges, but this is not runtime-enforced.

---

## Dependabot configuration

`dependabot.yml` targets `github-actions` ecosystem only. All updates use `fix(deps):` as the commit prefix. Major version bumps get their PR title rewritten to `feat(deps)!:` by the `dependabot-major-prefix` workflow job.

The `pull_request_target` trigger includes both `opened` and `reopened` event types. The `reopened` type is intentional — Dependabot PRs can be closed and reopened when the base branch changes, and the title rewrite must fire again in that case.
