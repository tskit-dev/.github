# Tskit‑dev Ecosystem Maintenance Guide

This document describes routine cross‑repository maintenance in the tskit‑dev organisation. Read it alongside the main tskit development guide, which explains day‑to‑day development practices and conventions: https://tskit.dev/tskit/docs/latest/development.html

That document is primarily for contributors to the ecosystem — whereas this one is for maintainers.

## Tskit Website and Content Repos

The website at [tskit.dev](https://tskit.dev/) is built and deployed via GitHub Actions in the [tskit-site](https://github.com/tskit-dev/tskit-site) repository. The build system pulls live data from several APIs to populate pages. The site build runs daily and also on any push to `main` in ecosystem repositories. Of particular interest is the [ecosystem status](https://tskit.dev/ecosystem-status/) page, which gives a live view of repository CI health, PR and commit activity, and release versions/timing. Web content is also sourced from the [tutorials](https://github.com/tskit-dev/tutorials) and [tskit-explore](https://github.com/tskit-dev/tskit-explore) repositories.

## Software Repositories

Core packages:

- [tskit](https://github.com/tskit-dev/tskit)
- [msprime](https://github.com/tskit-dev/msprime)
- [tsinfer](https://github.com/tskit-dev/tsinfer)
- [kastore](https://github.com/tskit-dev/kastore)
- [tszip](https://github.com/tskit-dev/tszip)
- [tsdate](https://github.com/tskit-dev/tsdate)
- [tscompare](https://github.com/tskit-dev/tscompare)
- [tsbrowse](https://github.com/tskit-dev/tsbrowse)
- [tstrait](https://github.com/tskit-dev/tstrait)
- [tsconvert](https://github.com/tskit-dev/tsconvert)
- [pyslim](https://github.com/tskit-dev/pyslim)
- [tskit-rust](https://github.com/tskit-dev/tskit-rust)

Most libraries are primarily Python projects (some include C/C++ components), built with standardised Python packaging and tested across macOS, Linux, and Windows using GitHub Actions. This `.github` repository contains shared policies and reusable workflow templates used across the organisation for linting, tests, and documentation builds.


## GitHub Actions

There are several common CI tasks maintained as reusable workflows in this repository. Downstream repositories reference them as reusable workflows using a line like `uses: tskit-dev/.github/.github/workflows/<file>@<ref>`.

- Linting: [.github/workflows/lint.yml](.github/workflows/lint.yml) runs `pre-commit` consistently across repos and installs `clang-format==6.0.1` if needed. Used from test workflows in repositories such as tskit and msprime.
- Documentation: [.github/workflows/docs-build-template.yml](.github/workflows/docs-build-template.yml) installs docs extras using `uv`, optionally runs extra setup and build commands, builds Sphinx docs, and triggers a `tskit-site` rebuild when pushing to `main`.
- Testing: Repositories maintain their own test workflows with OS/Python matrices, OS-specific system deps, and local extension builds before running `pytest`. See examples in [tskit tests.yml](https://github.com/tskit-dev/tskit/blob/main/.github/workflows/tests.yml) and [msprime tests.yml](https://github.com/tskit-dev/msprime/blob/main/.github/workflows/tests.yml).
- Wheels and releases: Repositories build and test wheels per platform, then publish via PyPI using OIDC. See “Wheel building and publishing” below.
- Merge queue: PRs use GitHub Merge Queue via branch protection, PRs enter a queue after required checks pass.  See GitHub's [docs](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue).



## Common Tasks

### Releases

Release preparation follows the approach documented for tskit (see https://tskit.dev/tskit/docs/latest/development.html#releasing-a-new-version). Before tagging, review the changelog and ensure it includes only user‑visible changes: public API additions/removals, behavior changes, deprecations, and significant bug fixes. Internal refactoring, CI plumbing, and formatting changes should not appear in release notes.

Git tags trigger the wheel build and test workflows. Python packages use tags of the form `vX.Y.Z`. In tskit there is also a separate C API release flow. The release workflow drafts a GitHub Release with the appropriate changelog content and build artifacts. On tag push, CI builds the sdist and wheels for each supported platform and runs import/smoke tests against the built artifacts. When the release notes are correct, publish the Release on GitHub. If a repository uploads to PyPI on “release published”, the action handles the upload; otherwise, build locally with `python -m build` and upload with `twine`.

After a release, update dependent repositories where necessary. For example, if the C API version changes in tskit, dependent projects such as msprime, tsinfer, pyslim, and the Rust bindings need to be checked and bumped as required. Conda‑forge feedstocks are maintained for these packages (for example, tskit’s feedstock: https://github.com/conda-forge/tskit-feedstock); the conda-forge bot usually opens PRs after a PyPI release. Verify the version, checksums and dependencies, then merge the feedstock PR once CI is green.

When doing a wave of releases, for example to update python versions, it is best to go in the order of dependencies: first kastore, then tskit, then msprime, then the rest.

### Upgrading Python Versions

We aim to keep supported Python versions consistent across projects. Generally we wait for Numba to support a new Python release before adding it. Deprecate EOL Python versions following the official schedule (see https://devguide.python.org/versions/ and https://endoflife.date/python).

When updating a project:
- Bump the `requires-python` field and Python classifiers in `pyproject.toml`.
- Adjust CI matrices in test and wheel workflows to add/remove Python versions.
- For repositories that build Linux wheels via custom Docker scripts, update any central version lists (for example, the `PYTHON_VERSIONS` array in `tskit/docker/shared.env`).
- Update development docs where a specific Python version is mentioned.
- Update branch protection rules (required checks are often Python‑version specific) to match the new CI matrix.

Deprecations and added support should be communicated in the release notes.

### Python Dependencies

Extras for development, testing, and documentation (specified in `pyproject.toml`) are typically tightly pinned to provide CI stability and should be refreshed periodically. For an example layout of extras, see tskit’s `pyproject.toml`: https://github.com/tskit-dev/tskit/blob/main/pyproject.toml

Runtime dependencies should not be constrained with strict upper bounds unless there is a known incompatibility; prefer blocking a single problematic version when needed. Specify minimum versions that reflect what the code actually requires. When raising a minimum dependency version, record the change in the release notes.


### Updating Actions Versions

GitHub Actions used in workflows should be updated periodically to pick up bug fixes and new features. For reusable workflows in this repo, bump versions here first; then move the tag so downstream repos are updated.

### Pre‑commit Configuration

Repositories use pre‑commit for linting and formatting, and the configuration is intentionally similar across projects. Periodically update hooks with `pre-commit autoupdate`, then run `pre-commit run --all-files` locally to refresh generated changes. Commit the hook updates and any resulting formatting changes together in a single PR. We standardise on `clang-format` 6.0 for cross‑platform compatibility.


## Tskit Ecosystem Meetings

These are usually held every two weeks at 16:30 UK time at the Jitsi link listed on the tskit-dev Slack.

Use Slack scheduled reminders to send a 24‑hour and 1‑hour reminder before the meeting. To record the meeting, we usually use YouTube’s live streaming: create a private, unlisted stream (Create → Go live), then copy the stream key into Jitsi’s “Start live stream” option. After the meeting ends, the recording will be available on YouTube as an unlisted video for sharing. When posting in Slack, remind attendees to check with the presenter before sharing more widely.
