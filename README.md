# buildscad-workflows

Shared GitHub reusable workflows for [buildscad](https://github.com/dduxx/buildscad)-based OpenSCAD projects.

## Workflows

| Workflow | Purpose |
|----------|---------|
| [`ci.yml`](.github/workflows/ci.yml) | Builds the project on pull requests |
| [`release.yml`](.github/workflows/release.yml) | Tags, bumps `buildscad.properties`, and publishes a release on merge to `main` |

## Inputs

### `ci.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `python_version` | string | `"3.13"` | Python version used to run buildscad |
| `buildscad_version` | string | `"v1.4.1"` | buildscad version (git tag) to install |
| `openscad` | string | `"openscad"` | OpenSCAD snap package (`openscad` or `openscad-nightly`) |

### `release.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `python_version` | string | `"3.13"` | Python version used to run buildscad |
| `buildscad_version` | string | `"v1.4.1"` | buildscad version (git tag) to install |
| `openscad` | string | `"openscad"` | OpenSCAD snap package (`openscad` or `openscad-nightly`) |
| `build_artifacts` | boolean | `true` | Build and attach the `build/` directory to the release |

## Usage

Add a thin wrapper workflow to each project that calls the shared workflow here.

### CI

Often used for projects that need to execute the build as part of a PR for testing and confirming
that the changes do successfully build:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:

jobs:
  ci:
    uses: dduxx/buildscad-workflows/.github/workflows/ci.yml@v1.2.3
```

### Release

Projects that build and attach artifacts:

```yaml
# .github/workflows/release.yml
name: Release

on:
  pull_request:
    branches: [main]
    types: [closed]

jobs:
  release:
    if: github.event.pull_request.merged == true
    permissions:
      contents: write
    uses: dduxx/buildscad-workflows/.github/workflows/release.yml@v1.2.3
```

Projects that only version and release, without build artifacts:

```yaml
# .github/workflows/release.yml
name: Release

on:
  pull_request:
    branches: [main]
    types: [closed]

jobs:
  release:
    if: github.event.pull_request.merged == true
    permissions:
      contents: write
    uses: dduxx/buildscad-workflows/.github/workflows/release.yml@v1.2.3
    with:
      build_artifacts: false
```

To use OpenSCAD nightly instead of stable, add `with: { openscad: openscad-nightly }`.

## Versioning

Releases follow semantic versioning and are determined from commit messages since the last tag:

| Bump | Trigger |
|------|---------|
| Major | `BREAKING CHANGE` or a `type!: message` commit |
| Minor | `feat:` or `feat(scope):` |
| Patch | `fix:` or `perf:` |

Any other commit (`chore:`, `docs:`, `refactor:`, etc.) does not bump the version. If there are no version-affecting commits since the last tag, the release is skipped entirely.

## Semantic-release

This repository itself is versioned with [semantic-release](https://semantic-release.gitbook.io/). On every push to `main`, [`.github/workflows/publish.yml`](.github/workflows/publish.yml) runs `npx semantic-release`, which:

- Analyzes commits since the last release (using the same conventional-commit rules above)
- Creates a Git tag (`vX.Y.Z`) and a GitHub release with generated notes

Configuration lives in [`.releaserc`](.releaserc); plugins are the commit analyzer, release-notes generator, and GitHub publisher (`@semantic-release/npm` is intentionally omitted — this is not an npm package). Tags are immutable exact versions, so consumers should pin `uses:` to the specific tag they need.
