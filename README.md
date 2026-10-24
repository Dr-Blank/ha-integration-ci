# ha-integration-ci

Reusable GitHub Actions workflows and a Copier template for Home Assistant custom integrations distributed through HACS, and for the Python libraries they depend on, published to PyPI.

| Workflow | For | Runs |
| --- | --- | --- |
| `test.yaml` | both | `uv sync --locked --all-extras --dev`, then `prek run --all-files --hook-stage manual` |
| `validate.yaml` | integrations | hassfest and HACS validation |
| `release.yaml` | integrations | checks the tag against `pyproject.toml` and `custom_components/*/manifest.json`, runs `test` and `validate` at the tag, then `github-release` |
| `library-build.yaml` | libraries | checks the tag against `pyproject.toml`, runs `test` at the tag, then `uv build` into the `dist` artifact |
| `github-release.yaml` | both | creates the GitHub release, optionally with an artifact's files attached; a re-run for an existing release only refreshes those files |

The release notes list the commits since the previous tag grouped by Conventional Commits type, so commits pushed straight to `main` are covered, followed by GitHub's generated notes for merged pull requests.

## Repository layout expected

- Integrations: exactly one `custom_components/<domain>/manifest.json`.
- Libraries: a package that `uv build` can build, and a PyPI trusted publisher for the repository's `publish.yaml` and its `pypi` environment.
- `pyproject.toml` with a static `[project] version`, and a tracked `uv.lock`.
- A `.pre-commit-config.yaml` run by [prek](https://github.com/j178/prek), with slow hooks such as pytest on the `manual` stage. CI runs no other check, so pytest has to be a hook.
- Release tags named `vX.Y.Z`. A `chore(release): ...` commit is left out of the notes.

## Usage

Each repository keeps its own triggers and `permissions`, since a called workflow cannot be granted more than its caller has. Pin to a full commit SHA with the version as a comment, so Renovate can bump it.

The examples below are for an integration; the library equivalents are under [Libraries](#libraries).

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

### Libraries

`.github/workflows/tests.yaml` runs `test.yaml` once per supported Python version:

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.12", "3.13", "3.14"]
    uses: Dr-Blank/ha-integration-ci/.github/workflows/test.yaml@<sha> # v1.3.0
    with:
      python-version: ${{ matrix.python-version }}
```

`.github/workflows/publish.yaml` keeps the PyPI upload itself, because a trusted publisher names the workflow file and environment of the repository that uploads:

```yaml
name: Publish

on:
  push:
    tags:
      - "v*.*.*"
  workflow_dispatch:
    inputs:
      tag:
        description: "Existing tag to (re)publish, e.g. v1.2.3"
        required: true
        type: string

permissions: {}

concurrency:
  group: publish-${{ inputs.tag || github.ref_name }}
  cancel-in-progress: false

jobs:
  build:
    permissions:
      contents: read
    uses: Dr-Blank/ha-integration-ci/.github/workflows/library-build.yaml@<sha> # v1.3.0
    with:
      tag: ${{ inputs.tag || github.ref_name }}
      python-version: "3.14"

  publish:
    name: Publish to PyPI
    needs: [build]
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/p/<name>
    permissions:
      id-token: write
    steps:
      - uses: actions/download-artifact@<sha> # v8.0.1
        with:
          name: dist
          path: dist/
      - uses: pypa/gh-action-pypi-publish@<sha> # v1.14.2
        with:
          skip-existing: true

  github-release:
    needs: [build, publish]
    permissions:
      contents: write
    uses: Dr-Blank/ha-integration-ci/.github/workflows/github-release.yaml@<sha> # v1.3.0
    with:
      tag: ${{ inputs.tag || github.ref_name }}
      artifact: dist
```

### Renovate

`.github/renovate.json`: copy [`template/.github/renovate.json`](template/.github/renovate.json). Minor, patch and digest updates merge once CI passes; the integration's own requirements in `manifest.json` and `pyproject.toml` are grouped and always wait for review.

## Inputs

| Input | Workflows | Default |
| --- | --- | --- |
| `tag` | `release`, `library-build`, `github-release` (required) | |
| `ref` | `test`, `validate` | the triggering ref |
| `hacs-ignore` | `validate`, `release` | `""` |
| `python-version` | `test`, `release`, `library-build` | `"3.14"` |
| `artifact` | `github-release` | `""`, nothing attached |

## Pinned actions

`actions/checkout` and `astral-sh/setup-uv` are pinned to release SHAs and bumped by Renovate. `home-assistant/actions/hassfest` and `hacs/action` have no recent tags, so their SHAs are bumped by hand. Both start a container image that is not pinned (`ghcr.io/home-assistant/hassfest`, `ghcr.io/hacs/action:main`), so validation follows upstream.

## Copier template

The same repository is a [Copier](https://copier.readthedocs.io/) template for the files used outside Actions. The first question, `project_type`, picks the set:

| File | Integration | Library |
| --- | --- | --- |
| Caller workflows above | `tests.yaml`, `validate.yaml`, `release.yaml` | `tests.yaml` (matrix over `python_versions`), `publish.yaml` |
| `.github/renovate.json`, `.github/release.yml`, `script/release.sh`, `.vscode/tasks.json`, `.gitignore` | yes | yes |
| `pyproject.toml`, `.pre-commit-config.yaml` | kept in step with the template | created in a new repository only; afterwards the library owns its tool configuration |
| `hacs.json`, `script/generate_translations.py`, `script/link_devcontainer.sh` | yes | |
| `.editorconfig`, `.vscode/extensions.json`, `docs/publishing.md` | | yes |

New repository, or adopting the template in an existing one:

```sh
uvx copier copy --overwrite https://github.com/Dr-Blank/ha-integration-ci .
```

Pulling later template changes, as a diff to review:

```sh
uvx copier update
```

`copier update` merges three ways, so a line the template never changes keeps the repository's own value. An integration's `pyproject.toml` relies on this: `version`, `dependencies` and any extra dev dependencies belong to the repository, and the template only carries placeholders for them. After a first `copier copy --overwrite`, restore those lines (and any repository-specific `.gitignore` entries) before committing.

A library adopting the template keeps its `pyproject.toml` and `.pre-commit-config.yaml`, so its lint and type-check settings do not change. Add a pytest hook on the `manual` stage if the configuration has none, since `test.yaml` runs nothing else.

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
