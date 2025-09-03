# .github
Defaults for github shared across tskit-dev repositories

### Documentation Build Template

**File:** `.github/workflows/docs-build-template.yml`

A reusable workflow for building documentation across all TSKit ecosystem repositories. This eliminates duplication of nearly identical documentation workflows.

**Usage:**
```yaml
jobs:
  docs:
    uses: tskit-dev/.github/.github/workflows/docs-build-template.yml@main
    with:
      requirements-path: requirements/CI-docs/requirements.txt  # optional
      additional-setup: sudo apt-get install -y libgsl0-dev    # optional  
      make-command: make -C python                             # optional
```
**Parameters:**
- `requirements-path`: Path to documentation requirements file (default: `requirements/CI-docs/requirements.txt`)
- `additional-setup`: Additional setup commands (system packages, special builds, etc.)
- `make-command`: Make command for building C modules before docs
