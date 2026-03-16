# .github/CLAUDE.md

Guidance for Claude Code working in the `.github/` directory of `release-action`.

---

## Workflow files

- `workflows/release-action.yml` — the reusable workflow (`workflow_call`); this is what consumers reference
- `workflows/release.yml` — the calling workflow; this repo dogfoods its own pipeline

---

## Workflow structure

`release-action.yml` has two jobs, both gated by the `inputs.trigger` value passed from the caller:

| Job | Runs when |
|-----|-----------|
| `release` | `inputs.trigger == 'push'` |
| `dependabot-major-prefix` | `inputs.trigger == 'pull_request_target'` |

### `release` job steps (in order)

1. **Init check** — assert `allow_squash_merge == true` and `allow_merge_commit == false` via `gh api`; fail with descriptive error if not
2. **Checkout** — `actions/checkout@v4` with `fetch-depth: 0` and `fetch-tags: true`
3. **Write cliff.toml** — bash heredoc writing bundled config to working directory
4. **Get current tag** — `git tag --list 'v*.*.*' --sort=-version:refname | head -1`
5. **Install git-cliff** — `orhun/git-cliff-action@v4` with `args: --bumped-version`
6. **Check if release needed** — direct binary call; sets `skip` and `version` outputs
7. **Generate changelog** — `orhun/git-cliff-action@v4`; use `steps.*.outputs.content` for multiline
8. **Create GitHub Release** — `gh release create`
9. **Update alias tags** — force-update `vN` and `vN.M` and force-push

### `dependabot-major-prefix` job steps

1. **Guard** — `if: github.actor != 'dependabot[bot]'` — exits 0 early (not a failure)
2. **Fetch metadata** — `dependabot/fetch-metadata@v2`
3. **Rewrite title** — bash substitution `${PR_TITLE/fix(deps)/feat(deps)!}`; update via `gh pr edit`

---

## Known gotchas

### git-cliff action outputs are broken

`orhun/git-cliff-action@v4` has a jq bug that causes `outputs.stdout` and `outputs.version` to be unreliable. Do NOT use them.

- To get the bumped version: export PATH, call `git-cliff --bumped-version` directly
- PATH to add: `$RUNNER_TEMP/git-cliff/bin` — the binary is NOT added to `$GITHUB_PATH` automatically
- For changelog body: use `steps.<id>.outputs.content` (heredoc format, works for multiline)

### Alias tags must be excluded

git-cliff and `git tag --list` must both exclude alias tags like `v1` and `v1.2`.

- `tag_pattern = "v[0-9]+\\.[0-9]+\\.[0-9]+"` in cliff.toml restricts git-cliff to full semver tags only
- `git tag --list 'v*.*.*'` glob pattern excludes single-segment and two-segment tags

Never use `git describe` or `git tag --sort` without the glob filter — alias tags will corrupt version detection.

### Breaking change syntax

git-cliff recognizes `feat(scope)!:` as a breaking change. It does NOT recognize `feat!(scope):`. Always use the `!` after the scope (or after the type if there is no scope).

### pull_request_target injection risk

The `dependabot-major-prefix` job uses `pull_request_target`, which runs with elevated write permissions. Never interpolate `github.event.pull_request.title` directly in a `run:` block. Always pass it through `env:`:

```yaml
env:
  PR_TITLE: ${{ github.event.pull_request.title }}
run: |
  NEW_TITLE="${PR_TITLE/fix(deps)/feat(deps)!}"
```

This job is safe because it never checks out PR code — it only reads event metadata and edits the PR title.

---

## Semver mapping

| Conventional commit | Bump |
|---------------------|------|
| `feat` | minor |
| `fix` | patch |
| `fix(deps)` | patch (grouped as Dependencies) |
| `feat(scope)!:` or `BREAKING CHANGE` footer | major |
| `chore`, `ci`, `style`, `test`, `build` | none (skipped) |

`features_always_bump_minor = true` and `breaking_always_bump_major = true` are set explicitly in cliff.toml.

---

## Squash merge requirement

**All PRs must be merged via squash merge.**

The `dependabot-major-prefix` job rewrites the PR title. That rewritten title becomes the commit message only when squash merging. A regular merge commit preserves original commit messages, causing git-cliff to see `fix(deps):` instead of `feat(deps)!:` and produce the wrong version bump.

The `release` job enforces this at runtime: it checks `allow_merge_commit == false` via the GitHub API and fails with an error if merge commits are enabled on the repository.

---

## Dependabot configuration

`dependabot.yml` targets `github-actions` ecosystem only. All updates use `fix(deps):` as the commit prefix. Major version bumps get their PR title rewritten to `feat(deps)!:` by the `dependabot-major-prefix` workflow job.
