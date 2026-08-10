> [中文文档](api-automation.md)

# API Automation

## Core Rule

Every automation step returns a `project_id`, `asset_id`, or `job_id`; the next step consumes those IDs directly.

## Minimal Chain

```text
login
  -> project_id
  -> protein asset_id
  -> pocket asset_id
  -> ligand asset_id
  -> prepared_ligand asset_id
  -> docking compatibility check
  -> job_id
  -> output_asset_ids
  -> file download
```

## Docking automation

```http
GET /api/v1/docking/compatibility?engine=unidock&project_id=<project_id>&protein_asset_id=<protein_asset_id>&ligand_asset_id=<ligand_asset_id>&pocket_asset_id=<pocket_asset_id>&ligand_index=0
POST /api/v1/docking/unidock
GET /api/v1/jobs/{job_id}/events
```

`/api/v1/docking/compatibility` is the gate for automated workflows. Submit docking only when `compatible=true`; otherwise follow `errors`, `warnings`, and `recommendations`.

## GROMACS / MD automation

```http
GET /api/v1/md/workflows/templates
POST /api/v1/md/workflows/validate
POST /api/v1/md/jobs
POST /api/v1/md/jobs/resume
GET /api/v1/jobs/{job_id}/events
```

`/api/v1/md/workflows/validate` is the preflight gate for MD jobs: it returns the step inspector, missing inputs, command preview, and output groups. Before a real run, require `ok=true`. `/api/v1/md/jobs/resume` creates a continuation job from `.cpt` and `.tpr` files in an existing `md_result` asset, for extending production or recovering an interrupted simulation.

## Model Zoo automation

```http
GET /api/v1/model-zoo/catalog
GET /api/v1/model-zoo/models/{model_id}
POST /api/v1/model-zoo/models/{model_id}/download
POST /api/v1/model-zoo/models/{model_id}/update
POST /api/v1/model-zoo/custom/download
POST /api/v1/model-zoo/custom/update
```

`catalog` returns pinned project models and custom model status. Download and update are global operations and require an administrator token. Model paths follow the ModelHub-compatible layout: `/data/export/ms|hf/<org>/<repo>/current`.

## API / Endpoints

The canonical API contract is generated from FastAPI OpenAPI:

- `/openapi.json`: machine-readable schema for agents and workflow engines.
- `/docs`: Swagger UI for interactive calls.
- `/redoc`: structured API reference.

The in-app `API Docs` page reads `/openapi.json` automatically and adds workflow chaining notes.

## Synchronization Rule

When a backend route changes, the OpenAPI schema updates automatically. The hand-written guide should only explain cross-step workflow logic, not duplicate every request/response field.
