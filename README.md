# buildscad-workflows

Shared GitHub reusable workflows for [buildscad](https://github.com/dduxx/buildscad)-based OpenSCAD projects.

## Workflows

| Workflow | Purpose |
|----------|---------|
| [`ci.yml`](.github/workflows/ci.yml) | Builds the project on pull requests |
| [`build.yml`](.github/workflows/build.yml) | Builds (optionally with a glob filter) and optionally zips the `build/` directory |
| [`release.yml`](.github/workflows/release.yml) | Tags, bumps `buildscad.properties`, and publishes a release on merge to `main` |

## Inputs

### `ci.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `python_version` | string | `"3.13"` | Python version used to run buildscad |
| `buildscad_version` | string | `"v1.5.0"` | buildscad version (git tag) to install |
| `openscad` | string | `"openscad"` | OpenSCAD snap package (`openscad` or `openscad-nightly`) |

### `build.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `python_version` | string | `"3.13"` | Python version used to run buildscad |
| `buildscad_version` | string | `"v1.5.0"` | buildscad version (git tag) to install |
| `openscad` | string | `"openscad"` | OpenSCAD snap package (`openscad` or `openscad-nightly`) |
| `filter` | string | `""` | Glob filter to select which assemblies to build |
| `name` | string | `""` | Optional name prefix for the output zip archive |
| `version` | string | `""` | Version used in the zip name (falls back to `BUILDSCAD_VERSION`) |
| `zip_artifacts` | boolean | `false` | Build and zip the `build/` directory into an archive |

### `release.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `python_version` | string | `"3.13"` | Python version used to run buildscad |
| `buildscad_version` | string | `"v1.5.0"` | buildscad version (git tag) to install |
| `openscad` | string | `"openscad"` | OpenSCAD snap package (`openscad` or `openscad-nightly`) |
| `build_artifacts` | boolean | `true` | Build and attach the `build/` directory to the release |
| `artifacts` | string | `'[{"filter":"","name":""}]'` | JSON array of artifacts to build, each with optional `filter` (glob) and `name` (zip prefix) |

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
    uses: dduxx/buildscad-workflows/.github/workflows/ci.yml@v1.0.1
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
    uses: dduxx/buildscad-workflows/.github/workflows/release.yml@v1.0.1
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
    uses: dduxx/buildscad-workflows/.github/workflows/release.yml@v1.0.1
    with:
      build_artifacts: false
```

To use OpenSCAD nightly instead of stable, add `with: { openscad: openscad-nightly }`.

### Build

Projects that want a standalone build (optionally filtered and zipped) outside of a release:

```yaml
# .github/workflows/build.yml
name: Build

on:
  workflow_dispatch:

jobs:
  build:
    uses: dduxx/buildscad-workflows/.github/workflows/build.yml@v1.0.1
    with:
      filter: "scad/group1/*.scad"
      name: group1
      zip_artifacts: true
```

### Multiple release artifacts

Large projects can build several filtered archives in parallel and attach them all to a single release by passing a JSON array of `{filter, name}` objects:

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
    uses: dduxx/buildscad-workflows/.github/workflows/release.yml@v1.0.1
    with:
      artifacts: '[{"filter":"scad/group1/*.scad","name":"group1"},{"filter":"scad/group2/*.scad","name":"group2"}]'
```

Each entry produces a zip named `<name>_v<version>.zip` (e.g. `group1_v1.2.3.zip`). When `name` is omitted, the archive is named `v<version>.zip`.


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
