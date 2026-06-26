# das-ego-stack
DAS Ego Local Data Acquisition and Processing Stack

## Update Log

**v1.0.4** (2026-06-26)
- Update VIO algorithm image, Resolve screen freezing issues and improve timeliness ratio
- Upgrade all Docker image versions to `v1.0.4` (public registry & AWS ECR)

**v1.0.3** (2026-06-05)
- Update VIO algorithm image, fix VIO hanging issue
- Upgrade `delivery_pipeline` wheel to `1.0.3`
- Upgrade all Docker image versions to `v1.0.3` (public registry & AWS ECR)

---

This repo ships:
- `delivery_pipeline-1.0.3-py3-none-any.whl` — the **host-side** orchestrator
- `sample_input/` — one complete `master + sub_left + sub_right` group, ready to smoke-test the pipeline end-to-end

Pipeline flow: `qc → merge → vio → vio_check`. CPU-only — no GPU
required.


## 1. Prerequisites

- Linux x86_64 (Ubuntu 20.04 / 22.04 or Debian 11 / 12), CPU ≥ 16C, RAM ≥ 32G **recommended**
- Python ≥ 3.9 with `pip` and `venv`
- Docker CLI on PATH; check disk space on the Docker root:
  `docker info | grep "Docker Root Dir"` → `df -h <that path>`
- AWS CLI v2 with credentials that can read ECR
  (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`)


## 2. Install the wheel

```bash
python3 -m venv ~/.venv/delivery
~/.venv/delivery/bin/pip install delivery_pipeline-1.0.3-py3-none-any.whl
```

The wheel installs a single CLI: `delivery-pipeline`. It is the **host
orchestrator only** — every algo step still runs in its own container.


## 3. Pull the images

Choose **one** of the two options below depending on your network.

### Option A — China mainland (recommended for domestic users)

Images are hosted at `imagepublic.genrobotai.com/genrobot/` — **no login
required**, just pull directly:

```bash
REG=imagepublic.genrobotai.com/genrobot/genimage
VER=v1.0.4
for step in qc merge vio vio_check; do
  docker pull ${REG}:${step}-${VER}
done
```

### Option B — Outside China (AWS ECR)

The algo images also live on Amazon ECR. You need an AWS account with
read permission on the registry plus AWS CLI v2 installed and configured.

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

Pre-pull every image the pipeline needs (optional but recommended — fail
fast on network / ECR / disk issues):

```bash
REG=764042516397.dkr.ecr.us-east-1.amazonaws.com/genimage
VER=v1.0.4
for step in qc merge vio vio_check; do
    docker pull ${REG}:${step}-${VER}
done
```


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


## 5. Run

```bash
# Absolute paths only — passed through to algo containers verbatim.
cd /path/to/das-ego-stack
export INPUT_DIR=$(realpath ./sample_input)
export OUTPUT_DIR=$(realpath -m ./output)
mkdir -p "$OUTPUT_DIR"
chmod -R a+rX  "$INPUT_DIR"
chmod -R a+rwX "$OUTPUT_DIR"

# --- China mainland ---
REG=imagepublic.genrobotai.com/genrobot/genimage
VER=v1.0.4
export ALGO_QC_IMAGE=${REG}:qc-${VER}
export ALGO_MERGE_IMAGE=${REG}:merge-${VER}
export ALGO_VIO_IMAGE=${REG}:vio-${VER}
export ALGO_VIO_CHECK_IMAGE=${REG}:vio_check-${VER}

~/.venv/delivery/bin/delivery-pipeline --steps qc,merge,vio,vio_check \
    --input-dir "$INPUT_DIR" --output-dir "$OUTPUT_DIR" \
    --continue-on-error

# --- Outside China (AWS ECR) ---
REG=764042516397.dkr.ecr.us-east-1.amazonaws.com/genimage
VER=v1.0.4
export ALGO_QC_IMAGE=${REG}:qc-${VER}
export ALGO_MERGE_IMAGE=${REG}:merge-${VER}
export ALGO_VIO_IMAGE=${REG}:vio-${VER}
export ALGO_VIO_CHECK_IMAGE=${REG}:vio_check-${VER}

~/.venv/delivery/bin/delivery-pipeline --steps qc,merge,vio,vio_check \
    --input-dir "$INPUT_DIR" --output-dir "$OUTPUT_DIR" \
    --continue-on-error
```

A successful run ends with:

```
summary: 1/1 groups succeeded
```

### 5.1 Override images (test locally built `:compile-dev` instead of ECR)

To smoke-test against images you built yourself (e.g.
`delivery_vio:compile-dev` from `dockerfiles/jobs/delivery/build_and_push_to_ecr.sh --skip-push`),
point each `ALGO_*_IMAGE` env var at your local tag:

```bash
export ALGO_QC_IMAGE=delivery_qc:compile-dev
export ALGO_MERGE_IMAGE=delivery_merge:compile-dev
export ALGO_VIO_IMAGE=delivery_vio:compile-dev
export ALGO_VIO_CHECK_IMAGE=delivery_vio_check:compile-dev

~/.venv/delivery/bin/delivery-pipeline --steps qc,merge,vio,vio_check \
    --input-dir "$INPUT_DIR" --output-dir "$OUTPUT_DIR" \
    --continue-on-error
```

### 5.2 Common flags

| Flag | Default | Meaning |
|---|---|---|
| `--steps` | from `DEFAULT_STEPS` env | Comma-separated step list to run; every step must have its `ALGO_*_IMAGE` env set |
| `--input-dir` | — | Directory containing the `.mcap` groups to process |
| `--output-dir` | — | Per-group result root |
| `--continue-on-error` | off | Keep going to the next group if one fails |


## 6. Output

```
$OUTPUT_DIR/<group-uuid>/
├── result.json                        # per-step status — check this first on failure
├── merged_output_ego_vio.mcap         # final artifact
└── debug/                             # intermediates; safe to delete after audit
    └── work/
        ├── step_outputs/<step>.json   # per-step stdout/result
        └── qc/ merge/ vio/ vio_check/
```

A top-level summary is logged at the end of the run:

```
summary: 3/4 groups succeeded
  FAIL ab12cd34 at vio
```


## 7. Troubleshooting

### `permission denied while trying to connect to the Docker daemon socket`

You aren't in the `docker` group. Fix:

```bash
sudo usermod -aG docker "$USER"
# log out and back in, then verify:
groups | grep docker
```

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
