# Repo administration

## Contents

- [Overview](#overview)
- [Repository layout](#repository-layout)
- [Dependency management](#dependency-management)
- [CI workflows](#ci-workflows)
- [Test coverage](#test-coverage)
- [Python version support](#python-version-support)
- [Releases](#releases)
  - [Standard Python release process](#standard-python-release-process)
  - [tskit and kastore releases](#tskit-and-kastore-releases)
- [conda-forge packages](#conda-forge-packages)
- [The tskit.dev website](#the-tskitdev-website)

## Overview

This document describes the standard layout, tooling, and CI conventions for
core tskit-dev Python packages.

**Core principles:**

- **Consistency.** All repos follow the same layout and conventions, making it
  easy to work across the ecosystem and reducing the maintenance burden of each
  individual repo.
- **Centralised CI logic.** Shared workflows and composite actions in this repo
  (`tskit-dev/.github`) avoid duplication. Individual repos call shared workflows via
  a pinned version tag, e.g. `uses: tskit-dev/.github/.github/workflows/<name>.yml@v15`
  (see [CI workflows](#ci-workflows) for the versioning policy).
- **Reproducibility.** All dependencies are managed exclusively with `uv` and
  locked via committed `uv.lock` files. Lint tools are pinned to exact versions
  so results are identical locally and in CI.
- **Minimal dependencies.** Dependencies should be declared in the appropriate
  group and pinned only where necessary (lint tools being the main exception).

There are three categories of package in this ecosystem:

- **Python-only:** tszip, tstrait, tsdate, tsbrowse, pyslim, tscompare, sc2ts
- **Python+C (external C library):** tskit, kastore — the C library has a public API,
  is independently versioned, and is consumed by other projects
- **Python+C (internal C library):** msprime, tsinfer — the C code is bundled as
  part of the Python package and is not released separately

The rules differ slightly between these categories, as described below.


## Repository layout

### Python-only repos

`pyproject.toml` and `uv.lock` live at the root. Tests live in `tests/`.
`prek.toml` lives at the root.

### Python+C repos (external C library)

The Python package and its `pyproject.toml` and `uv.lock` live in a `python/`
subdirectory. The standalone C library lives in `c/`.
`prek.toml` lives at the root.

Documentation (`docs/`) lives at the repository root and includes a `doxygen/`
subdirectory for generating C API docs.

### Python+C repos (internal C library)

The Python package and its `pyproject.toml` and `uv.lock` live in the
root, along with `prek.toml`. The C extension code lives in `lib/`.

There is no separate C API documentation or independent C release workflow.

### All Python+C repos

All shared workflows that accept a `pyproject-directory` input should be passed
`python` for these repos.

## Dependency management

We use `uv` exclusively for Python package management.

### Required groups

Every repo must define these dependency groups in `pyproject.toml`:

- **test** – pytest and test-only dependencies
- **docs** – JupyterBook and Sphinx extensions
- **lint** – linting tools (see below)
- **packaging** – `twine` and `validate-pyproject[all]`

Python+C repos additionally require:

- **wheels** – `cibuildwheel`

A **dev** group that includes all groups except `wheels` is recommended for
local development convenience.

### Lint pinning

Lint tools must be pinned to exact versions in the `lint` group to ensure
deterministic results:

```toml
[dependency-groups]
lint = [
    "clang-format==x.y.z",  # Python+C repos only
    "ruff==x.y.z",
    "prek==x.y.z",
]
```

Ruff can be updated periodically, with any required new fixes or "ignores"
included.

### Lock files

A `uv.lock` file must be maintained and committed to the repository. Run
`uv lock` after any change to dependencies. For Python+C repos the lock file
lives alongside `pyproject.toml` in `python/`.

Lock files should be updated periodically (say every 6 months) as part of routine maintenance.
This provides the opportunity to detect and fix problems caused by upstream packages in
a controlled fashion.


## CI workflows

Each repo calls shared workflows defined in this repository. Shared workflows are
versioned via git tags on this repository (`v5`, `v6`, … `v14`, etc.). Calling repos
pin to a specific tag, e.g.:

```yaml
uses: tskit-dev/.github/.github/workflows/python-tests.yml@v15
```

When a change is made to a shared workflow or composite action in this repo, create a
new tag (incrementing the version number) to publish the change. While developing
the changes you can use `@main` instead of the tagged version to pick up the
latest changes. Calling repos must
then update their pin to pick it up — this is a deliberate opt-in. To update a repo,
find all occurrences of the old tag in its `.github/workflows/` files and bump them to
the new version.

The available workflows are:

| Workflow | Purpose |
|---|---|
| `docs.yml` | Build and deploy documentation |
| `python-tests.yml` | Run the pytest suite with coverage |
| `lint.yml` | Run prek linting |
| `python-packaging.yml` | Validate packaging (sdist, wheel, twine) |
| `build-wheels.yml` | Build binary wheels (Python+C only) |
| `python-c-tests.yml` | Low-level C extension tests (Python+C only) |
| `c-tests.yml` | C unit tests with valgrind (Python+C only) |


### Docs

Documentation is built with JupyterBook. Each repo has a `docs/` directory
containing a `Makefile`, `build.sh`, and JupyterBook configuration.

**Local build:**

```
cd docs && make
```

This calls `build.sh`, which runs JupyterBook with verbose output and error
reporting. For Python+C repos, the Makefile also runs Doxygen before the build
(only when C headers have changed, so repeated builds are fast). Clean build
output with `make clean`.

**CI build:**

The `docs.yml` shared workflow calls the `build-docs` composite action
(`.github/actions/build-docs`), which:
1. Installs the `docs` dependency group
2. Substitutes the `__PKG_VERSION__` placeholder in `docs/_config.yml` with
   the installed package version
3. Optionally runs a pre-build command (e.g. for Doxygen)
4. Runs JupyterBook

On merge to `main`, the workflow triggers a rebuild of the tskit.dev website.
Two documentation versions are maintained: **latest** (built from `main`) and
**stable** (built from the most recent release tag).

**Requirements:**

- A `docs` dependency group in `pyproject.toml`
- A `__PKG_VERSION__` placeholder string in `docs/_config.yml`

For Python+C repos, pass Doxygen setup as inputs to the shared workflow:

```yaml
with:
  additional-apt-packages: doxygen
  pre-build-command: cd docs/doxygen && doxygen
```


### Tests

The `python-tests.yml` workflow runs pytest with branch coverage and parallel
execution, uploading results to CodeCov with the `python-tests` flag. Tests
should cover a minimum and a recent Python version across Linux, macOS, and
Windows.

Some repos have their own version of the tests.yml and don't use the shared
workflow because bespoke actions need to be taken or difficult system libraries
need to be installed.


### Lint

The `lint.yml` workflow installs only the `lint` group (no system dependencies
or package installation required) and runs prek against all files. The
`prek.toml` at the repo root defines all rules using only `builtin` and `local`
hooks for long-term stability.

Standard hooks cover: large file detection, merge conflict markers, line
endings, YAML/TOML validation, `ruff check`, and `ruff format`. Python+C repos
additionally run `clang-format` on C files.

Install prek as a pre-commit hook with `uv run prek install`. If local results
differ from CI, run `uv run prek cache clean`.


### Packaging

The `python-packaging.yml` workflow validates `pyproject.toml`, builds sdist
and wheel artifacts, and checks them with `twine --strict`. Pass a
`cli-test-cmd` input (e.g. `tszip --help`) for packages with entry points to
verify the entry point is correctly declared.


### Python+C specific

**`python-c-tests.yml`** builds the C extension with coverage instrumentation
and runs focused low-level interface tests, uploading results to CodeCov with
the `c-python` flag.

**`c-tests.yml`** runs the C unit tests under gcc (with coverage), clang, and
valgrind. Coverage is uploaded with the `C` flag.

### Required secrets and environments

Each repo needs the following configured before releases, coverage uploads
and website rebuild triggers to work:

- **`CODECOV_TOKEN`** GitHub Actions secret — for coverage uploads (see [Test coverage](#test-coverage)).
- **`ADMINBOT_TOKEN`** GitHub Actions secret — used by the shared `docs.yml` workflow to
  fire a `repository_dispatch` to tskit-site after a successful docs build on `main`.
  Without this, merging documentation changes will not trigger a site rebuild.
- **`release` GitHub Actions environment** — required by `wheels.yml` for the Trusted
  Publisher OIDC token (`id-token: write`). Create this environment in the repo's
  Settings → Environments.
- **Trusted Publisher on PyPI and TestPyPI** — configure a Trusted Publisher entry on
  both [pypi.org](https://pypi.org) and [test.pypi.org](https://test.pypi.org) pointing
  to the repo and the `release` environment. Without this, the `wheels.yml` upload steps
  will fail even if the workflow runs successfully.


## Test coverage

Coverage is monitored by CodeCov. The `CODECOV_TOKEN` secret must be set in
each repo's GitHub Actions secrets. Verify uploads are working by checking the
Actions logs after a CI run.

Flags used: **python-tests**, **c-python** (Python+C only), **C** (Python+C
only).

### `codecov.yml`

Each repo should include a `codecov.yml` at the repository root. This file
controls how CodeCov interprets and reports coverage data.

## Python version support

When adding or removing support for a Python version, update the following in every
affected repo:

### All repos

**`pyproject.toml`** (or `python/pyproject.toml` for Python+C repos):

- `requires-python` — set the floor version, e.g. `">=3.12"`.
- `classifiers` — add or remove the corresponding
  `"Programming Language :: Python :: 3.X"` entry. This is informational (displayed
  on PyPI) but should be kept in sync with `requires-python` and the test matrix.
- `[tool.ruff] target-version` — must match the minimum supported version (e.g.
  `"py312"`). Forgetting this means ruff silently permits syntax that is invalid on
  the stated floor version.

**`.github/workflows/tests.yml`**:

- Update the `matrix.python` list. Convention is to test the oldest and newest
  supported versions; intermediate versions do not need individual matrix entries.

### Python+C repos (cibuildwheel)

**`[tool.cibuildwheel]` in `pyproject.toml`**:

- Update the `build` list, e.g. `["cp311-*", "cp312-*", "cp313-*"]`. This is what
  controls which wheels are built — it is not inferred from the classifiers.
- When adding a new Python version, check that it is supported by the pinned
  `cibuildwheel` release in the `wheels` dependency group. Pre-release Python versions
  may require a newer cibuildwheel or explicit opt-in.
- After updating, push to the `test-publish` branch and verify that all new wheels
  build and pass their smoke tests before making a release. Wheel builds are not
  otherwise exercised in CI between releases, so this step is important.

### Workflow files

Check all `.github/workflows/` files in the repo for any explicit references to the
old minimum Python version and update them to the new minimum. In particular:

- Any bespoke workflow steps that install or invoke a specific Python version directly.

## Releases

Python releases use the `wheels.yml` workflow via the Trusted Publisher PyPI mechanism.
Because Trusted Publisher requires the upload to originate from the source repository,
this workflow is not shared — each repo maintains its own copy.


Python+C repos additionally use the shared `build-wheels.yml` workflow, which uses
`cibuildwheel` to build binary wheels across Linux, macOS, and Windows. All
configuration lives in `pyproject.toml` under `[tool.cibuildwheel]`.

tskit and kastore also have a repo-specific `release-c.yml` workflow solely for C API
releases (see below). This is separate from the Python release process.


### Version management

Most repos use `setuptools_scm` for versioning, which means that versions are automatically
created from git tags. This approach doesn't work for kastore and tskit, though, and so they
maintain version numbers manually.

### Standard Python release process

> **tskit and kastore:** follow the [tskit and kastore releases](#tskit-and-kastore-releases)
> section below instead of this standard process.

1. Prepare a PR that updates the CHANGELOG with the correct version number. Merge this PR.
2. Push to the `test-publish` branch on upstream to trigger `wheels.yml`, which builds
   release artifacts and publishes them to TestPyPI. Check the "Publish Python release"
   action succeeds. This step is especially important for repos with binary wheels
   (msprime, tsinfer, tskit, kastore) as the wheel-building step is **not tested** between releases.
3. Once the TestPyPI upload succeeds, delete the `test-publish` branch. Go to the
   "Releases" section on GitHub and click "Draft new release". Enter the version number
   in the tag box and click "create new tag" (the tag is created when the release is
   published). Fill in the release body with the CHANGELOG contents (the "latest" docs
   on tskit.dev have most of the formatting done already). Click "Publish release" and
   confirm the "Publish Python release" action succeeds on PyPI.
4. Open a post-release PR that opens a new section in the CHANGELOG.

### tskit and kastore releases

tskit and kastore each have an independently versioned C library and a Python package,
and do not use `setuptools_scm`. The steps below apply to both repos.

#### C API release

`release-c.yml` handles C API releases only — it is triggered by tags containing `C_`
and has no effect on the Python package.

1. Prepare a PR that:
   - Updates the version macros in the C header and `c/VERSION.txt`
   - Updates `c/CHANGELOG.rst` with the release date and version; check completeness
     by comparing `git log --follow --oneline -- c` with
     `git log --follow --oneline -- c/CHANGELOG.rst`
2. Merge the PR.
3. Tag and push:
   ```bash
   git fetch upstream
   git checkout upstream/main
   git tag -a C_MAJOR.MINOR.PATCH -m "C API version C_MAJOR.MINOR.PATCH"
   git push upstream C_MAJOR.MINOR.PATCH
   ```
4. After a couple of minutes, `release-c.yml` will build the release tarball and create
   a draft release on the repo's GitHub releases page. Update the release body with the
   changelog contents and publish.
5. Open a post-release PR that opens a new section in `c/CHANGELOG.rst` and closes the
   GitHub issue milestone.

#### Python release

Follow the standard Python release process above, with two differences:

- In step 1, also set the version in `python/<packagename>/_version.py` (these repos
  do not use `setuptools_scm`).
- In step 4, also bump the version in `python/<packagename>/_version.py` to
  `MAJOR.MINOR.PATCH.dev0`.


## conda-forge packages

Several tskit-dev packages are distributed via conda-forge:
tskit, msprime, kastore, tstrait, tszip, and tsinfer.
Each has a feedstock repository at `https://github.com/conda-forge/<package>-feedstock`.

### How conda-forge updates work

conda-forge bots monitor PyPI and automatically open a PR on the feedstock repository
when a new release is published. This PR updates the version number and source hash in
`recipe/meta.yaml`. In most cases the PR can simply be merged with no manual
intervention — the bot handles the version bump and rebuilds.

### What requires manual attention

Dependency changes are **not** picked up automatically. Whenever a package's
dependencies change (additions, removals, or version constraint changes), the
corresponding feedstock's `recipe/meta.yaml` must be updated manually:

- `run:` requirements must reflect the current `[project.dependencies]` in
  `pyproject.toml`.


## The tskit.dev website

The [tskit-site](https://github.com/tskit-dev/tskit-site) repository is the source for
the [tskit.dev](https://tskit.dev) website, including the landing page and the hosted
documentation for all packages in the ecosystem. It is a Jekyll site deployed to GitHub
Pages via the `gh-pages` branch.

### Site build overview

The `deploy.yml` workflow orchestrates the full build. It runs on:

- Every push to `main` (live deployment)
- Every pull request (preview artifact only — not deployed)
- A daily scheduled cron job (keeps the site up to date with any upstream changes)
- A `repository_dispatch` event (triggered automatically by per-repo doc builds — see below)
- Manual `workflow_dispatch`

`deploy.yml` calls three sub-workflows in parallel, then assembles and deploys the result:

- **`build-core.yml`** — builds the Jekyll site itself (landing page, software listings,
  news, etc.) using Ruby/Grunt
- **`build-docs.yml`** — builds per-package documentation for both `latest` (from `main`)
  and `stable` (from the most recent release tag) versions
- **`build-extras.yml`** — imports the tutorials site (from `tskit-dev/tutorials`), rust
  tutorials (from `tskit-dev/tskit-rust`), and builds the tskit-explore JupyterLite app

The assembled site is deployed to `gh-pages`. Package docs land at
`/<package>/docs/latest` and `/<package>/docs/stable`; a redirect page at
`/<package>/docs/index.html` points visitors to the `stable` version by default.

### How per-repo doc builds integrate

Each repo's own `docs.yml` uses the shared `docs.yml` workflow defined in this
repository. On merge to `main`, after a successful build, the shared workflow fires a
`repository_dispatch` event to `tskit-site`, which triggers a full site rebuild. This
means that merging a documentation change to any repo automatically propagates to
tskit.dev within a few minutes.

The `build-docs.yml` in tskit-site uses the same shared `build-docs` composite action
(`.github/actions/build-docs`) that the per-repo workflow uses, and passes the same
inputs (`pyproject-directory`, `additional-apt-packages`, `pre-build-command`). Builds
are cached by commit SHA, so a package's docs are only rebuilt when its `main` branch
has a new commit.

### What changing the local docs build means

When making changes that affect how docs are built (e.g. new APT dependencies, a new
pre-build command, changes to `build.sh` or the `docs` dependency group), be aware that
the build runs in two places:

1. **The repo's own `docs.yml`** — configured via inputs to the shared `docs.yml`
   workflow in the repo's `.github/workflows/docs.yml`.
2. **tskit-site's `build-docs.yml`** — has a matrix entry per package that passes the
   same inputs. This file must be updated in the `tskit-dev/tskit-site` repo to match.

If these two are out of sync, the per-repo CI will pass but the tskit-site build will
fail or produce incorrect output. The relevant matrix entry in `build-docs.yml` is
identified by the package name and specifies `additional-apt-packages`,
`pre-build-command`, `pyproject-directory`, and a `cache-version` (bump this whenever
the build environment changes to force a cache miss).

Changes to `build.sh`, `docs/_config.yml`, or the `docs` dependency group are picked up
automatically on the next run with no changes needed to tskit-site.

### Stable and latest versions

`latest` is built from `main` on every new commit. `stable` is built from the most
recent release tag, determined by sorting all tags and selecting the newest that does
not contain `b`, `a`, or `C_` in its name (i.e. no pre-releases or C API tags).

The critical distinction: **stable docs are built from the repo code at the tag, but
using the current shared build infrastructure** — the pinned `build-docs` action version
and whatever `additional-apt-packages` and `pre-build-command` are currently in the
tskit-site matrix entry for that package. The stable build does not use `main`'s
`build.sh` or `docs/` content; it uses whatever was committed at the tag.

This means:

- **Content and structural changes** (new pages, edited text, updated notebooks) only
  affect `latest` until a new release is tagged. `stable` continues to serve the old
  content. This is expected behaviour.

- **Build environment changes** (new APT packages, changed `pre-build-command`) require
  updating the tskit-site matrix entry, which applies to **both** `latest` and `stable`.
  If the stable release's code is not compatible with the updated build environment,
  the stable build will break.

- **Bumping `cache-version`** in the tskit-site matrix entry forces a cache miss for
  both versions. This is required when the build environment changes, but it also means
  the stable docs will be fully rebuilt — so the same compatibility concern applies.

When making a change that alters the build environment, verify that the stable version
still builds correctly, or make a new release first so that `stable` and `latest` point
to the same code.
