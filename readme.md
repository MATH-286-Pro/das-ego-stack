# das-ego-stack
DAS Ego Local Data Acquisition and Processing Stack

This repo ships:
- `delivery_pipeline-1.0.1-py3-none-any.whl` — the **host-side** orchestrator
- `sample_input/` — one complete `master + sub_left + sub_right` group, ready to smoke-test the pipeline end-to-end

Default flow documented below is **variant C**
(`qc → merge → vio → vio_check → pose6d → pose6d_check`, no convert).
Variants A and B are one CLI flag away — see [§5.2](#52-running-variant-a--b).


## 1. Prerequisites

### 1.1 Common (all variants)

- Linux x86_64 (Ubuntu 20.04 / 22.04 or Debian 11 / 12), CPU ≥ 16C, RAM ≥ 32G **recommended**
- Python ≥ 3.9 with `pip` and `venv`
- Docker CLI on PATH; check disk space on the Docker root:
  `docker info | grep "Docker Root Dir"` → `df -h <that path>`
- AWS CLI v2 with credentials that can read ECR
  (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`)

### 1.2 GPU (required for variant C and B — `pose6d` step)

`pose6d` runs `model.to(cuda)` inside its algo container, so the host
needs:

1. NVIDIA driver installed (`nvidia-smi` runs successfully)
2. NVIDIA Container Toolkit installed and registered with Docker

```bash
# 1) add nvidia-container-toolkit apt repo
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 2) install
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 3) register the nvidia runtime with docker
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 4) verify
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

The final command must print the GPU table. If it fails with
`Failed to initialize NVML: Driver/library version mismatch`, the
in-kernel driver disagrees with the userland library — reboot, or
unload + reload the nvidia kernel modules. Variant C / B will not
start until this passes.

Variant A is CPU-only — skip §1.2 if that's all you need.


## 2. Install the wheel

```bash
python3 -m venv ~/.venv/delivery
~/.venv/delivery/bin/pip install delivery_pipeline-1.0.1-py3-none-any.whl
```

The wheel installs a single CLI: `delivery-pipeline`. It is the **host
orchestrator only** — every algo step still runs in its own container.


## 3. Pull the images (AWS ECR)

The algo images live on Amazon ECR. You need an AWS account with read
permission on the registry plus AWS CLI v2 installed and configured.

```bash
# --- Install AWS CLI v2 (Linux x86_64) ---
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install          # requires sudo; --update if already installed
aws --version
aws configure               # set access key / secret / region (us-east-1)
```

> **Heads-up on `sudo`**: AWS and docker credentials are per-user
> (`~/.aws/`, `~/.docker/`). Run every step below — and
> `delivery-pipeline` itself — as the **same user**; mixing sudo and
> non-sudo leads to `Unable to locate credentials` / `no basic auth
> credentials`.

```bash
# Authenticate docker against ECR (token expires in 12 h).
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
      764042516397.dkr.ecr.us-east-1.amazonaws.com
```

Pre-pull every image variant C needs (optional but recommended — fail
fast on network / ECR / disk issues):

```bash
REG=764042516397.dkr.ecr.us-east-1.amazonaws.com/genimage
VER=v1.0.1
for step in qc merge vio vio_check pose6d pose6d_check; do
    docker pull ${REG}:${step}-${VER}
done
```

> Variant A only needs `qc merge vio vio_check`. Variant B adds
> `convert` on top of variant C.


## 4. Lay out inputs

This repo ships a working sample under `sample_input/`:

```
sample_input/
├── DAS-Ego_20260507180344_master_center_814084_59214a9e.mcap
├── DAS-Finger_20260507180344_sub_left_81709d_59214a9e.mcap
└── DAS-Finger_20260507180344_sub_right_3e6d0c_59214a9e.mcap
```

The three files share the trailing 8-hex `<uuid>` (`59214a9e`) — that's
how the orchestrator groups them. For your own data, drop every `.mcap`
into a single host directory; each complete group needs:

- one `DAS-Ego_..._master_<uuid>.mcap`
- one `DAS-Finger_..._sub_left_<uuid>.mcap`
- one `DAS-Finger_..._sub_right_<uuid>.mcap`

Incomplete groups are skipped (logged as `SKIP` with the missing roles).

Permissions matter — the algo containers run as a non-root uid. Open up
read/write on both sides before the run:

```bash
chmod -R a+rX  "$INPUT_DIR"      # readable for everyone (dirs + files)
chmod -R a+rwX "$OUTPUT_DIR"     # pipeline needs to write here
```

`a+X` (uppercase) only sets the execute bit on directories and
already-executable files, so it won't mark every `.mcap` as executable.


## 5. Run — variant C (default for this stack)

```bash
# Absolute paths only — passed through to algo containers verbatim.
cd /path/to/das-ego-stack
export INPUT_DIR=$(realpath ./sample_input)
export OUTPUT_DIR=$(realpath -m ./output)
mkdir -p "$OUTPUT_DIR"
chmod -R a+rX  "$INPUT_DIR"
chmod -R a+rwX "$OUTPUT_DIR"

~/.venv/delivery/bin/delivery-pipeline --variant c \
    --ecr-registry 764042516397.dkr.ecr.us-east-1.amazonaws.com/genimage \
    --image-version v1.0.1 \
    --input-dir "$INPUT_DIR" --output-dir "$OUTPUT_DIR" \
    --continue-on-error
```

A successful run ends with:

```
summary: 1/1 groups succeeded
```

