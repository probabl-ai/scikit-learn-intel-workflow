# Secure User for the Self-Hosted Runner

This repository runs manually triggered scikit-learn test workflows on
self-hosted Intel GPU machines. The workflows fetch the locked Pixi files, clone
a requested scikit-learn `owner:branch` into the runner temp directory, build
scikit-learn, run the selected tests, and clean the run-specific checkout.

Treat every run as code execution by anyone allowed to trigger the workflow and
choose `scikit_learn_ref`. Use a dedicated account and keep the runner isolated
from personal files, SSH keys, cloud credentials, and unrelated repositories.

## 1. Create the Runner User

```bash
sudo useradd \
  --system \
  --create-home \
  --home-dir /opt/gha-runner \
  --shell /usr/sbin/nologin \
  --user-group \
  gha-runner

sudo passwd --lock gha-runner

sudo chmod 0750 /opt/gha-runner
sudo install -d -o gha-runner -g gha-runner -m 0750 /opt/gha-runner/actions-runner
sudo install -d -o gha-runner -g gha-runner -m 0750 /opt/gha-runner/_work
sudo install -d -o gha-runner -g gha-runner -m 0750 /opt/gha-runner/_work/_temp
```

Do not add this user to `sudo`, `docker`, `lxd`, `adm`, or other privileged
groups.

## 2. Install Host Prerequisites

The host needs:

- the GitHub Actions runner for Linux x64;
- `git`, `wget`, `curl`, and a compiler toolchain;
- `pixi` on the runner service `PATH`, satisfying `requires-pixi` in
  [`pixi.toml`](pixi.toml);
- Intel GPU drivers and OpenCL / Level Zero runtime;
- Intel CPU OpenCL runtime for the CPU side of the `dpnp` tests;
- `clinfo` for diagnostics.

On Debian or Ubuntu:

```bash
sudo apt-get update
sudo apt-get install --no-install-recommends \
  ca-certificates \
  curl \
  wget \
  git \
  build-essential \
  clinfo
```

See [runners-setup.md](runners-setup.md) for the Intel driver, oneAPI OpenCL
runtime, and Pixi notes.

Install a compatible Pixi binary somewhere visible to systemd. This example
uses `pixi 0.68.1`, which satisfies the current manifest requirement:

```bash
curl -fsSL https://pixi.sh/install.sh \
  | sudo env PIXI_VERSION=v0.68.1 PIXI_BIN_DIR=/usr/local/bin PIXI_NO_PATH_UPDATE=1 bash
```

Validate:

```bash
sudo -u gha-runner -H env PATH=/usr/local/bin:/usr/bin:/bin pixi --version
```

Expected output should show a compatible Pixi version, for example:

```text
pixi 0.68.1
```

## 3. Grant GPU Access

Check device ownership:

```bash
ls -l /dev/dri
```

Add the narrow GPU group first:

```bash
sudo usermod -aG render gha-runner
```

Add `video` only if the installed GPU stack requires `/dev/dri/card*` access:

```bash
sudo usermod -aG video gha-runner
```

Restart the runner service after changing groups.

## 4. Configure the GitHub Runner

Download the latest Linux x64 runner from:

https://github.com/actions/runner/releases

Extract it as `gha-runner`:

```bash
cd /tmp
curl -L -o actions-runner-linux-x64.tar.gz \
  https://github.com/actions/runner/releases/download/vX.Y.Z/actions-runner-linux-x64-X.Y.Z.tar.gz

sudo -u gha-runner -H tar xzf actions-runner-linux-x64.tar.gz \
  -C /opt/gha-runner/actions-runner
```

Create a registration token from GitHub:

`Settings` -> `Actions` -> `Runners` -> `New self-hosted runner`

Configure the runner:

```bash
sudo -u gha-runner -H bash -lc '
  cd /opt/gha-runner/actions-runner
  ./config.sh \
    --url https://github.com/probabl-ai/scikit-learn-intel-workflow \
    --token REGISTRATION_TOKEN \
    --name RUNNER_NAME \
    --work /opt/gha-runner/_work \
    --labels intel-gpu,float64-gpu \
    --unattended
'
```

Use labels matching the machine:

- `intel-gpu,float64-gpu` for `dpnp-float64` and `torch-xpu`;
- `intel-gpu,float32-gpu` for `dpnp-float32`.

Avoid registering one machine with both `float32-gpu` and `float64-gpu` unless
you intentionally want both workflow classes to run there. The built-in
`self-hosted`, `Linux`, and `X64` labels are added automatically.

## 5. Install and Harden the Service

Install the official service:

```bash
sudo bash -lc 'cd /opt/gha-runner/actions-runner && ./svc.sh install gha-runner'
```

Find the unit name:

```bash
systemctl list-unit-files 'actions.runner.*.service'
```

Add a systemd drop-in:

```bash
sudo systemctl edit actions.runner.OWNER-REPO.RUNNER_NAME.service
```

Use:

```ini
[Service]
Environment=PATH=/usr/local/bin:/usr/bin:/bin
Environment=TMPDIR=/opt/gha-runner/_work/_temp
Environment=TMP=/opt/gha-runner/_work/_temp
Environment=TEMP=/opt/gha-runner/_work/_temp
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/gha-runner
DevicePolicy=closed
DeviceAllow=/dev/dri/renderD* rw
DeviceAllow=/dev/dri/card* rw
RestrictSUIDSGID=true
LockPersonality=true
MemoryDenyWriteExecute=false
```

The temp variables are needed because `ProtectSystem=strict` makes `/tmp`
read-only for the service, and Pixi needs a writable temp directory.

Start the service:

```bash
sudo systemctl daemon-reload
sudo bash -lc 'cd /opt/gha-runner/actions-runner && ./svc.sh start'
sudo bash -lc 'cd /opt/gha-runner/actions-runner && ./svc.sh status'
```

## 6. Validate

Before running workflows:

```bash
sudo -u gha-runner -H bash -lc 'whoami && groups'
sudo -u gha-runner -H bash -lc 'test ! -w /etc/passwd'
sudo -u gha-runner -H bash -lc 'command -v git && command -v wget'
sudo -u gha-runner -H bash -lc 'pixi --version'
sudo -u gha-runner -H bash -lc 'ls -l /dev/dri'
sudo -u gha-runner -H bash -lc 'clinfo | grep -E "Platform Name|Device Type|Device Name"'
```

Expected:

- `whoami` is `gha-runner`;
- `groups` has only the runner group and required GPU groups;
- `pixi --version` reports a version compatible with
  [`pixi.toml`](pixi.toml);
- `clinfo` sees the expected Intel GPU and CPU OpenCL devices.

If `clinfo` works with `sudo -u gha-runner -H` but the workflow only sees part
of the hardware, restart the service. If it still fails, inspect the systemd
device allow-list and replace the wildcard `DeviceAllow` entries with exact
paths from `ls -l /dev/dri`.

## 7. Operations

Keep `permissions: {}`, manual triggers, and the `refs/heads/main` guard unless
the runner is disposable. Keep workflow dispatch access limited to trusted
users. Patch the OS, drivers, Pixi, and the GitHub Actions runner regularly.

To clear old work directories:

```bash
sudo bash -lc 'cd /opt/gha-runner/actions-runner && ./svc.sh stop'
sudo find /opt/gha-runner/_work -mindepth 1 -maxdepth 1 -exec rm -rf {} +
sudo install -d -o gha-runner -g gha-runner -m 0750 /opt/gha-runner/_work/_temp
sudo bash -lc 'cd /opt/gha-runner/actions-runner && ./svc.sh start'
```
