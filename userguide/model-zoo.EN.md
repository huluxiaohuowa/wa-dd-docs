> [中文文档](model-zoo.md)

# Model Zoo

## Purpose

Model Zoo centrally downloads and updates models used by WA-DD. WA-DD owns downloads, updates, deletion, snapshot publication, and required-file validation. In VOS, it synchronizes status, progress, and paths with V-Scheduler without using V-Scheduler to download models. It does not include model runtime instance management.

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

Do not place models in another directory manually. Otherwise Model Zoo, V-Scheduler, and workers cannot recognize models through the same rules.

## VOS synchronization with V-Scheduler

Select the same `MODEL_HUB_SHARED_MODELS_PATH` public directory for WA-DD and V-Scheduler during installation; legacy Model Hub is also compatible. V-Scheduler is an optional viewer, not a prerequisite for WA-DD downloads.

Both Model Zoo and V-Scheduler mount this directory at `/data` and synchronize `registry/models.json` and `shared/tasks/*.json` through the existing shared-storage contract. Model records preserve other metadata and add `managed_by=wa-dd-model-zoo`, a relative root path, and a relative `current` path. Task records identify the WA-DD owner and continuously refresh download progress. Startup synchronizes existing project and custom models without registering never-downloaded project models as globally failed.

Successful downloads synchronize readiness and the requested revision (or the local snapshot identifier when no revision is specified); failures synchronize failure status; deletion removes the model and its WA-DD task record. V-Scheduler's `/data/...` and workers' `/modelhub/...` are different container mount paths for the same public directory, not interchangeable absolute paths.

Download, update, and delete WA-DD-managed models in WA-DD; use V-Scheduler to view status and paths. When another application has an unfinished task for the same model, WA-DD returns a conflict instead of overwriting progress or starting a download or deletion. The shared-task check is not an atomic cross-application operation lock, so avoid starting operations from both applications simultaneously. The management marker does not automatically disable V-Scheduler's management buttons.

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
./deploy.sh --profile amd --component model-zoo
```

For an already running deployment, `--root` may be omitted; the deployment script detects the runtime root and port from the running Web container. First deployments, recovery deployments, and new coexisting deployments must still pass `--root` explicitly.

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