### 5.1 Override images (test locally built `:compile-dev` instead of ECR)

To smoke-test against images you built yourself (e.g.
`delivery_pose6d:compile-dev` from `dockerfiles/jobs/delivery/build_and_push_to_ecr.sh --skip-push`),
set the per-step `ALGO_*_IMAGE` env vars — they take precedence over
`--variant`:

```bash
export ALGO_QC_IMAGE=delivery_qc:compile-dev
export ALGO_MERGE_IMAGE=delivery_merge:compile-dev
export ALGO_VIO_IMAGE=delivery_vio:compile-dev
export ALGO_VIO_CHECK_IMAGE=delivery_vio_check:compile-dev
export ALGO_POSE6D_IMAGE=delivery_pose6d:compile-dev
export ALGO_POSE6D_CHECK_IMAGE=delivery_pose6d_check:compile-dev

~/.venv/delivery/bin/delivery-pipeline --variant c \
    --input-dir "$INPUT_DIR" --output-dir "$OUTPUT_DIR" \
    --continue-on-error
```

The CLI still needs `--variant c` so it knows which step list to run;
the ECR registry / version values aren't used when every image is
pinned via env.

### 5.2 Running variant A / B

Same command, swap `--variant`:

| `--variant` | Steps | GPU | Final artifact at group root |
|---|---|---|---|
| `vio` (A) | qc → merge → vio → vio_check | no | `merged_output_ego_vio.mcap` |
| `pose6d` (B) | …→ pose6d → pose6d_check → convert | yes | `delivery.mcap` |
| `c` (this stack's default) | …→ pose6d → pose6d_check | yes | `merged_output_pose6d.mcap` |

Pre-pull list differs accordingly (variant A: 4 images, B: 7).

### 5.3 Common flags

| Flag | Default | Meaning |
|---|---|---|
| `--variant` | `vio` | One of `vio` / `pose6d` / `c` — selects the step list |
| `--steps` | from `--variant` | Override the step list (comma-separated subset) |
| `--ecr-registry` | `…/delivery` | `<registry>/<repo>` prefix; combined with `--image-version` to form `<prefix>:<step>-<version>` |
| `--image-version` | `v1.0.1` | Tag suffix on each ECR image |
| `--continue-on-error` | off | Keep going to the next group if one fails |


## 6. Output

Variant C output:

```
$OUTPUT_DIR/<group-uuid>/
├── result.json                        # per-step status — check this first on failure
├── merged_output_pose6d.mcap          # pose6d output (upstream topics back-filled)
└── debug/                             # intermediates; safe to delete after audit
    └── work/
        ├── step_outputs/<step>.json   # per-step stdout/result
        └── qc/ merge/ vio/ vio_check/ pose6d/ pose6d_check/
                                       # no empty convert/ dir
```

A top-level summary is logged at the end of the run:

```
summary: 3/4 groups succeeded
  FAIL ab12cd34 at pose6d
```

Variant A drops `merged_output_ego_vio.mcap` at the group root and
keeps only the four work dirs (`qc/ merge/ vio/ vio_check/`). Variant B
drops `delivery.mcap` and keeps all seven work dirs including
`convert/`.


## 7. Troubleshooting

### `permission denied while trying to connect to the Docker daemon socket`

You aren't in the `docker` group. Fix:

```bash
sudo usermod -aG docker "$USER"
# log out and back in, then verify:
groups | grep docker
```

### `could not select device driver "" with capabilities: [[gpu]]` (variant C / B)

NVIDIA Container Toolkit isn't installed or Docker wasn't restarted
after `nvidia-ctk runtime configure`. Redo
[§1.2](#12-gpu-required-for-variant-c-and-b--pose6d-step) and re-run the
`nvidia-smi` verification.

### `Failed to initialize NVML: Driver/library version mismatch` (variant C / B)

In-kernel driver disagrees with the userland library — usually after a
driver upgrade without a reboot. Reboot the host, or unload + reload
the nvidia kernel modules (`sudo rmmod nvidia_uvm nvidia_drm nvidia_modeset nvidia && sudo modprobe nvidia`)
and re-run `nvidia-smi`.

### ECR `denied: User: arn:aws:iam::... is not authorized to perform: ecr:GetAuthorizationToken`

The IAM principal doesn't have ECR read permission. Ask your admin to
attach `AmazonEC2ContainerRegistryReadOnly` (or an equivalent custom
policy) to your user / role.

### Algo container exits with code 1 but step JSON is empty / zero stderr

Most often a permission problem with `userns-remap` docker daemons
(`docker info | grep "Docker Root Dir"` → path ends in something like
`/165536.165536/`). The algo container's uid 0 maps to host uid 165536,
which can't read your root-owned `0644` mcap files. Re-run the `chmod`
commands from [§4](#4-lay-out-inputs):

```bash
chmod -R a+rX  "$INPUT_DIR"
chmod -R a+rwX "$OUTPUT_DIR"
```

### Input paths must be absolute and identical on both sides

`delivery-pipeline` shells out to `docker run` for every algo step and
passes the host path through unchanged. Always use absolute paths for
`--input-dir` / `--output-dir`. (When you run the orchestrator on the
host directly — as this doc describes — there is no second namespace,
so this is automatic.)

### Per-step audit

When a group fails:

```bash
# top-level status across all steps
cat $OUTPUT_DIR/<group-uuid>/result.json

# the specific step's stdout/error
cat $OUTPUT_DIR/<group-uuid>/debug/work/step_outputs/<step>.json
```
