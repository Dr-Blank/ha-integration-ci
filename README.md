# ha-integration-ci

Reusable GitHub Actions workflows for Home Assistant custom integrations distributed through HACS.

| Workflow | Runs |
| --- | --- |
| `test.yaml` | `uv sync --locked --all-extras --dev`, then `prek run --all-files --hook-stage manual` |
| `validate.yaml` | hassfest and HACS validation |
| `release.yaml` | checks the tag against `pyproject.toml` and `custom_components/*/manifest.json`, runs `test` and `validate` at the tag, then creates the GitHub release with notes grouped by Conventional Commits type |

## Repository layout expected

- Exactly one `custom_components/<domain>/manifest.json`.
- `pyproject.toml` with a static `[project] version`, and a tracked `uv.lock`.
- A `.pre-commit-config.yaml` run by [prek](https://github.com/j178/prek), with slow hooks such as pytest on the `manual` stage.
- Release tags named `vX.Y.Z`. A `chore(release): ...` commit is left out of the notes.

## Usage

Each integration repository keeps its own triggers and `permissions`, since a called workflow cannot be granted more than its caller has. Pin to a full commit SHA with the version as a comment, so Dependabot can bump it.

`.github/workflows/tests.yaml`:

```yaml
name: Test

on:
  workflow_dispatch: ~
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    uses: Dr-Blank/ha-integration-ci/.github/workflows/test.yaml@<sha> # v1.0.0
```

`.github/workflows/validate.yaml`:

```yaml
name: Validate

on:
  workflow_dispatch: ~
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 4 * * 1"

permissions:
  contents: read

jobs:
  validate:
    uses: Dr-Blank/ha-integration-ci/.github/workflows/validate.yaml@<sha> # v1.0.0
    with:
      # Only integrations shipped with core can be in home-assistant/brands.
      hacs-ignore: brands
```

`.github/workflows/release.yaml`:

```yaml
name: Release

on:
  push:
    tags:
      - "v*.*.*"
  workflow_dispatch:
    inputs:
      tag:
        description: "Existing tag to (re)release, e.g. v1.2.3"
        required: true
        type: string

permissions: {}

concurrency:
  group: release-${{ inputs.tag || github.ref_name }}
  cancel-in-progress: false

jobs:
  release:
    permissions:
      contents: write
    uses: Dr-Blank/ha-integration-ci/.github/workflows/release.yaml@<sha> # v1.0.0
    with:
      tag: ${{ inputs.tag || github.ref_name }}
      hacs-ignore: brands
```

`.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

## Inputs

| Input | Workflows | Default |
| --- | --- | --- |
| `tag` | `release` (required) | |
| `ref` | `test`, `validate` | the triggering ref |
| `hacs-ignore` | `validate`, `release` | `""` |
| `python-version` | `test`, `release` | `"3.14"` |

## Pinned actions

`actions/checkout` and `astral-sh/setup-uv` are pinned to release SHAs and bumped by Dependabot. `home-assistant/actions/hassfest` and `hacs/action` have no recent tags, so their SHAs are bumped by hand. Both start a container image that is not pinned (`ghcr.io/home-assistant/hassfest`, `ghcr.io/hacs/action:main`), so validation follows upstream.

## Versioning

Releases are tagged `vX.Y.Z`, and the `vX` tag is moved to the latest release of that major version. A breaking change to inputs or to the expected repository layout bumps the major version.
