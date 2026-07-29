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

### Attaching build artifacts

Pass `artifact_name` to attach a build artifact to the release. The artifact must be uploaded
by an earlier job **in the same workflow run**, so the release job lives in the workflow that
builds it rather than in a standalone `release.yml`:

```yaml
jobs:
  build:
    # ... produces the binaries, then:
    #   - uses: actions/upload-artifact@<sha>
    #     with:
    #       name: release-binaries

  release:
    needs: [build]
    if: github.ref == 'refs/heads/main'
    uses: RobinCombrink/actions-release/.github/workflows/release.yml@<commit-sha> # vX.Y.Z
    with:
      artifact_name: release-binaries
```

The artifact is unpacked into `release-artifacts/` before semantic-release runs. Gating the
release job behind every validation job is the caller's job — `needs:` is what stops a release
firing on a build that has not passed its checks.

### Stamping the version into a build

A build that bakes its version in needs that version before it compiles, which is earlier than
the release exists. Call the workflow twice: once with `dry_run` to learn the version, then
normally to cut the release.

```yaml
jobs:
  version:
    uses: RobinCombrink/actions-release/.github/workflows/release.yml@<commit-sha> # vX.Y.Z
    with:
      dry_run: true

  build:
    needs: [version]
    # stamp ${{ needs.version.outputs.version }} into the build, then upload-artifact

  release:
    needs: [build]
    uses: RobinCombrink/actions-release/.github/workflows/release.yml@<commit-sha> # vX.Y.Z
    with:
      artifact_name: release-binaries
```

`dry_run` creates no tag and no release. Both calls read the same release rules, so the version
the build stamps is the version the release cuts. `version` is empty when the commits since the
last release do not warrant one — branch on it to skip the build and release jobs.

## Behavior

- Breaking changes cut a major release — both the `type!:` shorthand and the
  `BREAKING CHANGE:` footer are recognized. `feat:` cuts a minor.
- Every other commit type cuts a patch release, except `docs:` and `test:`, which do not
  release.
- The semantic-release configuration is bundled inside the workflow — callers carry no
  `.releaserc`. Shared policy rides this workflow's version; a caller-local
  `.releaserc*`/`release.config.*` overrides the bundled config if you need to diverge.
- Release notes are generated from the commit history since the previous tag.
- `artifact_name` attaches its contents via the bundled config. A caller that overrides the
  bundled config replaces it wholesale, so it must declare `release-artifacts/**` in its own
  `@semantic-release/github` assets — the workflow still downloads the artifact, but only the
  config decides what gets attached.
- Only the plugins bundled with semantic-release are available (`commit-analyzer`,
  `release-notes-generator`, `github`, `npm`). A caller-local config referencing anything else,
  such as `@semantic-release/exec` or `@semantic-release/git`, fails to load.

## Requirements

- Commit messages follow Conventional Commits (enforced in this fleet by
  [actions-conventional-commits](https://github.com/RobinCombrink/actions-conventional-commits)).
- `permissions: contents: write` in the caller, as shown above.
