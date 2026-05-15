# scikit-learn-intel-workflow

Run scikit-learn tests (in particular array API tests) on Intel-hardware on a self-hosted runner.

Latest run information available at:

https://github.com/probabl-ai/scikit-learn-intel-workflow/actions

## Workflows

The workflows in this repository are manually triggered from the GitHub Actions UI. Each workflow clones the requested scikit-learn repository and ref, builds scikit-learn from source in a fresh virtual environment, installs the relevant array API backend, runs the matching subset of the scikit-learn test suite with `SCIPY_ARRAY_API=1`, and cleans up the runner temporary directory when the job finishes.

Available workflows:

- `dpnp-float32` runs the `dpnp` array API tests on a self-hosted Intel GPU runner labeled `float32-gpu`.
- `dpnp-float64` runs the `dpnp` array API tests on a self-hosted Intel GPU runner labeled `float64-gpu`.
- `torch-xpu` runs the PyTorch XPU array API tests on a self-hosted Intel GPU runner labeled `float64-gpu`. It can install either stable or nightly PyTorch XPU wheels.

All workflows accept optional inputs for the scikit-learn repository URL and the branch, tag, or commit to test. By default they test `main` from `https://github.com/scikit-learn/scikit-learn.git`.

Note that some runner setup was done manually outside these workflows, including Intel GPU drivers and OpenCL runtime installation.
See [runners-setup.md](runners-setup.md) for the current runner setup notes.
