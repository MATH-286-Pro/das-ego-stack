# das-ego-stack
DAS Ego Local Data Acquisition and Processing Stack


## 1. Prerequisites

- Linux x86_64 (Ubuntu 20.04 / 22.04 or Debian 11 / 12),CPU>=16C,RAM>=32G.**recommended**
- Python ≥ 3.9 with `pip` and `venv`
- Docker CLI on PATH;(check with `docker info | grep "Docker Root Dir"` → `df -h <that path>`)
- AWS CLI v2 with credentials that can read ECR
  (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`)


## 2. Install

```bash
python3 -m venv ~/.venv/delivery
~/.venv/delivery/bin/pip install delivery_pipeline-1.0.0-py3-none-any.whl
```

The wheel installs a single CLI: `delivery-pipeline`.
It is the host orchestrator only — every step still runs in its own
container pulled from ECR.

## 3. Pull the image (AWS ECR)

The image is hosted on Amazon ECR. You need an AWS account with read
permission on the registry, plus the AWS CLI installed and configured
(`aws configure`).

```bash
# --- Install AWS CLI v2 (Linux x86_64) ---
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install          # requires sudo; use --update if already installed
aws --version               # verify
aws configure               # set access key / secret / region (e.g. us-east-1)
```

> **Heads-up on `sudo`**: AWS and docker credentials are per-user
> (`~/.aws/`, `~/.docker/`). Run every step below — and
> `delivery-pipeline` later — as the **same user**; don't mix sudo and
> non-sudo, or you'll hit `Unable to locate credentials` /
> `no basic auth credentials`.

```bash
# Authenticate docker to the ECR registry (token expires in 12 h).
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
      764042516397.dkr.ecr.us-east-1.amazonaws.com

```

```bash
# Pre-pull all images (optional but recommended — catches
#     ECR/network/disk problems before the pipeline starts)
REG=764042516397.dkr.ecr.us-east-1.amazonaws.com/genimage
VER=v1.0.0
for step in qc merge vio vio_check; do
    docker pull ${REG}:${step}-${VER}
done

```
## 4. Lay out inputs

Put every `.mcap` for one or more sessions into a single host directory.
Files are grouped by the 8-hex `<uuid>` suffix in the filename
(`*_<uuid>.mcap`); each complete group must contain:

- one `DAS-EGO_..._master_<uuid>.mcap`
- one `DAS-F_..._sub_left_<uuid>.mcap`
- one `DAS-F_..._sub_right_<uuid>.mcap`

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
export INPUT_DIR=/abs/host/input #INPUT dir
export OUTPUT_DIR=/abs/host/output #OUTPUT dir

~/.venv/delivery/bin/delivery-pipeline --variant vio \
    --ecr-registry 764042516397.dkr.ecr.us-east-1.amazonaws.com/genimage \
    --input-dir "$INPUT_DIR" --output-dir "$OUTPUT_DIR" --continue-on-error
```

### Common flags

| Flag | Default | Meaning |
|---|---|---|
| `--steps` | from `--variant` | Override the step list (comma-separated subset of `qc,merge,vio,vio_check`) |
| `--continue-on-error` | off | Keep going to the next group if one fails |



## 6. Output

```
$OUTPUT_DIR/<group-uuid>/
├── result.json                       # per-step status — check this first on failure
├── merged_output_ego_vio.mcap        # vio step output (topic-augmented)
└── debug/                            # intermediates; safe to delete after audit
    └── work/
        ├── step_outputs/<step>.json  # per-step stdout/result
        └── qc/ merge/ vio/ ...       # per-step working dirs
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
host directly — as this doc describes — there is no second namespace, so
this is automatic.)

### Per-step audit

When a group fails:

```bash
# top-level status across all steps
cat $OUTPUT_DIR/<group-uuid>/result.json

# the specific step's stdout/error
cat $OUTPUT_DIR/<group-uuid>/debug/work/step_outputs/<step>.json
```
