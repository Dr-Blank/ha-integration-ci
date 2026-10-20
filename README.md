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

Each integration repository keeps its own triggers and `permissions`, since a called workflow cannot be granted more than its caller has. Pin to a full commit SHA with the version as a comment, so Renovate can bump it.

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

`.github/renovate.json`: copy [`template/.github/renovate.json`](template/.github/renovate.json). Minor, patch and digest updates merge once CI passes; the integration's own requirements in `manifest.json` and `pyproject.toml` are grouped and always wait for review.

## Inputs

| Input | Workflows | Default |
| --- | --- | --- |
| `tag` | `release` (required) | |
| `ref` | `test`, `validate` | the triggering ref |
| `hacs-ignore` | `validate`, `release` | `""` |
| `python-version` | `test`, `release` | `"3.14"` |

## Pinned actions

`actions/checkout` and `astral-sh/setup-uv` are pinned to release SHAs and bumped by Renovate. `home-assistant/actions/hassfest` and `hacs/action` have no recent tags, so their SHAs are bumped by hand. Both start a container image that is not pinned (`ghcr.io/home-assistant/hassfest`, `ghcr.io/hacs/action:main`), so validation follows upstream.

## Copier template

The same repository is a [Copier](https://copier.readthedocs.io/) template for the files used outside Actions: the caller workflows above, `.github/renovate.json`, `.github/release.yml`, `.pre-commit-config.yaml`, `.vscode/tasks.json`, `.gitignore`, `hacs.json`, `pyproject.toml` and the `script/` helpers (`release.sh`, `generate_translations.py`, `link_devcontainer.sh`).

New repository, or adopting the template in an existing one:

```sh
uvx copier copy --overwrite gh:Dr-Blank/ha-integration-ci .
```

Pulling later template changes, as a diff to review:

```sh
uvx copier update
```

`copier update` merges three ways, so a line the template never changes keeps the repository's own value. `pyproject.toml` relies on this: `version`, `dependencies` and any extra dev dependencies belong to the repository, and the template only carries placeholders for them. After a first `copier copy --overwrite`, restore those lines (and any repository-specific `.gitignore` entries) before committing.

## Versioning

Releases are tagged `vX.Y.Z`, and there is no moving `vX` tag: callers pin a SHA, and `copier update` needs every recorded tag to keep pointing at the commit it was rendered from. A breaking change to inputs or to the expected repository layout bumps the major version.

## Maintaining

Changing a shared workflow (`.github/workflows/`):

1. Commit to `main`, tag `vX.Y.Z` and push the tag.
2. Renovate in each integration repository opens a pull request that bumps the SHA pin, on its weekly run or straight away from the Dependency Dashboard issue. A minor or patch bump merges itself once CI passes; merge a major one by hand.
3. Bump the pin in `template/.github/workflows/` with the next template release. It lags one release because a commit cannot contain its own SHA; repositories already got the bump from Renovate, and `copier update` sees the same change on both sides.

Changing a template file (`template/`, `copier.yml`):

1. Commit to `main`, tag `vX.Y.Z` and push the tag.
2. In each integration repository run `uvx copier update`, review `git diff`, resolve any `<<<<<<< before updating` markers, then commit.

Bumping hassfest or the HACS action, which have no tags to track:

```sh
gh api repos/home-assistant/actions/commits/master --jq .sha
gh api repos/hacs/action/commits/main --jq .sha
```

Replace the SHAs in `.github/workflows/validate.yaml`, then release as a shared workflow change.

Never move or reuse a tag: `copier update` re-renders the version recorded in each repository, so a tag that now points elsewhere silently drops template changes.
