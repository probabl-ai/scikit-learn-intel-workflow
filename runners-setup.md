# Self-hosted runner setup

This repository currently uses two self-hosted GitHub Actions runners. Both are
Linux x86_64 laptops with Intel GPUs, but they target different test jobs:

- `float64-gpu`: runner with an Intel GPU that supports float64 operations. This
  runner is used by the `dpnp-float64` and `torch-xpu` workflows.
- `float32-gpu`: runner with an Intel GPU that supports float32 operations. This
  runner is used by the `dpnp-float32` workflow and is not currently compatible
  with the `torch-xpu` workflow.

Both runners also need the common labels used by the workflows:
`self-hosted`, `Linux`, `X64`, and `intel-gpu`.

## Manual system setup

The workflows create their own Python virtual environments and install the
Python dependencies needed for each run. But some lower-level system dependencies
were installed manually on the runner machines beforehand:

- Intel GPU drivers, following the Intel client GPU installation instructions
  for Ubuntu:
  https://dgpu-docs.intel.com/driver/client/overview.html#ubuntu-latest
- Intel oneAPI OpenCL runtime, so that `dpnp` and `dpctl` can discover the
  available CPU and GPU SYCL/OpenCL devices.

The GitHub Actions runner service user must also be able to access the Intel GPU
device nodes under `/dev/dri`. If the workflows cannot see the GPU, check the
runner user's group membership and the permissions on `/dev/dri/render*`.

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
before triggering the GitHub Actions workflows:

```bash
# Confirm that the Intel GPU device nodes are present.
ls -l /dev/dri

# Confirm that OpenCL devices are visible, if clinfo is installed.
clinfo

# Confirm that dpctl can list devices from a clean Python environment.
python3 -m venv /tmp/dpctl-check
source /tmp/dpctl-check/bin/activate
python -m pip install --upgrade pip dpctl dpnp
python -c "import dpctl; print(dpctl.get_devices())"
python -c "import dpnp; print(dpnp.__version__)"
deactivate
rm -rf /tmp/dpctl-check
```

The workflows also print runner, Python, OpenCL, and `/dev/dri` information at
the start of each job. Those logs are the first place to check when a runner is
online but a backend cannot find the expected Intel device.
