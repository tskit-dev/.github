# Repo administration

This document describes the standard layout and organisation principles for
core tskit-dev projects. The GitHub workflows defined in this repo require
that this layout is followed.


## tskit-dev packages

The tskit dev managed packages can be split into two groups: Python only, and Python and C.
The rules for administration are slightly different for the two. The repos are:

**Python only**

- tszip
- tstrait
- tsdate
- tsbrowse
- pyslim

**Python and C**

- tskit
- msprime
- kastore
- tsinfer


## Repository layout

### Python-only repos

The `pyproject.toml` lives at the repository root. Tests live in a `tests/`
directory at the root. The `prek.toml` lives at the root.

### Python and C repos

The Python package and its `pyproject.toml` live in a `python/` subdirectory.
The C library lives in a `c/` subdirectory. Tests are in `python/tests/`.
The `prek.toml` lives at the repo root (not in `python/`). All shared workflows
that accept a `pyproject-directory` input should be passed `python` for these
repos.

Documentation (`docs/`) lives at the repository root and includes a
`doxygen/` subdirectory for generating C API docs.


## Dependency management

We use uv exclusively for Python package management. All Python dependencies must
be declared explicitly in `pyproject.toml` using dependency groups. The dependencies
listed should be minimal, and pinned only where necessary.

### Required dependency groups

Every repo must define the following groups in `pyproject.toml`:

- **test** – packages needed to run the test suite (pytest, pytest-cov,
  pytest-xdist, and any domain-specific dependencies)
- **docs** – packages needed to build documentation (jupyter-book and
  Sphinx extensions)
- **lint** – linting tools; must include pinned versions of `prek` and `ruff`.
  Python+C repos must also include a pinned version of `clang-format`.
- **packaging** – `twine` and `validate-pyproject[all]`

Python+C repos additionally require:

- **wheels** – `cibuildwheel` (used by the build-wheels shared workflow)

A **dev** group that includes all other groups is recommended for local
development convenience.

### Version pinning in the lint group

Lint tools must be pinned to exact versions in the `lint` group to ensure
deterministic results across local development and CI. Example:

```toml
lint = [
    "clang-format==21.1.8",  # Python+C repos only
    "ruff==0.15.1",
    "prek==0.3.3",
]
```

### Lock files

All repos must maintain a `uv.lock` file at the root (or at the `pyproject.toml`
location for Python+C repos). This file is committed to the repository.


## Releases

### Python packages

Releases are performed using the ``wheels.yml`` workflow in each repo using the
Trusted Publisher PyPI mechanism. Note that this mechanism requires that the
actual upload be performed from the source repository and not from a shared
workflow. There is therefore some duplication across the repos of this logic.

To make a release first test that the process is working by pushing to
a branch named ``test-publish``:

```
git fetch upstream
git checkout upstream/main
git checkout -b test-publish
git push upstream test-publish
```

Then go to the Actions tab on github and watch the progress of the build. If
it all succeeds a release should be pushed to testPyPI (click on the
"Publish Distribition to TestPyPI" section to get the URL at the end). If this looks OK, then
proceed with the release on the GitHub UI.

This action is only run on demand and not on push to main branch because
the binary wheel building is complex and expensive and there's little point in
running it over and over again. If it's broken when you get ready to do a
release, fix it.

For Python-only repos the workflow is very simple and does nothing that the
python-packaging tests don't do.

### Binary wheel building

This is done using the shared workflow `build-wheels.yml`. This uses
cibuildwheel to do the heavy lifting. All configuration should be done
on the specific repo side using `pyproject.toml` under `[tool.cibuildwheel]`.

The workflow builds an sdist first, then builds binary wheels from that sdist
across a matrix of operating systems: ubuntu-24.04, windows-2025,
macos-15-intel, and macos-14. On macOS, only wheels native to the runner's
CPU architecture are built (controlled by `CIBW_ARCHS_MACOS`); this is
required because of how linking to GSL works with msprime.

All cibuildwheel configuration belongs in `pyproject.toml`. This includes
specifying which Python versions to build for, architecture constraints,
system dependency installation (e.g. `before-all` for GSL), and
post-build wheel repair (e.g. `delvewheel` on Windows).

### C API releases

Repos that contain a C library (tskit, kastore) have a separate `release.yml`
workflow for publishing C API releases. The release is triggered by pushing a
tag whose name contains `C_` (e.g. `C_1.1.3`). Tags that do not contain `C_`
trigger a Python release instead.

The workflow uses `meson dist` to build a source tarball from the `c/`
directory and then creates a draft GitHub release with that tarball attached.
The draft must be manually reviewed and published.

To make a C API release:

1. Tag the commit with a name containing `C_`: `git tag C_X.Y.Z && git push upstream C_X.Y.Z`
2. Go to the Actions tab and verify the release workflow succeeds.
3. Go to the Releases page and publish the draft release.


## Test coverage

