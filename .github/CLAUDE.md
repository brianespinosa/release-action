# .github/CLAUDE.md

Guidance for Claude Code working in the `.github/` directory of `release-action`.

---

## Workflow files

- `workflows/release-workflow.yml` — the reusable workflow (`workflow_call`); this is what consumers reference
- `workflows/release.yml` — the calling workflow; this repo dogfoods its own pipeline

---

## Workflow structure

`release-workflow.yml` has two jobs:

| Job | Runs when |
|-----|-----------|
| `release` | `inputs.trigger == 'push'` |
| `dependabot-major-prefix` | `inputs.trigger == 'pull_request_target' && github.actor == 'dependabot[bot]'` |

The actor check for `dependabot-major-prefix` is at the **job level**, not per-step. This is intentional — a job-level `if:` is the correct GitHub Actions pattern for "this job only applies to a specific actor." Do not move it to individual steps.

### `release` job steps (in order)

1. **Checkout** — `actions/checkout@v4` with `fetch-depth: 0` and `fetch-tags: true`
2. **Write cliff.toml** — bash heredoc writing bundled config to working directory
3. **Get current tag** — `git tag --list 'v*.*.*' --sort=-version:refname | head -1`; logs whether this is a first release or an existing tag
4. **Install git-cliff** — `orhun/git-cliff-action@v4` with `args: --bumped-version`; followed by a **Verify git-cliff installation** step that confirms the binary is present at `$RUNNER_TEMP/git-cliff/bin`
5. **Check if release needed** — direct binary call; stderr captured to temp file (not merged with stdout) to prevent warnings contaminating the version string; sets `skip=true` or `version` output (not both — `skip` is only written when skipping); skip fires when `BUMPED` is empty, equal to `CURRENT`, or less than `CURRENT` (git-cliff emits a lower version when there is nothing to bump)
6. **Generate changelog** — direct binary call; VERSION passed via env; stderr captured to temp file; multiline output written to `$GITHUB_OUTPUT` using `content<<CLIFF_OUTPUT` delimiter syntax
7. **Create GitHub Release** — `gh release create`; guards against empty VERSION and CHANGELOG before running
8. **Update alias tags** — force-update `vN` and `vN.M` with explicit per-push error handling and recovery instructions

### `dependabot-major-prefix` job steps

1. **Fetch metadata** — `dependabot/fetch-metadata@v2`
2. **Rewrite title for major updates** — only runs when `steps.metadata.outputs.update-type == 'version-update:semver-major'`; bash substitution `${PR_TITLE/fix(deps)/feat(deps)!}`; logs before/after; fails if substitution produces no change; updates via `gh pr edit`

---

## Known gotchas

### cliff.toml bump settings belong in [bump], not [git] (git-cliff v2)

In git-cliff v2, `features_always_bump_minor`, `breaking_always_bump_major`, and `initial_tag` moved from the `[git]` section to a dedicated `[bump]` section. Keeping them in `[git]` silently ignores them — git-cliff falls back to its hardcoded default `initial_tag` of `0.1.0`, which then fails the `tag_pattern = "v[0-9]+\\.[0-9]+\\.[0-9]+"` validation with a fatal error.

The `Install git-cliff` step uses `continue-on-error: true` as a secondary defense: the action runs git-cliff during install, and any git-cliff error would fail the step. Since we only use that action to install the binary, its run-time result is irrelevant.

### git-cliff action outputs are broken

`orhun/git-cliff-action@v4` has a jq bug that causes `outputs.stdout` and `outputs.version` to be unreliable. Do NOT use them. Confirmed broken at `orhun/git-cliff-action@v4`; re-verify when upgrading to a new major version before removing the direct binary call workaround.

- To get the bumped version: export PATH, call `git-cliff --bumped-version` directly
- To generate the changelog: call `git-cliff --unreleased --strip header --tag "$VERSION"` directly; write multiline output to `$GITHUB_OUTPUT` using the `content<<CLIFF_OUTPUT` heredoc delimiter syntax
- PATH to add: `$RUNNER_TEMP/git-cliff/bin` — the binary is NOT added to `$GITHUB_PATH` automatically
- Do NOT use any outputs from the action. The "Generate changelog" step calls the binary directly and writes its own `content` key to `$GITHUB_OUTPUT` using the `content<<CLIFF_OUTPUT` heredoc delimiter. `steps.changelog.outputs.content` refers to that manually-written step output, not anything produced by the git-cliff action itself.

### cliff.toml cannot be written with a shell heredoc

The `run: |` YAML literal block parser terminates when it sees a non-empty line with less indentation than the block's first content line. Shell heredocs require the `EOF` end marker at column 0. These constraints are mutually exclusive: if the cliff.toml content is indented to satisfy YAML, the `EOF` marker at column 0 is outside the block and the YAML parser errors; if the content is at column 0 to satisfy bash, the YAML parser terminates the `run:` block at the first `[section]` header.

The fix: put the TOML content in an `env:` variable (`CLIFF_TOML: |`) and write it with `printf '%s\n' "$CLIFF_TOML" > cliff.toml`. YAML strips the block indentation from the env value automatically. Do not convert this back to a heredoc.

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

`features_always_bump_minor = true` and `breaking_always_bump_major = true` are set in the `[bump]` section of cliff.toml (git-cliff v2 moved these from `[git]`). `perf`, `refactor`, and `doc` produce patch bumps via git-cliff's default behavior (no explicit bump override, no `skip = true`). The `doc` parser uses a prefix match (`^doc`), so both `doc:` and `docs:` commits are matched.

---

## Squash merge requirement

**All PRs must be merged via squash merge.**

The `dependabot-major-prefix` job rewrites the PR title. That rewritten title becomes the commit message only when squash merging. A regular merge commit preserves original commit messages, causing git-cliff to see `fix(deps):` instead of `feat(deps)!:` and produce the wrong version bump.

Squash merge must be enabled and merge commits must be disabled in the repository settings — these are required for git-cliff to produce correct semver bumps from PR titles used as squash commit messages. This is not runtime-enforced; verify manually when setting up a new consumer.

---

## Dependabot configuration

`dependabot.yml` targets `github-actions` ecosystem only. All updates use `fix(deps):` as the commit prefix. Major version bumps get their PR title rewritten to `feat(deps)!:` by the `dependabot-major-prefix` workflow job.

The `pull_request_target` trigger includes both `opened` and `reopened` event types. The `reopened` type is intentional — Dependabot PRs can be closed and reopened when the base branch changes, and the title rewrite must fire again in that case.
