# TPU Setup

## Environment variables

Paste these into any new terminal before running the commands below:

```bash
export TPU_NAME=boris
export PROJECT_ID=tpu-2026
export ZONE=us-east5-a
export ACCELERATOR_TYPE=v6e-4
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
pip install jupyterlab    # one-time
jupyter lab --no-browser --port=8888 --ip=127.0.0.1
```

Open the printed `http://127.0.0.1:8888/lab?token=...` URL in your laptop's browser.

## Delete when done

```bash
gcloud compute tpus tpu-vm delete $TPU_NAME --project=$PROJECT_ID --zone=$ZONE
```
