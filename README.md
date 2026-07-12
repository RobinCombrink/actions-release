# actions-release

Reusable GitHub Actions workflow that cuts a release on every push to `main`: version, tag,
and GitHub release derived from [Conventional Commits](https://www.conventionalcommits.org)
via [semantic-release](https://github.com/semantic-release/semantic-release). No manual
steps, no release PRs.

Part of a fleet of independently versioned reusable workflows
(`actions-*` repos on this account). Each fleet repo releases itself with this workflow —
including this one.

## Usage

`.github/workflows/release.yml` in the calling repo:

```yaml
name: release

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  release:
    uses: RobinCombrink/actions-release/.github/workflows/release.yml@<commit-sha> # vX.Y.Z
```

Pin the exact commit SHA of the release you want (the `# vX.Y.Z` comment keeps it readable);
Dependabot bumps SHA and comment together.

## Behavior

- Breaking changes cut a major release — both the `type!:` shorthand and the
  `BREAKING CHANGE:` footer are recognized. `feat:` cuts a minor.
- Every other commit type cuts a patch release, except `docs:` and `test:`, which do not
  release.
- The semantic-release configuration is bundled inside the workflow — callers carry no
  `.releaserc`. Shared policy rides this workflow's version; a caller-local
  `.releaserc*`/`release.config.*` overrides the bundled config if you need to diverge.
- Release notes are generated from the commit history since the previous tag.

## Requirements

- Commit messages follow Conventional Commits (enforced in this fleet by
  [actions-conventional-commits](https://github.com/RobinCombrink/actions-conventional-commits)).
- `permissions: contents: write` in the caller, as shown above.
