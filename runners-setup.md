Currently two runners are set-up in this repo. Both are laptops, one has
a GPU with float64 support, and compatible with torch XPU. The other one has 
a GPU with float32 support and not compatbile with torch XPU.

Both have been set-up with intel drivers, following the instructions here:
https://dgpu-docs.intel.com/driver/client/overview.html#ubuntu-latest

In addition to that, to have `dpnp`/`dpctl` detects CPUs, `intel-oneapi-runtime-opencl` was installed using:

```bash
wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" \
  | sudo tee /etc/apt/sources.list.d/oneAPI.list

sudo apt update
sudo apt install intel-oneapi-runtime-opencl
```

