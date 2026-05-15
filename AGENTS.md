# Repository Guidelines

## Repository Scope

This is a workflow-only repository. There is no Python package, application
source tree, or local test suite here. The important behavior lives in
`.github/workflows/`, where GitHub Actions jobs clone scikit-learn, create a
temporary environment, install backend dependencies, run tests, and clean up.

## Project Structure

- `.github/workflows/dpnp32.yml`: runs dpnp tests on a float32 Intel GPU runner.
- `.github/workflows/dpnp64.yml`: runs dpnp tests on a float64 Intel GPU runner.
- `.github/workflows/torch-xpu.yml`: runs PyTorch XPU tests on a float64 runner.
- `README.md`: describes workflow purpose, inputs, and runner labels.
- `runners-setup.md`: records self-hosted Intel GPU runner setup notes.

## Development Commands

There is no local build. Useful checks are:

- `git diff --check`: catch whitespace and patch formatting issues.
- `gh workflow run dpnp64.yml -f scikit_learn_ref=scikit-learn:main`: trigger a
  workflow manually, if you have runner access.

The actual test commands are inside the workflow YAML files, for example
`SCIPY_ARRAY_API=1 pytest sklearn -k dpnp` and
`SCIPY_ARRAY_API=1 pytest sklearn -k xpu`.

## Style Guidelines

Use two-space indentation in GitHub Actions YAML and fenced code blocks in
Markdown examples. CI shell steps should use `set -euo pipefail`. Keep workflow
names backend-specific and use explicit runner labels such as `intel-gpu`,
`float32-gpu`, and `float64-gpu`.

## Commit & Pull Request Guidelines

Use short, direct commit summaries, for example `improve runners doc` or
`tighten security`. Pull requests should explain the workflow behavior changed.