Test coverage is monitored by CodeCov. When tests are run in CI, coverage is computed
and uploaded to CodeCov for each PR (and on merge). This is done using the
``CODECOV_TOKEN`` secret, which is generated by CodeCov and associated with a
given repo by GitHub. The token must be generated by logging in to the CodeCov UI,
copying the token and pasting it into the GitHub UI. This is in the settings/secrets
and variables/Actions section (repository secrets). It is important to check that
uploads are working correctly - do this by examining the GitHub actions logs and looking
at the output of the CodeCov upload command.

Coverage is uploaded with the following flags:

- **python-tests** – from the standard Python test suite
- **c-python** – from the Python-C interface tests (Python+C repos only)
- **C** – from the C unit tests (Python+C repos only)


## Standard workflows

### Docs

All documentation is built using jupyterbook and is defined in the top-level `docs/`
directory. The ``docs.yml`` workflow builds documentation and incorporates it
into the tskit.dev website. Two versions of the documentation are maintained,
"latest" and "stable". The latest version is updated from the repo each time
there is a merge. The stable version is updated on release.

The workflow requires:

- A `docs` dependency group in `pyproject.toml`
- A `__PKG_VERSION__` placeholder string in `docs/_config.yml`, which is
  replaced at build time with the package version derived from the installed package.

For Python+C repos, pass `doxygen-directory: docs/doxygen` to generate C API
documentation with Doxygen before the JupyterBook build.

On merge to `main`, the workflow triggers a rebuild of the tskit-site via
the AdminBot-tskit token.


### Lint

Linting is performed by prek using the ``lint.yml`` shared workflow. The workflow
installs only the `lint` dependency group (not the package itself, so no system
dependencies are required) and runs:

```
prek -c prek.toml run --all-files --show-diff-on-failure
```

The `prek.toml` file at the repo root defines all linting rules. The design
principle is to use only `builtin` and `local` hooks (no remote hook repos) to
ensure long-term stability and determinism. The standard hooks are:

**Builtin hooks (all repos):** `check-added-large-files`, `check-merge-conflict`,
`mixed-line-ending`, `check-case-conflict`, `check-yaml`, `check-toml`.

**Local hooks (all repos):** `ruff check --fix --force-exclude` and
`ruff format --force-exclude` applied to Python files.

**Local hooks (Python+C repos only):** `clang-format -i` applied to C files.
`clang-format` is installed as part of the `lint` dependency group and invoked
as a system command.

The `ruff` configuration (rules, target version, line length) is defined in
`pyproject.toml` under `[tool.ruff]`.

prek is intended to be run as a pre-commit hook. Run ``uv run prek install``
on the first time you use it. If you hit an obscure problem with gitlab
URLs, run ``uv run prek install -c prek.toml``.

If you are having problems with prek locally giving different results to
what's seen on CI, try running ``uv run prek cache clean``.


### Python packaging tests

The ``python-packaging.yml`` shared workflow validates that the package can be
correctly built and distributed. It runs on ubuntu-24.04 and:

1. Validates `pyproject.toml` using `validate-pyproject`
2. Builds sdist and wheel artifacts with `uv build`
3. Checks the artifacts with `twine check --strict`
4. Installs the built wheel into a fresh venv and optionally runs a CLI smoke test

To run a CLI smoke test, pass the `cli-test-cmd` input (e.g. `tszip --help`).
This verifies the entry point is correctly declared.


### Python tests

The ``python-tests.yml`` shared workflow runs the pytest suite on a configurable
Python version and OS. It installs only the `test` dependency group, runs pytest
with branch coverage and xdist parallelism (`-n2`), and uploads coverage to
CodeCov with the `python-tests` flag.

Repos should test against a minimum and a recent Python version (currently 3.11
and 3.13 or 3.14). Tests should run on ubuntu-24.04, macOS, and Windows.

For Python+C repos, system dependencies (e.g. GSL) must be installed separately
in the calling workflow before invoking the shared workflow.


### Python C tests

If the project defines a Python C extension module, use the ``python-c-tests.yml``
shared workflow to run focused low-level tests against the C bindings. This
workflow runs on ubuntu-24.04 only and:

1. Builds the C extension in-place with coverage instrumentation:
   `CFLAGS=--coverage uv run python setup.py build_ext --inplace`
2. Runs the designated low-level test file (default: `tests/test_python_c.py`;
   configurable via the `tests` input)
3. Generates a coverage report with gcovr and uploads it to CodeCov with the
   `c-python` flag

Pass `additional-apt-packages` for any system libraries required (e.g. `libgsl-dev`).


### C tests

If the project has a C library, use the ``c-tests.yml`` shared workflow to run
the C unit tests. The workflow runs on ubuntu-24.04 and requires the `library-directory`
input pointing to the C library root (e.g. `c`). It:

1. Installs system dependencies: `libcunit1-dev`, `libconfig-dev`, `ninja-build`,
   `valgrind`, `clang`, plus any extras via `additional-apt-packages`
2. Installs meson and gcovr via uv tool
3. Builds with gcc and coverage enabled, runs tests, generates a coverage report,
   and uploads it to CodeCov with the `C` flag
4. Builds again with clang and runs tests (no coverage upload)
5. Builds again with default settings and runs tests under valgrind with
   `--leak-check=full`
