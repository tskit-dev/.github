# .github
Defaults for github shared across tskit-dev repositories

### Documentation Build Template

**File:** `.github/workflows/docs-build-template.yml`

Reusable workflow for building documentation across TSKit ecosystem repositories. It installs the docs extras using `uv` from `pyproject.toml` (in the repo root or `python/`), builds Sphinx docs, and triggers a rebuild of `tskit-site` on pushes to `main`.

**Usage:**
```yaml
jobs:
  docs:
    uses: tskit-dev/.github/.github/workflows/docs-build-template.yml@main
    with:
      additional-setup: sudo apt-get install -y libgsl0-dev   # optional
      make-command: make -C python                            # optional
```
**Inputs:**
- `additional-setup` (string, optional): Extra commands to run before building docs (e.g., install system packages).
- `make-command` (string, optional): Command to build any required C modules before building docs.

### Reusable Lint Workflow

**File:** `.github/workflows/lint.yml`

Runs `pre-commit` consistently across repositories and installs `clang-format==6.0.1` if needed.

**Usage:**
```yaml
jobs:
  lint:
    uses: tskit-dev/.github/.github/workflows/lint.yml@main
```

### Reusable Python Tests

**File:** `.github/workflows/python-tests.yml`

Installs test extras using `uv`, optionally runs additional build steps, executes `pytest` with coverage, and uploads coverage to Codecov. Requires `python_version` and `os` inputs.

**Matrix example:**
```yaml
jobs:
  tests:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-24.04, macos-13]
        python: ["3.9", "3.11"]
    uses: tskit-dev/.github/.github/workflows/python-tests.yml@main
    with:
      os: ${{ matrix.os }}
      python_version: ${{ matrix.python }}
      additional_commands: |
        # optional: build extensions, set env, etc.
        echo "Additional setup"
      make_command: ""  # optional
```

Notes:
- Set the `CODECOV_TOKEN` repository secret if your repo requires a token for Codecov uploads.
- For long-term stability, consider pinning `@main` to a release tag or commit SHA.
