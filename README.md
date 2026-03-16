# release-action

![Last Commit](https://img.shields.io/github/last-commit/brianespinosa/release-action)
![License](https://img.shields.io/github/license/brianespinosa/release-action)

Reusable GitHub Actions workflow for automated semver releases, changelog generation, and alias tag management for composite action repositories.

## Usage

**`dependabot.yml`**

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    commit-message:
      prefix: "fix(deps):"
```

**`release.yml`** (consumer workflow)

```yaml
name: Release

on:
  push:
    branches: [main]
  pull_request_target:
    types: [opened, reopened]

permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    uses: brianespinosa/release-action/.github/workflows/release-workflow.yml@main
    with:
      trigger: ${{ github.event_name }}
    secrets: inherit
```

## Assumptions

- Squash merge is required; merge commits are not supported and the workflow enforces this at startup. Rebase merges are also not supported but are not enforced at runtime — consumers must disable them manually
- Not published to the GitHub Marketplace; personal use only
- Consuming repositories must have merge commits disabled and squash merge enabled — the workflow will fail on the first run if this is not the case
- Consuming repositories must use conventional commits for PR titles
- The workflow skips automatically when no version-bump-worthy commits exist since the last tag
- The `pull_request_target` trigger includes `reopened` intentionally — Dependabot PRs can be closed and reopened when the base branch changes, and the title rewrite must fire again in that case
