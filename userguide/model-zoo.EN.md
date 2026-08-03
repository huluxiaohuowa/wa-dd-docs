> [中文文档](model-zoo.md)

# Model Zoo

## Purpose

Model Zoo centrally downloads and updates models used by WA-DD. It uses a path layout compatible with later VOS model-hub integration and keeps only model download, update, and path management. It does not include model runtime instance management.

Pinned project models are shown at the top:

- PocketXMol: used by molecule generation.
- DeepTernary: used by TPD / PROTAC ternary-complex modeling.

The lower section provides a custom model download form for ModelScope or HuggingFace repository IDs.

## Path contract

In standalone deployments, the `wa-dd-model-zoo` container mounts `${MODEL_HUB_SHARED_MODELS_PATH}` as `/data`. The model service therefore maintains:

```text
/data/export/ms/<org>/<repo>/snapshots/<snapshot>
/data/export/ms/<org>/<repo>/current
/data/export/hf/<org>/<repo>/snapshots/<snapshot>
/data/export/hf/<org>/<repo>/current
```

The web and worker containers mount the same host directory as `/modelhub` and read:

```text
/modelhub/export/ms/<org>/<repo>/current
/modelhub/export/hf/<org>/<repo>/current
```

Do not place models in another directory manually. Otherwise Model Zoo, the later integrated model-hub, and workers cannot recognize and update them through the same rules.

## Workflow

1. Open `Model Zoo` from the top navigation.
2. In "Project models", check PocketXMol and DeepTernary status.
3. If a model is not ready, click "Download model".
4. To force a new snapshot, click "Update model".
5. To download another model, choose the source and enter a repository ID in "Custom model download", for example `huluxiaohuowa/deepternary`.

Download or update writes into a new snapshot and switches `current` only after required files pass validation. If an existing `current` already satisfies required files, a normal download does not blindly re-download it.

## Permissions

- Viewing model status requires login.
- Downloading and updating models are global operations and require administrator permission.

## Pinned models

### PocketXMol

Pinned repository:

```text
ms://huluxiaohuowa/pocketxmol
```

Stable read path:

```text
/modelhub/export/ms/huluxiaohuowa/pocketxmol/current
```

Required files:

```text
data/trained_models/pxm/checkpoints/pocketxmol.ckpt
data/trained_models/pxm/train_config/train.yml
```

### DeepTernary

Pinned repository:

```text
ms://huluxiaohuowa/deepternary
```

Stable read path:

```text
/modelhub/export/ms/huluxiaohuowa/deepternary/current
```

Required files:

```text
checkpoints/PROTAC/checkpoint.pth
checkpoints/MGD/checkpoint.pth
configs/protac.py
configs/glue.py
```

## Deployment component

Model Zoo is an independent CPU service:

```bash
./build_image.sh --profile amd --component model-zoo
./deploy.sh --profile amd --root /data/vos_workspace --component model-zoo
```

Thor / ARM uses the same Dockerfile and only changes the platform tag to `arm_YYYYMMDD`:

```bash
./build_image.sh --profile thor --component model-zoo
```

Related image variables:

```text
WA_DD_MODEL_ZOO_AMD_IMAGE=
WA_DD_MODEL_ZOO_THOR_IMAGE=
```

The web service accesses Model Zoo through the internal address:

```text
WA_DD_MODEL_ZOO_URL=http://wa-dd-model-zoo:8810
```
