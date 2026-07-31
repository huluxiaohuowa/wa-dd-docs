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

## API / Endpoints

The canonical API contract is generated from FastAPI OpenAPI:

- `/openapi.json`: machine-readable schema for agents and workflow engines.
- `/docs`: Swagger UI for interactive calls.
- `/redoc`: structured API reference.

The in-app `API Docs` page reads `/openapi.json` automatically and adds workflow chaining notes.

## Synchronization Rule

When a backend route changes, the OpenAPI schema updates automatically. The hand-written guide should only explain cross-step workflow logic, not duplicate every request/response field.
