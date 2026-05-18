# scikit-learn-intel-workflow

Run scikit-learn tests (in particular array API tests for `dpnp` and `torch` XPU) on Intel-hardware on a self-hosted runner.

## Workflows

The CI runners being self-hosted, we want to keep it isolated from the main scikit-learn CI
to limit security impacts: in particular it's not possible to trigger those workflows on usual
pull request but only via manual trigger by a limited list of authorized GitHub accounts.
We can grant access to trigger the CI to scikit-learn maintainers on request.

Each workflow mostly:
- clones the requested scikit-learn repository and ref
- builds scikit-learn from source in a fresh virtual environment
- installs the relevant array API backend
- and runs the matching subset of the scikit-learn test suite with `SCIPY_ARRAY_API=1`

Available workflows:
- `torch-xpu` runs the PyTorch XPU array API tests on a self-hosted machine with a
  [compatible GPU](https://docs.pytorch.org/docs/2.12/notes/get_start_xpu.html#intel-client-gpu). This runner is labeled `float64-gpu`.
- `dpnp-float64` runs the `dpnp` array API tests (CPU and GPU)  on the same `float64-gpu` runner.
- `dpnp-float32` runs the `dpnp` array API tests (CPU and GPU) on a second machine with a GPU that does not support float64 operations,
  which allows to catch some edge cases. This runner is labeled `float32-gpu`.

All workflows accept an optional `scikit_learn_ref` input in `owner:branch` format, such as `scikit-learn:main` or `cakedev0:doc/dpnp_xpu_support`. The workflow clones `https://github.com/<owner>/scikit-learn.git` and fetches the requested branch. By default, workflows test `scikit-learn:main`.


## Dependency locking

Common build/test dependencies, `dpnp`, and PyTorch XPU are managed by Pixi in
[`pixi.toml`](pixi.toml) and locked in [`pixi.lock`](pixi.lock).

The manifest intentionally does not use `exclude-newer` yet. Attempts to apply a 7-day cooldown exposed
incompatibilities with the PyTorch XPU index metadata and package publication times
(see [pytorch/pytorch#179374 (comment)](https://github.com/pytorch/pytorch/issues/179374#issuecomment-4467404210)).

This means lockfile refreshes are not safe to automate for now. We'll refresh it manually only when scikit-learn
needs compatibility updates. And when we do, we'll wait for ~7 days before merging the update lockfile, and we'll check
no supply chain attacks happened during this time.

Refresh command is `pixi update --no-install`.


## CI Runners

The runner labeled `float64-gpu` is a dedicated laptop with an integrated Intel GPU that sits on [ogrisel](https://github.com/ogrisel) personal office desk for now.
On the longer term, we plan to move this runner to a dedicated server available with Probabl and/or Intel supporting the hosting fees.

The Runner labeled `float32-gpu` is [cakedev0](https://github.com/cakedev0) personal laptop. It's not always reachable and will likely not be maintained on the longer term.

Some runner setup was done manually outside these workflows, including Intel GPU drivers and OpenCL runtime installation.
See [runners-setup.md](runners-setup.md) for the current runner setup notes.
