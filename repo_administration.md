# Repo administration

## Overview

This document describes the standard layout, tooling, and CI conventions for
core tskit-dev Python packages.

**Core principles:**

- **Consistency.** All repos follow the same layout and conventions, making it
  easy to work across the ecosystem and reducing the maintenance burden of each
  individual repo.
- **Centralised CI logic.** Shared workflows and composite actions in this repo
  (`tskit-dev/.github`) avoid duplication. Individual repos call into them via
  `uses: tskit-dev/.github/.github/workflows/<name>.yml@main` or
  `uses: tskit-dev/.github/.github/actions/<name>@main`.
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

The Python package and its `pyproject.toml` live in a `python/` subdirectory.
The standalone C library lives in `c/`. Tests are in `python/tests/`. `prek.toml`
and `uv.lock` live at the root.

Documentation (`docs/`) lives at the repository root and includes a `doxygen/`
subdirectory for generating C API docs.

### Python+C repos (internal C library)

The Python package and its `pyproject.toml` live in a `python/` subdirectory.
The C extension code lives in `lib/` within the `python/` subdirectory. Tests
are in `python/tests/`. `prek.toml` and `uv.lock` live at the root.

There is no separate C API documentation or independent C release workflow.
These repos (msprime, tsinfer) use `setuptools_scm` for versioning, so no manual
version file needs to be maintained.

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

Each repo calls shared workflows defined in this repository. The available
workflows are:

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

For Python+C repos, required system libraries must be installed in the calling
workflow before invoking the shared workflow.


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


## Test coverage

Coverage is monitored by CodeCov. The `CODECOV_TOKEN` secret must be set in
each repo's GitHub Actions secrets. Verify uploads are working by checking the
Actions logs after a CI run.

Flags used: **python-tests**, **c-python** (Python+C only), **C** (Python+C
only).

### `codecov.yml`

Each repo should include a `codecov.yml` at the repository root. This file
controls how CodeCov interprets and reports coverage data.

## Releases

Python releases use the `wheels.yml` workflow via the Trusted Publisher PyPI mechanism.
Because Trusted Publisher requires the upload to originate from the source repository,
this workflow is not shared — each repo maintains its own copy.

Python+C repos additionally use the shared `build-wheels.yml` workflow, which uses
`cibuildwheel` to build binary wheels across Linux, macOS, and Windows. All
configuration lives in `pyproject.toml` under `[tool.cibuildwheel]`.

tskit and kastore also have a repo-specific `release-c.yml` workflow solely for C API
releases (see below). This is separate from the Python release process.

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
  `MAJOR.MINOR.PATCH.dev1`.
