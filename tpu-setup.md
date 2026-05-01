# TPU Setup

## Environment variables

Paste these into any new terminal before running the commands below:

```bash
export TPU_NAME=boris
export PROJECT_ID=tpu-2026
export ZONE=us-east5-a
export ACCELERATOR_TYPE=v6e-1
export VERSION=v2-alpha-tpuv6e
```

## One-time: enable Private Google Access on the subnet

```bash
gcloud compute networks subnets update default \
  --region=us-east5 \
  --enable-private-ip-google-access \
  --project=$PROJECT_ID
```

## Create the TPU VM

```bash
gcloud compute tpus tpu-vm create $TPU_NAME \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --accelerator-type=$ACCELERATOR_TYPE \
  --version=$VERSION \
  --internal-ips
```

Or run `./create_tpu_env.sh`.

## One-time: Cloud NAT (for outbound internet from VM)

Internal-IP VMs can't reach the public internet (e.g. `pip`, `curl claude.ai`) without NAT.

```bash
gcloud compute routers create nat-router --network=default --region=us-east5 --project=$PROJECT_ID
gcloud compute routers nats create nat-config --router=nat-router --region=us-east5 --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges --project=$PROJECT_ID
```

## One-time: IAP firewall rule

```bash
gcloud compute firewall-rules create allow-iap-ssh --project=$PROJECT_ID --network=default --source-ranges=35.235.240.0/20 --allow=tcp:22
```

## SSH (via IAP tunnel — alpha track)

```bash
gcloud alpha compute tpus tpu-vm ssh $TPU_NAME --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap
```

## Python environment on the TPU VM

The fastest path is `./bootstrap.sh` from this repo — it installs python3.12,
creates the venv, runs the install order below, pulls secrets from Secret
Manager, and wires up `~/.bashrc`. The rest of this section explains what
the script does and why; if you just want a working VM, run the script.

Ubuntu 22.04 on these TPU VMs ships with `python3.10` and `python3.11` only —
the tunix stack needs `python3.12`. The deadsnakes PPA is already configured,
so `apt` can install it directly.

```bash
sudo apt-get install -y python3.12 python3.12-venv python3.12-dev
python3.12 -m venv ~/venvs/tunix
source ~/venvs/tunix/bin/activate
pip install --upgrade pip setuptools wheel
```

Install the tunix / jax / flax stack. Order matters here:

1. PyPI batch first.
2. `jax` from git (the PyPI release lags behind what tunix expects).
3. `tunix` and `qwix` from git — `tunix` pulls `flax` from PyPI as a
   dependency, and downgrades `transformers` (→ 4.57) and `huggingface_hub`
   (→ 0.36). Both downgrades are expected.
4. Replace the PyPI `flax` with the GitHub version **after** tunix installs,
   otherwise the tunix install would overwrite it again.

```bash
pip install python-dotenv kagglehub ipywidgets tensorflow tensorflow_datasets \
            tensorboardX transformers grain huggingface_hub datasets 'numpy>2'
pip install git+https://github.com/jax-ml/jax
pip install git+https://github.com/google/tunix git+https://github.com/google/qwix
pip uninstall -y flax
pip install git+https://github.com/google/flax
```

Note: `python-dotenv` is the correct PyPI name for the `import dotenv` package.
The bare `dotenv` package on PyPI is a different, unmaintained project.

**`libtpu` is required for jax to see the TPU.** Installing `jax` from git does
*not* pull `libtpu` (the TPU runtime), so without it jax silently falls back
to CPU with this warning:
```
WARNING:jax._src.xla_bridge:A Google TPU may be present on this machine, but
either a TPU-enabled jaxlib or libtpu is not installed. Falling back to cpu.
```
`requirements.txt` pins `libtpu`, so `bootstrap.sh` handles this. To verify:
```python
import jax; print(jax.default_backend(), jax.devices())
# expect: tpu [TpuDevice(...), ...]
```

## Run a Jupyter notebook

Two terminals on your laptop.

**Terminal 1** — open a port-forwarding tunnel (stays running, no shell):
```bash
gcloud alpha compute tpus tpu-vm ssh $TPU_NAME --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap -- -L 8888:localhost:8888 -N
```

**Terminal 2** — SSH in normally and launch Jupyter on the TPU:
```bash
gcloud alpha compute tpus tpu-vm ssh $TPU_NAME --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap
# then on the TPU:
source ~/venvs/tunix/bin/activate
pip install jupyterlab    # one-time, into the venv so the kernel sees tunix/jax/flax
jupyter lab --no-browser --port=8888 --ip=127.0.0.1
```

Open the printed `http://127.0.0.1:8888/lab?token=...` URL in your laptop's browser.

## Delete when done

```bash
gcloud compute tpus tpu-vm delete $TPU_NAME --project=$PROJECT_ID --zone=$ZONE
```
