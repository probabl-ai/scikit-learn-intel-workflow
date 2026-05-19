# Self-hosted runner setup

This repository currently uses two self-hosted GitHub Actions runners. Both are
Linux x86_64 laptops with Intel GPUs, but they target different test jobs:

- `float64-gpu`: runner with an Intel GPU that supports float64 operations. This
  runner is used by the `array-api-intel` workflow for `dpnp` and `torch-xpu`
  jobs.
- `float32-gpu`: runner with an Intel GPU that supports float32 operations. This
  runner is used by the `array-api-intel` workflow for `dpnp` jobs and is not
  currently compatible with `torch-xpu` jobs.

Both runners also need the common labels used by the workflow:
`self-hosted`, `Linux`, `X64`, and `intel-gpu`.

For the dedicated Unix user, GitHub Actions runner service, and systemd
hardening setup, see [runner-isolation-setup.md](runner-isolation-setup.md).
This file only records the repository-specific runner inventory and host
runtime dependencies.

## Manual system setup

The workflow jobs install their Python dependencies from the repository Pixi
lockfile. But some lower-level system dependencies were installed manually on
the runner machines beforehand:

- Intel GPU drivers, following the Intel client GPU installation instructions
  for Ubuntu:
  https://dgpu-docs.intel.com/driver/client/overview.html#ubuntu-latest
- Intel oneAPI OpenCL runtime, so that `dpnp` and `dpctl` can discover the
  available CPU and GPU SYCL/OpenCL devices. The CPU OpenCL runtime is required
  for the CPU side of the `dpnp` test parametrization; otherwise those tests are
  skipped even when the Intel GPU runtime works.
- Pixi, available to the GitHub Actions runner service. The workflow currently
  require `pixi 0.68.1`.

If a workflow cannot see the expected GPU, first check the runner service user's
GPU groups and the systemd device allow-list documented in
[runner-isolation-setup.md](runner-isolation-setup.md).

## oneAPI OpenCL runtime installation

The Intel oneAPI APT repository was configured manually on each runner, then
`intel-oneapi-runtime-opencl` was installed from that repository.

```bash
# Remove a stale oneAPI source file if the runner was configured before.
sudo rm -f /etc/apt/sources.list.d/oneAPI.list

# Install the Intel package signing key in the system keyring directory.
wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null

# Add the Intel oneAPI APT repository using the dedicated keyring.
echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" \
  | sudo tee /etc/apt/sources.list.d/oneAPI.list

# Refresh package metadata and install the OpenCL runtime needed by dpnp/dpctl.
sudo apt update
sudo apt install intel-oneapi-runtime-opencl
```

## Validation checks

After changing runner setup, run a few checks directly on the runner machine
before triggering the GitHub Actions workflow:

```bash
# Confirm that the Intel GPU device nodes are present.
ls -l /dev/dri

# Confirm that OpenCL devices are visible, if clinfo is installed.
clinfo

# Confirm that both CPU and GPU OpenCL devices are visible.
clinfo | grep -E "Device Type|Device Name"
```

The workflow jobs also print runner, Python, OpenCL, `dpctl`, and `/dev/dri`
information near the start of each job. Those logs are the first place to check
when a runner is online but a backend cannot find the expected Intel device.
