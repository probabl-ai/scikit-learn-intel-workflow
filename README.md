# scikit-learn-intel-workflow

Run scikit-learn tests (in particular array API tests) on Intel-hardware on a self-hosted runner.

Latest run information available at:

https://github.com/probabl-ai/scikit-learn-intel-workflow/actions

## Workflows

The workflows in this repository are manually triggered from the GitHub Actions UI. Each workflow clones the requested scikit-learn repository and ref, builds scikit-learn from source in a fresh virtual environment, installs the relevant array API backend, runs the matching subset of the scikit-learn test suite with `SCIPY_ARRAY_API=1`, and cleans up the runner temporary directory when the job finishes.

Available workflows:

- `dpnp-float32` runs the `dpnp` array API tests on a self-hosted Intel GPU runner labeled `float32-gpu`.
- `dpnp-float64` runs the `dpnp` array API tests on a self-hosted Intel GPU runner labeled `float64-gpu`.
- `torch-xpu` runs the PyTorch XPU array API tests on a self-hosted Intel GPU runner labeled `float64-gpu`.
   It can install either stable or nightly PyTorch XPU wheels.

All workflows accept an optional `scikit_learn_ref` input in `owner:branch` format, such as `scikit-learn:main` or `cakedev0:doc/dpnp_xpu_support`. The workflow clones `https://github.com/<owner>/scikit-learn.git` and fetches the requested branch. By default, workflows test `scikit-learn:main`.

Note that some runner setup was done manually outside these workflows, including Intel GPU drivers and OpenCL runtime installation.
See [runners-setup.md](runners-setup.md) for the current runner setup notes.

## CI Runners

The CI runners being self-hosted, we want to keep it isolated from the main scikit-learn CI
to limit security impacts: in particular it's not possible to trigger it on usual pull request
but only via manual trigger by a limited list of authorized GitHub accounts.

We can grant access to trigger the CI to scikit-learn maintainers on request.

**More details:**
- Runner labeled `float64-gpu`: This CI runner is a dedicated laptop with an integrated Intel GPU that sits on [ogrisel](https://github.com/ogrisel) personal office desk for now. On the longer term, we plan to move this runner to a dedicated server available with Probabl and/or Intel supporting the hosting fees.
- Runner labeled `float32-gpu`: This CI runner is [cakedev0](https://github.com/cakedev0) personal laptop. It's not always reachable and will likely not be maintained on the longer term.
