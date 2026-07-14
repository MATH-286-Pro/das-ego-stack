```bash
python3 -m venv ~/.venv/delivery
~/.venv/delivery/bin/pip install delivery_pipeline-1.0.4-py3-none-any.whl
```

```bash
# 下载镜像
REG=imagepublic.genrobotai.com/genrobot/genimage
VER=v1.0.5
for step in qc merge vio vio_check pose6d pose6d_check; do
  docker pull ${REG}:${step}-${VER}
done
```

```bash
# 检查下载
docker images
```

```bash
# 测试转化流程
REG=imagepublic.genrobotai.com/genrobot/genimage
VER=v1.0.5
export ALGO_QC_IMAGE=${REG}:qc-${VER}
export ALGO_MERGE_IMAGE=${REG}:merge-${VER}
export ALGO_VIO_IMAGE=${REG}:vio-${VER}
export ALGO_VIO_CHECK_IMAGE=${REG}:vio_check-${VER}

~/.venv/delivery/bin/delivery-pipeline --steps qc,merge,vio,vio_check \
    --input-dir "$INPUT_DIR" --output-dir "$OUTPUT_DIR" \
    --continue-on-error



# 完整流程
unset REG VER
unset ALGO_QC_IMAGE ALGO_MERGE_IMAGE
unset ALGO_VIO_IMAGE ALGO_VIO_CHECK_IMAGE
unset ALGO_POSE6D_IMAGE ALGO_POSE6D_CHECK_IMAGE

REG=imagepublic.genrobotai.com/genrobot/genimage
VER=v1.0.5

export ALGO_QC_IMAGE="${REG}:qc-${VER}"
export ALGO_MERGE_IMAGE="${REG}:merge-${VER}"
export ALGO_VIO_IMAGE="${REG}:vio-${VER}"
export ALGO_VIO_CHECK_IMAGE="${REG}:vio_check-${VER}"

env | grep '^ALGO_.*_IMAGE='

~/.venv/delivery/bin/delivery-pipeline \
  --steps qc,merge,vio,vio_check,pose6d, pose6d_check\
  --input-dir "$INPUT_DIR" \
  --output-dir "$OUTPUT_DIR" \
  --continue-on-error
```